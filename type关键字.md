# type关键字
## 类型定义

    type <类型1> <类型2>


有别于C语言的typedef，这里类型1并非类型2的别名。而是在类型2的基础上创建的一个新类型。两个类型是不同的类型，但是两个类型的底层数据结构是一样的。  
类型1和类型2的变量不能直接赋值，必须要进行显示的类型转换。

    type int32_t int

    func main() {
        var a int
        var b int32_t
        
        a = 1
        b = a
    }

    ./main.go:18:7: cannot use a (type int) as type int32_t in assignment

    需要写成b = int32_t(a)

为什么要有类型定义？  
根据《Go programming language》上面温度变量的例子可知，这种写法可以避免两个含义相近（都是表示温度的变量）之间直接赋值导致不必要的错误。

## 类型别名
Golang在1.9版本中加入了这个特性。这就类似于C语言中的typedef的作用。

    type <类型1> = <类型2>

类型1和类型2是相同的类型，不仅能够互相赋值，还共享定义的方法。

    type T1 struct {
        a int
    }

    func (t1 *T1)InitT1() {
        t1.a = 1
        fmt.Println("init T1", t1.a)
    }

    type T2 = T1

    func (t2 *T2)InitT2() {
        t2.a = 2
        fmt.Println("init T2", t2.a)
    }

    func main() {
        var x T1
        var y T2
        
        x.InitT1()
        x.InitT2()
        y.InitT1()
        y.InitT2()
    }

    输出：
    init T1 1
    init T2 2
    init T1 1
    init T2 2

T1的方法T2可以使用，T2的方法T1也可以使用。

type还可以用于定义结构体，接口等在对应文档已有说明，不再赘述。