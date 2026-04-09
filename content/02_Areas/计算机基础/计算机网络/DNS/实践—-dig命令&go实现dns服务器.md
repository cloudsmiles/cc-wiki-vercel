# 实践—-dig命令&go实现dns服务器

通常，DNS查询和应答将作为UDP消息传递，因为它们是轻量级的。来自DNS服务器的查询和应答将使用 DNS标头中的ID进行映射。

> DNS请求也有可能使用TCP传输，当报文长度过长时，会用TCP重试
> 

TCP将在DNS服务器之间消息传递时使用，例如：zone传输，其中DNS服务器的全部内容被复制到另一个 DNS服务器。

要构建DNS服务器，我们必须侦听特定端口以接收传入查询。默认情况下，DNS服务器在端口53上运行。

```go

go复制代码
//Listen on UDP Portaddr := net.UDPAddr{
        Port: 8090,
        IP:   net.ParseIP("127.0.0.1"),
}
u, _ := net.ListenUDP("udp", &addr)

```

## 监听端口

当我们以UDP消息的形式接收数据时，我们必须从中解码DNS消息。由于DNS消息结构复杂，我们很难自己编写DNS编码和解码。我们可以改为使用现有的开源库，或者我们可以构建一个库来满足更多定制化的需求。

这里我们使用Google的[packets](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgoogle%2Fgopacket)

```go

//Listen on UDP Portaddr := net.UDPAddr{
    Port: 8090,
    IP:   net.ParseIP("127.0.0.1"),
}
u, _ := net.ListenUDP("udp", &addr)

// Wait to get request on that portfor {
    tmp := make([]byte, 1024)
    _, addr, _ := u.ReadFrom(tmp)
    clientAddr := addr
    packet := gopacket.NewPacket(tmp, layers.LayerTypeDNS, gopacket.Default)
    dnsPacket := packet.Layer(layers.LayerTypeDNS)
    tcp, _ := dnsPacket.(*layers.DNS)
    serveDNS(u, clientAddr, tcp)
}
```

我们现在可以监听UDP端口并获取UDP消息并使用Google数据包将其解码为DNS消息。在实现`serveDNS`功能之前，我们只需要很少的记录来服务DNS查询。

```go

go复制代码
package main

import (
    "fmt"    "net"    "github.com/google/gopacket"    layers "github.com/google/gopacket/layers")

var records map[string]stringfunc main() {
    records = map[string]string{
            "baidu.com": "223.143.166.121",
            "github.com": "79.52.123.201",
    }

    //Listen on UDP Port    addr := net.UDPAddr{
        Port: 8090,
        IP:   net.ParseIP("127.0.0.1"),
    }
    u, _ := net.ListenUDP("udp", &addr)

    // Wait to get request on that port    for {
        tmp := make([]byte, 1024)
        _, addr, _ := u.ReadFrom(tmp)
        clientAddr := addr
        packet := gopacket.NewPacket(tmp, layers.LayerTypeDNS, gopacket.Default)
        dnsPacket := packet.Layer(layers.LayerTypeDNS)
        tcp, _ := dnsPacket.(*layers.DNS)
        serveDNS(u, clientAddr, tcp)
    }
}

func serveDNS(u *net.UDPConn, clientAddr net.Addr, request *layers.DNS) {
    replyMess := request
    var dnsAnswer layers.DNSResourceRecord
    dnsAnswer.Type = layers.DNSTypeA
    var ip string    var err error    var ok bool    ip, ok = records[string(request.Questions[0].Name)]
    if !ok {
            //Todo: Log no data present for the IP and handle:todo    }
    a, _, _ := net.ParseCIDR(ip + "/24")
    dnsAnswer.Type = layers.DNSTypeA
    dnsAnswer.IP = a
    dnsAnswer.Name = []byte(request.Questions[0].Name)
    fmt.Println(request.Questions[0].Name)
    dnsAnswer.Class = layers.DNSClassIN
    replyMess.QR = true    replyMess.ANCount = 1    replyMess.OpCode = layers.DNSOpCodeNotify
    replyMess.AA = true    replyMess.Answers = append(replyMess.Answers, dnsAnswer)
    replyMess.ResponseCode = layers.DNSResponseCodeNoErr
    buf := gopacket.NewSerializeBuffer()
    opts := gopacket.SerializeOptions{} // See SerializeOptions for more details.    err = replyMess.SerializeTo(buf, opts)
    if err != nil {
            panic(err)
    }
    u.WriteTo(buf.Bytes(), clientAddr)
}

```

这些记录在主函数的开始初始化。

并且主函数不断侦听端口8090，并提供查询DNS服务器功能。

该`serveDNS`函数获取连接，客户端地址和查询请求作为参数。然后，该请求消息用于模拟响应消息，并且必须在响应变化的值也被修改。

然后，响应消息被序列化到UDP消息格式并写回，使用在接收作为UDP包中的DNS消息中获得的客户端地址的客户端。

运行此程序，我们应该能够解析主函数中指定的记录的DNS查询，即运行下面的命令应该从我们创建的DNS服务器带来的DNS响应。

```

dig github.com @localhost -p8090 +short

```

返回：

```yaml

; <<>> DiG 9.10.6 <<>> github.com @localhost -p8090
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: NOTIFY, status: NOERROR, id: 16550
;; flags: qr aa rd ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;github.com.                    IN      A

;; ANSWER SECTION:
github.com.             0       IN      A       79.52.123.201

;; Query time: 0 msec
;; SERVER: 127.0.0.1#8090(127.0.0.1)
;; WHEN: Thu Aug 12 16:33:53 CST 2021
;; MSG SIZE  rcvd: 65
```