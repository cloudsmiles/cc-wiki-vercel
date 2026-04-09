# text-fu

## stdout(Standard Out)

 

```go
$ echo Hello World > peanuts.txt
```

>是一个重定向操作符，它允许我们将echo Hello World的输出发送到一个文件而不是屏幕上。如果该文件不存在，它将为我们创建它。然而，如果它确实存在，它将覆盖它（你可以添加一个shell标志来防止这种情况，这取决于你所使用的shell）。

## stdin(Standard In)

```go
$ cat < peanuts.txt > banana.txt
```

就像我们用>来重定向stdout一样，我们可以用<来重定向stdin。

通常在cat命令中，你向它发送一个文件，该文件就成为stdin，在这个例子中，我们把peanuts.txt重定向为我们的stdin。然后，cat peanuts.txt的输出是Hello World，被重定向到另一个叫做banana.txt的文件。

## stderr(Standard Error)

所以现在如果我们想把stderr重定向到该文件，我们可以这样做：

```go
$ ls /fake/directory 2> peanuts.txt
```

你应该只看到peanuts.txt中的stderr信息。

现在，如果我想在peanuts.txt文件中同时看到stderr和stdout呢？用文件描述符也可以这样做：

```go
$ ls /fake/directory > peanuts.txt 2>&1
```

这将ls /fake/directory的结果发送到peanuts.txt文件，然后它通过2>&1将stderr重定向到stdout。这里的操作顺序很重要，2>&1将stderr发送到stdout所指向的地方。在本例中，stdout指向一个文件，所以2>&1也将stderr发送到一个文件。因此，如果你打开peanuts.txt文件，你应该同时看到stderr和stdout。在我们的例子中，上面的命令只输出stderr。

有一个更短的方法可以将stdout和stderr都重定向到一个文件：

```go
$ ls /fake/directory &> peanuts.txt
```

现在，如果我不想要这些杂物，想完全摆脱stderr信息怎么办？那么你也可以将输出重定向到一个特殊的文件，称为/dev/null，它将抛弃任何输入。

```go
$ ls /fake/directory &> peanuts.txt
```

## pipe & tee

```go
$ ls -la /etc | less
```

管道操作符|，用一个竖条表示，允许我们获得一个命令的输出，并将其作为另一个进程的输出。在这个例子中，我们把ls -la /etc的stdout拿出来，然后用管道把它送到less命令中。管道命令是非常有用的，我们将继续使用它，直到永远。

那么，如果我想把我的命令的输出写到两个不同的流中呢？这就可以用tee命令来实现：

```go
ls | tee peanuts.txt
```

你应该在屏幕上看到ls的输出，如果你打开peanuts.txt文件，你应该看到同样的信息

## cut

cut命令将文件的每一行进行切割，可以根据字符的位置，分隔符进行切分

[https://linuxjourney.com/lesson/cut-command](https://linuxjourney.com/lesson/cut-command)

## paste

paste命令将文件的每一行进行合并

[https://linuxjourney.com/lesson/paste-command](https://linuxjourney.com/lesson/paste-command)

## join & split

The join command allows you to join multiple files together by a common field:

Let's say I had two files that I wanted to join together:

```
file1.txt

1 John

2 Jane

3 Maryfile2.txt

1 Doe

2 Doe

3 Sue$ join file1.txt file2.txt

1 John Doe

2 Jane Doe

3 Mary Sue
```

See how it joined together my files? They are joined together by the first field by default and the fields have to be identical, if they are not you can sort them, so in this case the files are joined via 1, 2, 3.

How would we join the following files?

```
file1.txt

John 1

Jane 2

Mary 3file2.txt

1 Doe

2 Doe

3 Sue
```

To join this file you need to specify which fields you are joining, in this case we want field 2 on file1.txt and field 1 on file2.txt, so the command would look like this:

```

$ join -1 2 -2 1 file1.txt file2.txt

1 John Doe

2 Jane Doe

3 Mary Sue
```

- 1 refers to file1.txt and -2 refers to file2.txt. Pretty neat. You can also split a file up into different files with the split command:

```
$ split somefile
```

This will split it into different files, by default it will split them once they reach a 1000 line limit. The files are named x** by default.

## sort

The sort command is useful for sorting lines.

```

file1.txt

dog

cow

cat

elephant

bird$ sort file1.txt

bird

cat

cow

dog

elephant
```

You can also do a reverse sort:

```
$ sort -r file1.txt

elephant

dog

cow

cat

bird
```

And also sort via numerical value:

```
$ sort -n file1.txt

bird

cat

cow

elephant

dog
```

## **tr (Translate)**

The tr (translate) command allows you to translate a set of characters into another set of characters. Let's try an example of translating all lower case characters to uppercase characters.

```
$ tr a-z A-Z

hello

HELLO
```

As you can see we made the ranges of a-z into A-Z and all text we type that is lowercase gets uppercased.

## **uniq (Unique)**

The uniq (unique) command is another useful tool for parsing text.

Let's say you had a file with lots of duplicates:

```
reading.txt

book

book

paper

paper

article

article

magazine
```

And you wanted to remove the duplicates, well you can use the uniq command:

```
$ uniq reading.txt

book

paper

article

magazine
```

Let's get the count of how many occurrences of a line:

```
$ uniq -c reading.txt

2 book

2 paper

2 article

1 magazine
```

Let's just get unique values:

```
$ uniq -u reading.txt

magazine
```

Let's just get duplicate values:

```
$ uniq -d reading.txt

book

paper

article

```

**Note** : uniq does not detect duplicate lines unless they are adjacent. For eg:

Let's say you had a file with duplicates which are not adjacent:

```
reading.txt

book

paper

book

paper

article

magazine

article
```

```
$ uniq reading.txt

reading.txt

book

paper

book

paper

article

magazine

article
```

The result returned by uniq will contain all the entries unlike the very firstexample.

To overcome this limitation of uniq we can use sort in combination with uniq:

```

$ sort reading.txt | uniq

article

book

magazine

paper
```

reference

[https://linuxjourney.com/lesson/stderr-standard-error-redirect](https://linuxjourney.com/lesson/stdout-standard-out-redirect)