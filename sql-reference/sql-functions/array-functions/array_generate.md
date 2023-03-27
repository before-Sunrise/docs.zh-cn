# array_generate

## 功能

生成一个从start到end的数组，步长为step。

## 语法

```Haskell
ARRAY array_generate([start,] end [, step])
```

## 参数说明

`start`：可选参数。支持数据类型为 TINYINT、SMALLINT、INT、BIGINT、LARGEIN的常量或列。默认值为1。
`end`：必选参数。支持数据类型为 TINYINT、SMALLINT、INT、BIGINT、LARGEINT的常量或列。
`step`：可选参数。支持数据类型为 TINYINT、SMALLINT、INT、BIGINT、LARGEINT的常量或列。当start < end 时, 默认值为1。当start >= end 时，默认值为-1。

## 返回值说明

返回的数据类型为 ARRAY。

## 注意事项

* 当任意参数为列时需指定列所属的表。
* 当任意参数为列时，不支持使用默认参数。
* 当任意参数为NULL时，结果返回NULL。
* 当step = 0时，返回空数组。

## 示例

参数为常量的情况如下。

```Plain Text
mysql> select array_generate(9);
+---------------------+
| array_generate(9)   |
+---------------------+
| [1,2,3,4,5,6,7,8,9] |
+---------------------+
```

参数为列的情况如下。

```Plain Text
mysql> select array_generate(1,b,2) from A;
+---------------------------+
| array_generate(1, b, 2)   |
+---------------------------+
| [1,3]                     |
| [1,3,5]                   |
| [1,3,5,7]                 |
| [1,3,5,7,9]               |
+---------------------------+
```
