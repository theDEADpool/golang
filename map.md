# GO语言 #
## map ##

**定义**

	var <varName> map[<keyType>]<valueType>

	var m map[int]string = make(map[int]string)
	m := make(map[int]string，10)

`make`方法第二个参数表示`map`的容量。该参数可以省略，当`map`超过容量时会自动扩容。但为了减少不必要的内存分配，应该指定合适的容量大小。

如果已知`map`中每一项的内容，还可以使用这种方式：

	m := map[int]string{1:"a", 2:"b", 3:"c"}

**特性**

`map`是一种`key-value`格式的数据结构。

`map`的`key`要求必须使用能够通过`==`或`!=`比较运算的类型。因此`key`不可以是函数，`map`或者`slice`。

`map`的`value`也可以是一个`map`，比如`var m map[int]map[string]int`。

在使用`map`之前一定要调用`make`方法对`map`进行初始化。相当于C语言中的`alloc`。对于`var m map[int]map[string]int`这种`value`也是`map`的`map`。给`value`赋值的时候也要进行初始化操作。

	m := make(map[int]map[string]int)
	m[0] = make(map[string]int)
	m[0]["first"] = 100

	fmt.Println(m)
	//m = [0:[first:100]]

此时如果执行`m[1]["second"] = 90`，则会报错。因此`m[1]`这个`map`没有进行初始化。

想要判断map是不是被初始化可以通过下面这种方法：
	
	_, ret := m[1]["second"]

这里两个返回值，第一个是`map`中`key`对应的`value`。第二个是一个bool类型的返回值表示`map`中该项有没有被初始化。如果没有初始化，则`ret = false`。

`map`的存储是无序的，即使`key`是`0, 1, 2, 3`这种，遍历整个`map`的得到的结果也不一定就是按照`0, 1, 2, 3`这个顺序排列的。这个要注意。

**常用方法**

**增加和修改**

	m := make(map[int]string, 10)
	m[0] = "ok"

	fmt.Println(m)
	//m = [0:ok]

键值对不存在，则自动创建。

	m[0] = "hello"

	fmt.Println(m)
	//m = [0:hello]

**查询**

	a := m[0]

	fmt.Println(a)
	//a = hello

**删除**

	delete(m, 0)

	fmt.Println(m)
	//m = []

**迭代操作**

	for k,v :=range map {
		...
	}

类似于`slice`，此处得到的`k`是map中key值的拷贝，而`v`则是`map[k]`的拷贝。对于`k`和`v`的修改，都不会影响到`map`中的值。

	sm = make([]map[int]string)
	for i := range sm {
		sm[i] = make(map[int]string, 1)
	}

