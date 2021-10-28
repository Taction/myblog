---
type: docs
title: "基于WebAsssembly的Serverless探索1:wasm概述"
linkTitle: "wasm概述"
date: 2021-10-17T16:16:20+08:00
draft: false
---

### 一、介绍

#### 1.1 什么是 WebAssembly

> 运行在沙盒中的二进制格式代码

WebAssembly 是一种低层次的二进制格式代码，体积小，因此加载和执行速度快。你不需要直接编写 WebAssembly 代码，而是可以从其他高级语言编译而来。**它是一种可移植且通用的二进制指令格式，用于在虚拟机中进行内存安全、沙盒执行。**可以用 C、C++、Rust、AssemblyScript、C#、Go、Swift 等多种语言编写程序，并将它们编译为 Wasm。

WebAssembly对比docker冷启动延迟可以降低90%以上，对比于js通常具有两倍的运行速度优势。**由于其冷启动快、体积小、执行速度快、灵活性高、安全性高和具有可移植性，因此正在被逐渐被应用于serverless和边缘计算领域。**

#### 1.2 系统接口

##### 1.2.1 WASI

**WebAssembly System Interface (WASI)是一套标准的函数接口定义，作为wasm与“系统”进行交互的标准函数；类似于操作系统为运行的程序提供的标准调用接口一样。** 这个接口的设计考虑到了Wasm 的目标——可移植性和安全性。WASI 是二进制兼容的，这意味着 Wasm二进制文件可以在不同的具体系统（如 Linux 和 Windows）之间移植，也可以在不同浏览器之间移植。Wasm运行时应该实现这些接口并将其转换为底层的具体操作系统。WASI 还在安全性方面进行了创新，超越了经典的粗粒度访问控制（由于中间存在运行时抽象，所以在运行时中可以定义任何标准的或者自定义的访问控制）。

##### 1.2.2 自定义系统接口

WebAssembly对系统的函数调用并不局限于wasi，而是取决于运行wasm的运行时宿主向WebAssembly导入了什么函数。一个比较典型的例子就是enovy和mosn目前在用的一套代理协议标准：`proxy_abi_version_0_2_0`，运行时在启动wasm程序的时候，将此类函数注入，wasm中就可以直接使用这些函数了。

当然我们也可以任意的自定义webassembly可以从宿主系统调用的函数，你将在后面模型试验的例子里看到两个典型的例子。为WebAssembly程序提供了函数计算和网络访问两个函数。这两个函数不在任何的interface标准中，完全是我们自定义的函数。

下图展示了部分`proxy_abi_version_0_2_0`中定义的函数，是运行时将这些函数注册给wasm的代码。

![image-20210708135302772](/images/registerfunctions.png)

#### 1.3 导出函数

> 利用这个特点，可以将wasm作为插件机制来使用。

除了WebAssembly可以调用运行时宿主提供的“系统调用”外，WebAssembly可以导出实现的函数，宿主机可以调用此函数，这类函数被称为导出函数。

![image-20210708142246927](/images/functionoverview.png)

#### wasm的安全

Wasm 使用故障隔离技术来沙箱执行模块。与宿主环境的互动只能通过导入的函数实现。这种机制代表了一个“安全的外部函数接口”，因为它可以与外部环境通信，但不能脱离沙箱。这是 Wasm 的一个重要安全特性。从主机的角度来看，没有导入的模块没有副作用。甚至打印 Hello World！都需要导入打印功能。同样，写入文件、网络套接字或读取时钟都需要导入。

##### 内存安全

在 Wasm 中，所有内存访问都仅限于模块的线性内存。该存储器与代码空间分开，可防止程序覆盖指令。程序可以只能在自己的执行环境中运行，不能逃逸。这意味着 Wasm 运行时可以安全地执行多个不受信任的模块，它们具有自己的线性内存，在同一进程内存空间中并且不需要额外的隔离。特别是，“无法在 WebAssembly 中表达对任意内存位置的读取和写入”，因为 Wasm 的内存指令使用偏移量而不是地址。此外，运行时的边界检查可确保指令仅写入线性内存，这是使 Wasm 成为一种轻量级容器技术的关键方面。

