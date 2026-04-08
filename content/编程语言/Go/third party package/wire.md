# wire

## **快速使用**

先安装工具：

`$ go get github.com/google/wire/cmd/wire`

上面的命令会在`$GOPATH/bin`中生成一个可执行程序`wire`，这就是代码生成器。我个人习惯把`$GOPATH/bin`加入系统环境变量`$PATH`中，所以可直接在命令行中执行`wire`命令。

## **基础概念**

`wire`有两个基础概念，`Provider`（构造器）和`Injector`（注入器）。`Provider`实际上就是创建函数，大家意会一下。我们上面`InitMission`就是`Injector`。每个注入器实际上就是一个对象的创建和初始化函数。在这个函数中，我们只需要告诉`wire`要创建什么类型的对象，这个类型的依赖，`wire`工具会为我们生成一个函数完成对象的创建和初始化工作。

## **参数**

同样细心的你应该发现了，我们上面编写的`InitMission()`函数带有一个`string`类型的参数。并且在生成的`InitMission()`函数中，这个参数传给了`NewPlayer()`。`NewPlayer()`需要`string`类型的参数，而参数类型就是`string`。所以生成的`InitMission()`函数中，这个参数就被传给了`NewPlayer()`。如果我们让`NewMonster()`也接受一个`string`参数呢？

```jsx
func NewMonster(name string) Monster {
  return Monster{Name: name}
}
```

那么生成的`InitMission()`函数中`NewPlayer()`和`NewMonster()`都会得到这个参数：

```jsx
func InitMission(name string) Mission {
  player := NewPlayer(name)
  monster := NewMonster(name)
  mission := NewMission(player, monster)
  return mission
}
```

实际上，`wire`在生成代码时，构造器需要的参数（或者叫依赖）会从参数中查找或通过其它构造器生成。决定选择哪个参数或构造器完全根据类型。如果参数或构造器生成的对象有类型相同的情况，运行`wire`工具时会报错。如果我们想要定制创建行为，就需要为不同类型创建不同的参数结构

## **错误**

不是所有的构造操作都能成功，没准勇士出山前就死于小人之手：

```jsx
func NewPlayer(name string) (Player, error) {
  if time.Now().Unix()%2 == 0 {
    return Player{}, errors.New("player dead")
  }
  return Player{Name: name}, nil
}
```

我们使创建**随机失败**，修改注入器`InitMission()`的签名，增加`error`返回值

```jsx
func InitMission(name string) (Mission, error) {
  wire.Build(NewMonster, NewPlayer, NewMission)
  return Mission{}, nil
}
```

## 高级特性

下面简单介绍一下`wire`的高级特性。

### **ProviderSet**

有时候可能多个类型有相同的依赖，我们每次都将相同的构造器传给`wire.Build()`不仅繁琐，而且不易维护，一个依赖修改了，所有传入`wire.Build()`的地方都要修改。为此，`wire`提供了一个`ProviderSet`（构造器集合），可以将多个构造器打包成一个集合，后续只需要使用这个集合即可。

我们观察到两次调用`wire.Build()`都需要传入`NewMonster`和`NewPlayer`。两个还好，如果很多的话写起来就麻烦了，而且修改也不容易。这种情况下，我们可以先定义一个`ProviderSet`：

```jsx
var monsterPlayerSet = wire.NewSet(NewMonster, NewPlayer)
```

### **结构构造器**

因为我们的`EndingA`和`EndingB`的字段只有`Player`和`Monster`，我们就不需要显式为它们提供构造器，可以直接使用`wire`提供的**结构构造器**（Struct Provider）。结构构造器创建某个类型的结构，然后用参数或调用其它构造器填充它的字段。例如上面的例子，我们去掉`NewEndingA()`和`NewEndingB()`，然后为它们提供结构构造器:

```
var monsterPlayerSet = wire.NewSet(NewMonster, NewPlayer)

var endingASet = wire.NewSet(monsterPlayerSet, wire.Struct(new(EndingA), "Player", "Monster"))
var endingBSet = wire.NewSet(monsterPlayerSet, wire.Struct(new(EndingB), "Player", "Monster"))

func InitEndingA(name string) EndingA {
  wire.Build(endingASet)
  return EndingA{}
}

func InitEndingB(name string) EndingB {
  wire.Build(endingBSet)
  return EndingB{}
}
```

### **绑定值**

有时候，我们需要为某个类型绑定一个值，而不想依赖构造器每次都创建一个新的值。有些类型天生就是单例，例如配置，数据库对象（`sql.DB`）。这时我们可以使用`wire.Value`绑定值，使用`wire.InterfaceValue`绑定接口。例如，我们的怪兽一直是一个`Kitty`，我们就不用每次都去创建它了，直接绑定这个值就 ok 了:

```jsx
var kitty = Monster{Name: "kitty"}

func InitEndingA(name string) EndingA {
  wire.Build(NewPlayer, wire.Value(kitty), NewEndingA)
  return EndingA{}
}

func InitEndingB(name string) EndingB {
  wire.Build(NewPlayer, wire.Value(kitty), NewEndingB)
  return EndingB{}
}
```

### **结构字段作为构造器**

有时候我们编写一个构造器，只是简单的返回某个结构的一个字段，这时可以使用`wire.FieldsOf`简化操作。现在我们直接创建了`Mission`结构，如果想获得`Monster`和`Player`类型的对象，就可以对`Mission`使用`wire.FieldsOf`：

```
func NewMission() Mission {
  p := Player{Name: "dj"}
  m := Monster{Name: "kitty"}

  return Mission{p, m}
}

// wire.go
func InitPlayer() Player {
  wire.Build(NewMission, wire.FieldsOf(new(Mission), "Player"))
}

func InitMonster() Monster {
  wire.Build(NewMission, wire.FieldsOf(new(Mission), "Monster"))
}

// main.go
func main() {
  p := InitPlayer()
  fmt.Println(p.Name)
}
```

### **清理函数**

构造器可以提供一个清理函数，如果后续的构造器返回失败，前面构造器返回的清理函数都会调用：

```
func NewPlayer(name string) (Player, func(), error) {
  cleanup := func() {
    fmt.Println("cleanup!")
  }
  if time.Now().Unix()%2 == 0 {
    return Player{}, cleanup, errors.New("player dead")
  }
  return Player{Name: name}, cleanup, nil
}

func main() {
  mission, cleanup, err := InitMission("dj")
  if err != nil {
    log.Fatal(err)
  }

  mission.Start()
  cleanup()
}

// wire.go
func InitMission(name string) (Mission, func(), error) {
  wire.Build(NewMonster, NewPlayer, NewMission)
  return Mission{}, nil, nil
}
```

reference:

[Go 每日一库之 wire](https://zhuanlan.zhihu.com/p/110453784)

[[]] at main · [[]]

[[]] at main · [[]]#advanced-features)