# git配置多用户环境

本地生成，指定私钥名称github_rsa，此时git也会生成对应的公钥名称github_rsa.pub。然后编辑config文件

cd ~/.ssh/

vi config

```json
# 指定github.com域名使用指定私钥，不然默认会使用id_rsa

Host github.com
 HostName github.com
 User git
 IdentityFile ~/.ssh/github_rsa

```

ESC+:wq保存退出

重新尝试ssh -T [[git@github.com]]，即可搞定

reference:

[github提示Permission denied (publickey)，如何才能解决？ - 知乎](https://www.zhihu.com/question/21402411)