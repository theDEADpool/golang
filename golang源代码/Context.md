# Context
## 作用
用于goroutine上下文之间传递信息，从而实现协程之间的控制。

golang后台服务器在收到请求的时候通常会创建协程进行处理。协程在处理过程中又会创建子协程。当某一个协程处理异常导致请求失败的时候，需要有一种机制通知其他协程结束。Context包就提供了这种机制。

## 数据结构
**Context**  

    type Context interface {
        Deadline() (deadline time.Time, ok bool)
        Done() <-chan struct
        Err() error
        Value(key interface{}) interface{}
    }

**cancelCtx**  

    type cancelCtx struct {
        Context
        mu sync.Mutex
        done chan struct{}
        children map[canceler]struct{}
        err error
    }

*cancel方法*  
入参removeFromParent：是否需要将当前Context从其父Context的子Context列表中移除。  
入参err：错误信息。  
出参：无。  
1.c.err说明当前Context已经被取消过了，不能再被取消一次。  
2.关闭当前Context的done频道。  
3.遍历当前Context所有的子Context，逐个调用cancel函数执行取消操作。  
4.根据入参判断是否需要将当前Context从其父Context的列表中去除。

*Done方法*  
入参：无。  
出参：一个只读的channel。  
1.c.done频道只有在Done方法被调用的时候才会创建。  
2.将c.done以只读的方式返回。调用者只能从中读取数据。  
3.这里有一个问题就是Context包中所有的代码，并没有向c.done频道写数据的操作。那么调用者怎么从c.done频道中读到数据？
4.通过close(c.done)会让所有<-c.Done()的操作返回。这是一种广播机制，同时所有在监听c.Done()频道的调用者，频道已经被关闭。

**timerCtx**

    type timerCtx struct {
        cancelCtx
        timer *timer.Timer
        deadline time.Time
    }
cancelCtx：保存父Context。

*Deadline方法*  
1.返回创建timerCtx时指定的超时时间。

*cancel方法*  
1.调用c.cancelCtx.cancel方法，取消了当前Context的父Context。同时由于cancelCtx的机制，当前Context作为子Context也被取消了。所以可以看到该方法里没有单独取消当前Context。

## API接口
**Background**  
入参：无。  
出参：Context接口。  
1.函数返回background变量。  
2.background变量是包内定义的变量。由emptyCtx进行初始化。  
3.emptyCtx是int类型的别名，但实现了Context接口定义的所有方法所以可以作为函数的返回值。  
4.该函数作用于创建一个空的Context，作为后续Context的父Context。

**WithCancel**  
入参parent：父Context，一般是Background创建的Context。  
出参ctx：创建的新Context。  
出参cancel：新Context对应的cancel函数。  
1.调用newCancelCtx创建一个cancelCtx数据结构，入参parent赋值给Context字段。  
2.propagateCancel判断如果父Context取消了，那么子Context也要取消。如果父Context没有取消，则建立父子Context之间的关系。  
3.parent.Done() == nil这个条件，只有Background创建的Context才满足。相当于这个parent是永远不会取消的。  
4.parentCancelCtx里面有个.(type)的写法。这个写法目的是用获取parent的真实类型。c是指向真实类型对象的指针。这个写法必须和switch...case结合使用。  
5.parentCancelCtx内部实现，如果父Context是cancelCtx，则返回就是父Context的指针。如果父Context是timerCtx，则返回的是父Context的cancelCtx对应的对象。如果父Context是valueCtx，那么parent = c.Context，实际上就是取父Context的父Context重新进行判断。  
6.如果父Context已经取消，那么调用child.cancel取消子Context下属的所有子Context，但是没有接触父子Context之间的关系。个人理解应该是没有必要，因为在这个流程里面父Context已经取消了。    
7.如果父Context没有取消，则建立父子Context之间的关系。  
8.上面两个步骤都要加锁，因为同一个父Context可能用于多个不同的协程创建不同的子Context。  
9.如果找不到父Context的真实类型，说明父Context是一个自定义的结构。那么就通过一个协程来监听父Context的Done方法。  
10.回到WithCancel的流程中，已经完成了父子Context之间的关系处理。返回新创建的CancelCtx结构以及对应的cancel函数。  

**WithDeadline**  
入参parent：父Context。  
入参d：超时时间。  
出参：可能是cancelCtx数据结构，也可能是timerCtx数据结构。  
出参：取消函数。  
1.如果父Context的超时时间存在且比入参指定的超时时间早的话，那么函数实际创建一个withCancel的结构。这样父Context超时之后直接通过withCancel的机制取消所有的子Context即可。  
2.将父Context重新封装为cancelCtx类型。作为当前timerCtx的父Context。个人理解这里之所以要将父Context重新封装为cancelCtx主要是想利用cancelCtx的机制方便建立父子Context之间的关联关系。毕竟timerCtx没有定义独立的成员来保存父子Context之间的关系。  
3.propagateCancel来建立父子Context之间的关系。  
4.如果当前时间已经超过了入参指定的超时时间，那么直接取消当前的Context。返回出去的实际上一个已经取消的Context。  
5.如果当前时间没有超过入参指定的超时时间，那么创建一个定时器。超时之后调用cancel方法。  
6.由于cancel方法也是作为返回值返回的。说明timerCtx类型的Context也是可以人工取消，不等待定时器超时。

