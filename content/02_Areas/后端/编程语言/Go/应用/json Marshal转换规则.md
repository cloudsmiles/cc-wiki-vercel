# json Marshal转换规则

## 1.golang to json

| golang | json |
| --- | --- |
| bool | Boolean |
| int、float等数字 | Number |
| string | String |
| []byte(base64编码) | String |
| struct | Object，再递归打包 |
| array/slice | Array |
| map | Object |
| interface{} | 按实际类型转换 |
| nil | null |
| channel,func | UnsupportedTypeError |

## 2.json to golang

| json | golang |
| --- | --- |
| Boolean | bool |
| Number | float64 |
| String | string |
| Array | []interface{} |
| Object | map[string]interface{} |
| null | nil |