# REST

**REST(Representational State Transfer，表征性状态转移)是一种架构风格，多用于Web应用**

REST是网络的基本架构原则。网络的神奇之处在于，客户（浏览器）和服务器可以以复杂的方式进行互动，而客户事先对服务器和它所承载的资源一无所知。关键的制约因素是，服务器和客户端必须就所使用的媒体达成一致，在网络的情况下，这就是HTML。

遵循REST原则的API并不要求客户对API的结构有任何了解。相反，服务器需要提供客户需要的任何信息来与服务进行交互。一个HTML表格就是这样的例子： 服务器指定了资源的位置和所需的字段。浏览器不会事先知道在哪里提交信息，也不会事先知道要提交什么信息。这两种形式的信息都完全由服务器提供。(这个原则被称为HATEOAS：超媒体作为应用状态的引擎）。

那么，这如何适用于HTTP，以及如何在实践中实现？HTTP是以动词和资源为导向的。主流用法中的两个动词是GET和POST，我想每个人都会认识到这一点。然而，HTTP标准定义了其他几个动词，如PUT和DELETE。这些动词然后根据服务器提供的指示应用于资源。

例如，让我们想象一下，我们有一个用户数据库，由一个网络服务管理。我们的服务使用基于JSON的自定义超媒体，为此我们指定了mimetype application/json+userdb（也可能有application/xml+userdb和application/whatever+userdb--可能支持许多媒体类型）。客户端和服务器都被编程为理解这种格式，但它们对彼此并不了解。正如Roy Fielding所指出的：

> REST API应该把几乎所有的描述性工作都花在定义用于表示资源和驱动应用状态的媒体类型上，或者定义现有标准媒体类型的扩展关系名称和/或超文本标记。
> 

reference

[https://stackoverflow.com/questions/671118/what-is-restful-programming](https://stackoverflow.com/questions/671118/what-is-restful-programming)