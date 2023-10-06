

## 何为wasm？

WebAssembly 是一种新的编码方式，可以在现代的网络浏览器中运行 － 它是一种低级的类汇编语言，具有紧凑的二进制格式，可以接近原生的性能运行，**并为诸如 C / C ++等语言提供一个编译目标**，以便它们可以在 Web 上运行。它也被设计为可以**与 JavaScript 共存**，允许两者一起工作。

对于网络平台而言，WebAssembly 具有巨大的意义——这为客户端 app 提供了一种在网络平台以接近本地速度的方式运行多种语言编写的代码的方式；在这之前，客户端 app 是不可能做到的。

WebAssembly 被设计为可以和 JavaScript 一起协同工作——通过使用 WebAssembly 的 JavaScript API，你可以把 WebAssembly 模块加载到一个 JavaScript 应用中并且在两者之间共享功能。这允许你在同一个应用中利用 WebAssembly 的性能和威力以及 JavaScript 的表达力和灵活性，即使你可能并不知道如何编写 WebAssembly 代码。



**webassembly的目标**

- 快速、高效、可移植
- WebAssembly 是一门低阶语言，但是它有确实有一种人类可读的文本格式（wat格式），这允许通过手工来写代码，看代码以及调试代码。
- 保持安全——WebAssembly 被限制运行在一个安全的沙箱执行环境中。像其他网络代码一样，它遵循浏览器的同源策略和授权策略
- 不破坏网络——WebAssembly 的设计原则是与其他网络技术和谐共处并保持向后兼容



webassembly如何适应网络平台，从两方面入手：

- 提供运行时宿主环境：vm
- 可以使用网络平台的一些api：如DOM, CSSOM, WebGL, IndexDB, WebAudio





## 前景和发展，有什么用？





## 原理，大致怎么做的？

### 常规使用方式

(使用基于llvm的语言)如C++编写程序 => 借用编译工具(Emscripten) => wassm格式的代码 => 用来加载和运行该模块的 JavaScript”胶水“代码