#### 1.4 代码案例

接下来以一个简单的案例了解下，其中wasm程序导出一个乘法函数。wasm运行时提供一个加法函数供wasm调用。其中wasm提供的函数是可以任意定义实现的，这一点在后面模型试验的案例中可以看到。

其结构及接口定义如下图所示：

![image-20210708143343468](/images/example.png)

![image-20210708164115373](/images/structure.png)

##### 1.4.1 wasm代码

wasm代码中可以看到对于加法**add**函数，只定义了函数签名，没有定义实际的逻辑，表明这个函数是一个导入的由运行时提供的函数。同时提供了**multiply**函数，并标注这是一个导出函数，编译器会自动帮我们进行导出工作。具体的代码如下：

```
// This calls a JS function from Go.
func main() {
   println("webassembly打印，调用加法函数2+3= ", add(2, 3)) // expecting 5
}

// This function is imported from JavaScript, as it doesn't define a body.
// You should define a function named 'main.add' in the WebAssembly 'env'
// module from JavaScript.
func add(x, y int) int


// This function is exported to JavaScript, so can be called using
// exports.multiply() in JavaScript.
//export multiply
func multiply(x, y int) int {
   return x * y
}
```

##### 1.4.2 运行时宿主代码

```
func main() {

   // 加载wasm字节码
   pwd, _ := os.Getwd()
   path := filepath.Join(pwd, "../tinygowasm/number.wasm")
   wasm, err := ioutil.ReadFile(filepath.Clean(path))
   // 省略具体加载的细节

   // 为webassembly提供加法函数
   importObject.Register(
      "env",
      map[string]wasmer.IntoExtern{
         "main.add": wasmer.NewFunction(
            store,
            wasmer.NewFunctionType(wasmer.NewValueTypes(wasmer.I32, wasmer.I32, wasmer.I32, wasmer.I32), wasmer.NewValueTypes(wasmer.I32)),
            func(args []wasmer.Value) ([]wasmer.Value, error) {
               return []wasmer.Value{wasmer.NewI32(args[0].I32() + args[1].I32())}, nil
            },
         ),
      },
   )
   instance, err = wasmer.NewInstance(module, importObject)
   check(err)

   // 运行时宿主调用webassembly中定义的乘法函数
   multi, err := instance.Exports.GetFunction("multiply")
   check(err)
   i, err := multi(3, 5)
   check(err)
   fmt.Printf("运行时调用wasm乘法函数，运算3X5= %d\n", i)
   // 运行webassembly主程序，即go文件中定义的main函数
   start, err := instance.Exports.GetFunction("_start")
   check(err)
   start()
}
```

##### 1.4.3 运行结果

可以通过控制台打印看到，无论是wasm调用运行时提供的加法函数，还是运行时调用wasm中定义的乘法函数都可以正常运行

![image-20210708144330696](/images/result.png)

#### 1.5 应用场景及发展方向

1. 对于WebAssembly的后续发展思路分为两个大方向，
   1. 第一个是对接k8s调度层（调度器底层），通过k8s来调度wasm程序的运行
   2. 第二个是以sidecar作为WebAssembly的运行时引擎，通过控制面来调控运行的程序。
2. 业务场景的思路：在抗流量设施中，或者有突发流量的场景中，可以改造成这种具有弹性的WebAssembly的运行模式。
3. 通过WebAssembly实现彻底的语言无关的流量治理，使用统一的apm体系，打造新一代运行时
4. 数据面策略型逻辑代码的处理：流量治理、遥测策略、安全规则校验、服务自定义策略......

### 二、模型试验

对于数字加减的模型试验我们在介绍原理的时候已经进行过介绍了，这表明对于纯逻辑计算，无论是wasm调用运行时还是运行时调用wasm函数都是没有问题的。

