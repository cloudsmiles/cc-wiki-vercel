# form表单

# **定义**

`form` 表单在网页中主要负责[数据采集](https://so.csdn.net/so/search?q=%E6%95%B0%E6%8D%AE%E9%87%87%E9%9B%86&spm=1001.2101.3001.7020)功能，属于一个容器标记。

# **表单组成**

一个表单由 `form`元素、表单控件 和 表单按钮 组成。

（1） `form`元素

`form`元素用来创建表单，语法格式如下：

```jsx
<form action="" method="" name="" enctype="">
     ...
</form>
```

`action`：属性用于指定接收并处理表单数据的服务器程序的URL地址

`name`：属性用于指定表单的名称，以区分同一个页面中的多个表单

`method`：设置表单的提交方式（主要有`post`和`get`）

`enctype` : 设置表单内容数据类型，规定在表单发送到服务器前应该用何种方式对表单数据进行编码

1.application/x-www-form-urlencoded(默认方式)
2.multipart/form-data不对字符进行编码，在使用包含文件上传控件的表单时，必须使用该属性值。它支持文本数据，也支持二进制数据上传，使用此值时，说明一般居有多媒体数据，数据大量的情况下，规定上传文件method是post方法，type属性必须是file。
3.text/plain:空格转换为“+”，但不对特殊字符进行编码。

（2） 表单控件

表单控件包含了具体的表单功能项，主要用来收集用户数据，包括 `label(标签)、input、textarea、select、datalist、keygen` 等，还有对组件分组显示的 `fieldset` 和 `legend` 控件。

根据功能的不同，`input`又可以分为 `text、password、radio、checkbox、file、submit、reset、search、tel、url、email、number、range、color、Date Pickers` 等类型。

### input表单控件

type属性

| 属性值 | 表示 | 意义 |
| --- | --- | --- |
| text | 单行文本框 |  |
| password | 密码框 | 输入的内容会被遮挡 |
| checkbox | 复选框 | 必须使用value属性来描述该组件所提交的值，使用checked属性默认 |
| radio | 单选按钮 | 必须用value属性描述该组件提交的值，一个单选按钮组的所有控件都应该具有相同的name值，这样，才能具有单选的意义。 |
| submit | 提交按钮 | <form ><button>提交</button></form>都具有真实的提交功能 |
| reset | 重置按钮 |  |
| file | 文件按钮 | 该控件用于选中文件系统的某个文件 |
| hidden | 隐藏域 | 该控件不显示在页面内，但是其值会被提交 |
| image | 图像按钮 | 使用src来加载图片，使用alt来声明替换文本 |
| button | 普通按钮 | 不具有真实的提交功能 |

（3）表单按钮

包括普通按钮、提交按钮(`submit`)和重置按钮(`reset`)

# 表单提交原理

表单能够提交并提交成功接收返回数据时建立在http协议的基础上，http的工作原理就是客户端如何向服务器请求数据，服务器如何向浏览器返回数据。

客户端向服务器发送一个请求报文（请求行（请求的方法，URL，协议版本），请求头部，请求体），服务器以一个状态行作为响应。

http请求方法最常用到的是get和post。

## get和post异同

1.安全性： get不安全，由于数据传输时数据被放在请求的URL中；post的所有操作对用户来说都是不可见的。
2.传送数据大小：get传送的数据量受URL限制，数据量较小，而post较大。
3.数据集的值限制：get限制form表单的数据集的值必须为ASCII字符，而post支持整个ISO10646字符集。
4.执行效率：get 执行效率比post好，form表单默认get。