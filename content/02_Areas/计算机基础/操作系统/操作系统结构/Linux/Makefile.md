# Makefile

## Makefile 简介

`Makefile` 是一个管理项目的配置文件，它主要有 2 个作用：

1. 组织工程文件，编译成复杂的程序
2. 安装及卸载程序

Makefile 是被 `make` 这个程序执行的，当执行 `make` 时，如果发现当前目录下有 `Makefile` 或者 `makefile` 文件，那么 `make` 命令就会根据这个文件的内容来找到相关的文件，然后去执行编译，链接，安装，卸载等操作，这就实现了使用 `make + Makefile` 来管理项目的功能。

这篇文章主要是介绍编写 Makefile 的基础语法，帮助你能够编写和理解一般的 Makefile 文件。

## Makefile 的工作流程

在学习 Makefile 语法之前，我们先来学习下 `make` 是如何解析 Makefile 文件的，主要分为下面 4 个步骤：

1. 在当前目录下查找 Makefile 或者 makefile 的文件
2. 找到之后，解析并得到最终要生成的目标文件
3. 根据时间戳生成目标文件
4. 递归去寻找其他目标文件的依赖文件，并递归生成

## Makefile 基础语法

### Makefile 编写规则

**Makefile 由若干条规则组成**，规则格式如下:

```jsx
目标（target）: 依赖（prerequisites）
	命令（command）
```

例如：

```jsx
main : main.o
	gcc main.o -o main

main.o : main.c
	gcc -c main.c -o main.o
```

其中 `main` 是最后生成的目标文件，`main.o` 是依赖文件，`gcc main.o -o main` 是编译命令，要注意的是 `main.o` 是由 `main.c` 生成的，所以还需要一条编译 `main.c` 生成 `main.o` 的规则，这些依赖关系规则共同组成了最后的 Makefile。

### Makefile 变量

### 用户自定义变量

定义格式如下：

`VAR_NAME = var_value`

使用 `$(VAR_NAME)`来引用变量，比如：

```jsx
file_name = hello.c
hello:
	gcc $(file_name) -o hello
```

### 预定义变量

Makefile 常见的预定义变量有下面这些：

- AR：库文件维护程序，默认为 `ar`
- AS：汇编程序，默认为 `as`
- CC：C 编译器，默认为 `cc`
- CXX：C++ 编译器，默认为 `g++`
- ARFLAGS：库文件维护程序选项，无默认值
- ASFLAGS：汇编程序选项，无默认值
- CFLAGS：C 编译器选项，无默认值
- CXXFLAGS：C++ 编译器选项，无默认值

### 环境变量

Makefile 常用的环境变量有下面这些：

- `$*`：不包含扩展名的目标文件名称
- `$<`：第一个依赖文件名称
- `$?`：所有时间戳比目标文件晚的依赖文件
- `$@`：目标文件完整名称
- `$^`：所有不重复的依赖文件

### Makefile 伪目标

先来看一个伪目标：

```jsx
.PHONY: install
install:
	cp hello /usr/local/bin/hello
```

伪目标 `install` 表示即使当前文件夹内有 `install` 这个文件，但是 `make` 执行的仍然是 `Makefile` 内部的 `install`，不会使用外部的文件，相当于进行了一个内部封装。

reference:

[What is the purpose of .PHONY in a Makefile?](https://stackoverflow.com/questions/2145590/what-is-the-purpose-of-phony-in-a-makefile)

[Linux 高级编程 - Makefile 基础语法 - 登龙（DLonng）](https://dlonng.com/posts/makefile)

[GNU make](https://www.gnu.org/software/make/manual/make.html#Introduction)