![img](https://developer.mozilla.org/en-US/docs/WebAssembly/Concepts/emscripten-diagram.png)

1. Emscripten 首先把 C/C++提供给 clang+LLVM——一个成熟的开源 C/C++编译器工具链，比如，在 OSX 上是 XCode 的一部分。
2. Emscripten 将 clang+LLVM 编译的结果转换为一个.wasm 二进制文件。
3. 就自身而言，WebAssembly 当前不能直接的存取 DOM；它只能调用 JavaScript，并且只能传入整形和浮点型的原始数据类型作为参数。这就是说，为了使用任何 Web API，WebAssembly 需要调用到 JavaScript，然后由 JavaScript 调用 Web API。因此，Emscripten 创建了 HTML 和 JavaScript 胶水代码以便完成这些功能。



### 直接写wasm

就像真实的汇编语言一样，WebAssembly 的二进制格式也有文本表示——两者之间 1:1 对应。你可以手工书写或者生成这种格式然后使用这些工具（[WebAssemby text-to-binary tools](http://webassembly.org/getting-started/advanced-tools/)）中的任何一个把它转换为二进制格式。

### wasm基本概念

其实上面提到了js作为胶水代码，那必然有相关的概念，

模块：

内存：

表格：

**实例**：







## wasm基本概念

### C++ => wassm

Wassm: 机器理解的语言

wat: 人可理解的wassm

wassm可以通过一些工具（）转为人可阅读的一段文本，这段文本是树状的表达式，类似抽象语法树(AST)，树里的每个节点都是用括号括起来的，并且每个节点都有自己的标记，代表属于语言的某一部分（是函数？还是参数？等等），而没有用括号包裹的一般都是operation，如下：

```
(func (param i32) (param f32) (local f64)
  local.get 0
  local.get 1
  local.get 2)
```

这是一个函数节点，函数节点包含3个子节点（2个参数节点和一个局部变量节点，参数的标记是param，局部变量1的标记是local），并且还有3个operation（下面会提到一些基本的operation）。



常见节点：

```
module
param
result
export: (export "add" (func $add)), 导出函数$add, 暴露对外部语言的一个接口：add，简写方式：在定义函数的时候即刻导出

import: (import "console" "log" (func $log (param i32))), wassm提供了二级命名空间，如这里的console.log，代表的是即将导入的外部语言的console模块下的log变量，后面func $log (param i32)是导入函数的签名，必须要和原语言的类型一致，对于js这种语言其实不起作用。
WebAssembly has a two-level namespace so the import statement here is saying that we're asking to import the log function from the console module. 

Imported functions are just like normal functions: they have a signature that WebAssembly validation checks statically, and they are given an index and can be named and called.

JavaScript functions have no notion of signature, so any JavaScript function can be passed, regardless of the import's declared signature. （会忽略js的函数签名）

Once a module declares an import, the caller of WebAssembly.instantiate() must pass in an import object that has the corresponding properties.
```



operation：

read：

​	按索引读（其实就是按照函数栈内的偏移量读）：The instruction `local.get 0` would get the i32 parameter, `local.get 1` would 	get the f32 parameter, and `local.get 2` would get the f64 local.

​	具名读：write `local.get $p1` instead of `local.get 0`, etc

​	具名读速度应该会慢点，只是方便开发人员阅读，因为最终还是要转成机器可理解的方式

write：



常见opertaion

```
call ${name}
${type}.const 
```



在讲write操作之前，讨论关于函数的计算模型——堆栈机，因为write操作比较抽象，就是一条简单的指令；







来看写操作：

```
i32.add
```

表示基于i32类型的，从栈内取出两个数字相加并放回栈内

> 如果类型不对会怎么样？





在js里使用wassm（涉及相关api，memory，global）



在wassm里使用js或者其他语言（涉及导入声明和instance传入的一个对象）



**全局变量**

WebAssembly has the ability to create global variable instances, accessible from both JavaScript and importable/exportable across one or more [`WebAssembly.Module`](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Module) instances. This is very useful, as it allows dynamic linking of multiple modules.

wassm可以创建全局变量，这些变量可以来自js或者可导入/导出的[`WebAssembly.Module`](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Module) instances。

```
(module
   (global $g (import "js" "global") (mut i32)) // 声明来自js的全局变量
   (func (export "getGlobal") (result i32) // 函数1
        (global.get $g))
   (func (export "incGlobal")// 函数2，导出
        (global.set $g
            (i32.add (global.get $g) (i32.const 1))))
)
```

**内存操作**

Memory:

The above example is a pretty terrible logging function: it only prints a single integer! What if we wanted to log a text string? To deal with strings and other more complex data types, WebAssembly provides **memory** (although we also have [Reference types](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format#reference_types) in newer implementation of WebAssembly). According to WebAssembly, **memory is just a large array of bytes** that can grow over time. WebAssembly contains instructions like `i32.load` and `i32.store` for reading and writing from [linear memory](https://webassembly.github.io/spec/core/exec/index.html#linear-memory).

对应到js里，其实就是ArrayBuffer

So a string is just a sequence of bytes somewhere inside this linear memory. Let's assume that we've written a suitable string of bytes to memory; how do we pass that string out to JavaScript?



Wassm提供了memory接口，可以暴露wassm的memory给js消费，js可以用ArrayBuffer相关api进行消费



当然也可以创建memory给wassm使用

```
js:
WebAssembly.Memory

wassm:
(import "js" "mem" (memory 1))
```



> **Note:** Above, note the double semicolon syntax (`;;`) for allowing comments in WebAssembly files.



**类型**

WebAssembly currently has four available *number types*:

- `i32`: 32-bit integer
- `i64`: 64-bit integer
- `f32`: 32-bit float
- `f64`: 64-bit float

### [Vector types](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format#vector_types)

- `v128`: 128 bit vector of packed integer, floating-point data, or a single 128 bit type.



### [Reference types](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format#reference_types)



bulk mem operations

seven new built-in operations are provided for bulk memory operations such as copying and initializing, to allow WebAssembly to model native functions such as `memcpy` and `memmove` in a more efficient, performant way.

- `data.drop`: Discard the data in an data segment.
- `elem.drop`: Discard the data in an element segment.
- `memory.copy`: Copy from one region of linear memory to another.
- `memory.fill`: Fill a region of linear memory with a given byte value.
- `memory.init`: Copy a region from a data segment.
- `table.copy`: Copy from one region of a table to another.
- `table.init`: Copy a region from an element segment.

**多线程**





https://github.com/AllenWrong/Self-learning-Record/blob/master/CS143%20Compiler.md



Sidecar:

https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar



