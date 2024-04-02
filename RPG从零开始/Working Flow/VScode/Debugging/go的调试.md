# go的调试

## 调试工具

`Delve`是Go语言调试工具。`vscode`调试，实质是集成了`Delve`

### 调试main.go

```json
#启动调试
dlv debug .\main.go

#打断点
b main.go:75 #main.go的75行打断点

#执行至断点
c

#退出
q
```

debug命令会先编译go源文件，同时执行attach命令进入调试模式，该命令会在当前目录下生成一个名为debug的可执行二进制文件`__debug_bin`，退出调试模式会自动被删除。

- `b`：**break-打断点**
- `c`：**continue-继续运行，直到断点处停止**
- `n`：**next-单步运行**
- `locals`：**打印local varables**
- `p`：**print打印一个变量或者表达式**
- `r`：**restart 重启进程**

### 调试*_test.go

单元测试的重要性就不赘述。go语言里面 `_test.go` 结尾的文件会被认为是测试文件，go语言作为现代化的语言，语言工具层面就支持单元测试。

利用 `go test` 命令，会直接编译测试文件为二进制文件后，再运行。

**但是**，有时候我们需要知道执行单元测试的细节，无论是验证也好，还是去寻找单元测试没有PASS的原因。那么**调试测试代码**就成了刚需。

```json
#启动调试
dlv test .\main_test.go

#打断点
b main_test.go:10

#或者具体测试方法
b TestSum

#执行至断点
c

#退出
q
```

## VScode调试

### 添加调试文件

项目创建.vscode/launch.json配置文件，添加以下格式，具体配置的命名意义请参考上级文档

```json
## launch.json
{   
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "program": "${workspaceFolder}/cmd/boss-test",
            "mode": "auto",
            "args": ["-c", "${workspaceFolder}/conf/app_local.yaml"],
            "env": {},
            "dlvFlags": ["--check-go-version=false"]
        }
    ]
}
```

也支持多个配置，切换即可

### 单元测试

**vscode为开发者提供了4个一键操作。**

![Untitled](RPG从零开始/Working%20Flow/VScode/Debugging/go的调试/Untitled.png)

- **run package tests**
    - 运行整个包中的测试，等价于执行`go test -cover`，**请谨慎使用**
- **run file tests**
    - 运行本文件的测试方法，等价于执行`go test *_test.go`，**请谨慎使用**
- **run test**
    - 只运行单个测试方法，等价于执行`go test -v main_test.go --test.run 测试方法名称`
- **debug test**
    - 只调试单个测试方法

reference:

[https://stackoverflow.com/questions/59142164/how-to-pass-command-line-arguments-in-debug-mode-in-vscode-with-golang](https://stackoverflow.com/questions/59142164/how-to-pass-command-line-arguments-in-debug-mode-in-vscode-with-golang)

[【Vscode】调试go语言程序的最佳实践-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2028975)

[[]] at master · [[]]