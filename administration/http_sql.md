# HTTP SQL API

# 功能

通过http协议使用STARROCKS的查询功能，当前支持select，show，explain，kill语句。

# 请求报文

指定catalog, 跨database查询, SQL语句中出现的表需要前面冠以database名（当前仅支持内表查询）

```SQL
POST /api/v1/catalogs/<catalog_name>/sql
```

指定catalog和database查询

```SQL
POST /api/v1/catalogs/<catalog_name>/databases/<database_name>/sql
```

### Request Header

使用Authorization进行basic鉴权

```SQL
Authorization: Basic <credentials>
```

### Request body

| field                    | Description                                                  |
| ------------------------ | :----------------------------------------------------------- |
| query                    | string格式。执行的sql字符串，当前支持select，show，explain, kill。一次http请求只允许执行一条sql |
| sessionVariables（可选） | json格式。指定session变量, 默认为空。设置的session变量在同一连接中始终有效，连接断开后session变量失效。 |

# 响应报文

## 状态码

1. http请求成功，且在发送数据给客户端之前服务器未出现异常，返回200
2. 当http请求错误时，返回4xx，表示客户端出错
3. http请求成功，但是在发送数据给客户端之前出现异常，返回500 Internal Server Error
4. http请求成功，但是FE当前无法提供服务，返回503 

### Response header

Content-type表示response body的格式。这里使用Newline delimited JSON格式，即response body由若干json object组成，json object之间以\n 隔开

|                      | Description                                                  |
| -------------------- | :----------------------------------------------------------- |
| content-type         | 格式为Newline delimited JSON. 默认为"application/x-ndjson charset=UTF-8" |
| X-StarRocks-Query-Id | Query id                                                     |

### Response body

#### 在结果发出之前执行失败

- 客户端的请求出错或者服务端在将数据发送给客户端之前出现异常，此时response body格式如下。其中msg为对应的错误异常信息

```SQL
{
   "status":"FAILED",
   "msg":"xxx"
}
```

#### 在结果发出部分后执行失败

此时结果已经部分发出，http状态码为200，因此选择不再继续发送数据，并关闭连接，记录错误日志。

#### 执行成功

对于select命令：每行是一个json object，json object之间以换行符隔开，从而便于客户端解析。共存在三种json object

| object       | Description                                                  |
| ------------ | :----------------------------------------------------------- |
| connectionId | 该连接对应的ID，可通过kill connectionId来取消某个执行时间过长的query |
| meta         | 用来描述每个列。key为“meta", value为一个json array，array中的每个object表示一个column。 |
| data         | 代表一行数据。key为“data" ,value为一个json array，array中包含一条数据 |
| statistics   | 本次执行结果的统计信息                                       |

一个样例：这里换行以\n表示。发送时STARROCKS使用http chunked方式传输数据，FE每获取一批数据，就将该批数据流式转发给客户端。客户端可以按行解析STARROCKS发送的数据，而不需要缓存现有结果直到所有数据发送完毕再进行解析。这可以降低客户端的内存消耗。

```SQL
{"connectionId": 7}\n
{"meta": [
    {
      "name": "stock_symbol",
      "type": "varchar"
    },
    {
      "name": "closing_price",
      "type": "decimal64(8, 2)"
    },
    {
      "name": "closing_date",
      "type": "datetime"
    }
  ]}\n
{"data": ["JDR", 12.86, "2014-10-02 00:00:00"]}\n
{"data": ["JDR",14.8, "2014-10-10 00:00:00"]}\n
...
{"statistics": {"scanRows": 0,"scanBytes": 0,"returnRows": 9}}
```

# 取消查询

当想要取消一个耗时过长的查询时，可以直接断开连接。当检测到连接已断开，STARROCKS会自动取消该查询。

* 此外，也可以通过发送kill connectionId的方式取消该查询。

```
curl -X POST 'http://127.0.0.1:8030/api/v1/catalogs/default_catalog/databases/test/sql' -u 'root:' -d '{"query": "kill 17;"}' --header "Content-Type: application/json"
```

connectionId除了在Response body中会返回，也可以通过show processlist中获得

```
curl -X POST 'http://127.0.0.1:8030/api/v1/catalogs/default_catalog/databases/test/sql' -u 'root:' -d '{"query": "show processlist;"}' --header "Content-Type: application/json"
```

# 示例

发起查询：

```
curl -X POST 'http://127.0.0.1:8030/api/v1/catalogs/default_catalog/databases/test/sql' -u 'root:' -d '{"query": "select * from agg;"}' --header "Content-Type: application/json"
```

返回结果：

```
{"connectionId":49}
{"meta":[{"name":"no","type":"int(11)"},{"name":"k","type":"decimal64(10, 2)"},{"name":"v","type":"decimal64(10, 2)"}]}
{"data":[1,"10.00",null]}
{"data":[2,"10.00","11.00"]}
{"data":[2,"20.00","22.00"]}
{"data":[2,"25.00",null]}
{"data":[2,"30.00","35.00"]}
{"statistics":{"scanRows":0,"scanBytes":0,"returnRows":5}}
```

发起查询的同时设置session变量:

```
curl -X POST 'http://127.0.0.1:8030/api/v1/catalogs/default_catalog/databases/test/sql' -u 'root:'  -d '{"query": "SHOW VARIABLES;", "sessionVariables":{"broadcast_row_limit":14000000}}'  --header "Content-Type: application/json"
```
