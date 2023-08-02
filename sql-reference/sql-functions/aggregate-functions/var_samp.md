
# VAR_SAMP, VARIANCE_SAMP

## 功能

返回 expr 表达式的样本方差。

## 语法

```Haskell
VAR_SAMP(expr)
```

## 参数说明

`epxr`: 被选取的表达式。

当表达式为列值时，支持以下数据类型: TINYINT、SMALLINT、INT、BIGINT、LARGEINT、FLOAT、DOUBLE、DECIMAL

## 返回值说明

返回值为double类型。

## 示例

```plain text
MySQL > select var_samp(scan_rows)
from log_statis
group by datetime;
+-----------------------+
| var_samp(`scan_rows`) |
+-----------------------+
|    5.6227132145741789 |
+-----------------------+
```
