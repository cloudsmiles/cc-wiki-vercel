---
up:
  - "[[ssh]]"
---
## known_hosts文件

known_hosts文件是SSH客户端中的一个重要配置文件。当首次与一个SSH服务器建立连接时，客户端会记录下该服务器返回的的公钥，并保存在known_hosts文件中，以后每次连接该服务器时，客户端都会验证该服务器返回的公钥是否与known_hosts文件中保存的一致。如果不一致，则会发出警告，提示可能存在DNS劫持、中间人攻击等安全问题。因此，known_hosts文件可以保证SSH连接的安全性，防止恶意攻击。

Linux和McOS系统中所在路径为 ~/.ssh/known_hosts ，Windows系统中所在路径为 %USERPROFILE%\.ssh\known_host 。


## 常见问题和解决方法

使用SSH连接远程服务器或者使用Git拉取代码时，偶尔会出现“WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!”警告，一般是以下几种情况导致的：

1. 远程服务器重装或更换了系统，导致系统生成的密钥发生了变化。
2. 本地计算机重装了系统或者更改了SSH客户端软件。
3. 发生了中间人攻击，远程服务器可能是伪造的。

如果确定是第三种情况，即中间人攻击，此时请不要继续连接，并做好安全防护措施。如果确定连接是安全的，可以通过如下几种方法解决:
**一、运行如下命令，刷新known_hosts中对应远程服务器公钥，推荐此方法**

```shell
ssh-keygen -R server_ip_address
ssh-keyscan -H server_ip_address >> ~/.ssh/known_hosts
```

**二、直接删除known_hosts文件**

```shell
rm -f ~/.ssh/known_hosts
```

**三、只删除对应ip的相关公钥信息**

编辑 ~/.ssh/known_hosts 文件，将目标ip公钥信息删除后保存即可。