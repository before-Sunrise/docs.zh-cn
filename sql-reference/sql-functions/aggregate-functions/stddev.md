
# stddev, stddev_pop, std

## 功能

返回 `expr` 表达式的总体标准差。

### 语法

```Haskell
STDDEV(expr)
```

## 参数说明

`epxr`: 被选取的表达式。

当表达式为列值时，支持以下数据类型: TINYINT、SMALLINT、INT、BIGINT、LARGEINT、FLOAT、DOUBLE、DECIMAL

## 返回值说明

返回值为double类型。

## 示例

```plain text
mysql> SELECT  stddev(lo_quantity), stddev_pop(lo_quantity) from lineorder;
+---------------------+-------------------------+
| stddev(lo_quantity) | stddev_pop(lo_quantity) |
+---------------------+-------------------------+
|   14.43100708360797 |       14.43100708360797 |
+---------------------+-------------------------+
```