接下来进行一个稍微复杂一些的模型试验，在现实世界中通常进行最多的就是网络请求的处理。因此在接下来的模型试验中，运行时为webassembly提供了一个“网络接口”函数`getHttpResp`。这个函数的功能是接收一个url参数，并且请求此url，并将请求得到的结果返回给wasm。wasm逻辑就是调用此函数并打印此次请求的结果。

![image-20210708144948920](/images/examplesimple.png)



#### 2.1 wasm代码

wasm代码中定义了一个url，调用`getHttpResp`函数并打印请求结果

```
func main() {
   url := "http://apis.juhe.cn/ip/ip2addr"
   var rvs int
   var raw *byte
   st := getHttpResp(url, &raw, &rvs)
   if st != 1 {
      println("请求错误,code", st)
   }
   println("wasm打印，调用运行时网络请求函数并打印结果:", url, string(RawBytePtrToByteSlice(raw, rvs)))
}

func getHttpResp(s string, returnValueData **byte, returnValueSize *int) int
```

#### 2.2 运行时宿主代码

wasm运行时主要代码就是为wasm提供`getHttpResp`函数，在接下来的代码示例中去掉在上文中提到的其他细节，主要展示提供`getHttpResp`函数部分代码.此代码的主要逻辑就是先获取到传参url，请求url，将请求得到的结果拷贝到共享内存因此wasm可以从共享内存中获取请求结果。并返回wasm一个数字作为是否请求成功的标记。

```
importObject.Register(
   "env",
   map[string]wasmer.IntoExtern{
      "main.getHttpResp": wasmer.NewFunction(
         store,
         wasmer.NewFunctionType(wasmer.NewValueTypes(wasmer.I32, wasmer.I32, wasmer.I32, wasmer.I32, wasmer.I32, wasmer.I32), wasmer.NewValueTypes(wasmer.I32)),
         func(args []wasmer.Value) ([]wasmer.Value, error) {
            in, bufferType, returnBufferData, returnBufferSize, _, _ := args[0].I32(), args[1].I32(), args[2].I32(), args[3].I32(), args[4].I32(), args[5].I32()
            m, err := instance.Exports.GetMemory("memory")
            if err != nil {
               return nil, err
            }
            mem := m.Data()
            url := string(mem[in:in+bufferType])
            res, err := http.Get(url)
            if err != nil {
               return []wasmer.Value{wasmer.NewI32(0)}, nil
            }
            byt, err := ioutil.ReadAll(res.Body)
            res.Body.Close()
            if err != nil {
               return []wasmer.Value{wasmer.NewI32(0)}, nil
            }
            blen := int32(len(byt))
            // 由于没做溢出检查，先最多取100个字符防止溢出
            if blen > 100 {
               blen = 100
            }
            byt = byt[:blen]
            // todo 需要check 防止 overflow
            var addrIndex int32
            malloc, err := instance.Exports.GetRawFunction("malloc")
            if err == nil {
               addr, err := malloc.Call(blen)
               if err != nil {
                  return []wasmer.Value{wasmer.NewI32(0)}, nil
               }
               addrIndex , _ = addr.(int32)
            } else {
               addrIndex = returnBufferData + int32(blen) + 32
            }

            copy(mem[addrIndex:], byt[:blen])
            binary.LittleEndian.PutUint32(mem[returnBufferSize:], uint32(blen))
            binary.LittleEndian.PutUint32(mem[returnBufferData:], uint32(addrIndex))
            s := RawBytePtrToByteSlice((*byte)(unsafe.Pointer(uintptr(unsafe.Pointer(&mem)) + uintptr(addrIndex - blen))),blen)
            _=s
            watch := mem[in:addrIndex+blen]
            _=watch
            return []wasmer.Value{wasmer.NewI32(1)}, nil
         },
      ),
   },
)
```

#### 2.3 网络试验结果

程序正常运行并且在wasm中可以正常打印网络请求的结果。

![image-20210708150016833](/images/resultprint.png)

### 额外说明

##### 通过共享内存交互

wasm只支持4种数据结构：int32、int64、float32、float64。当传递其他类型的数据，比如字符串时，需要通过共享内存来通信。通过

