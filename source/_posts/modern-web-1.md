---
title: 现代Web技术（一）——WebAssembly
date: 2024/9/20 8:54:00
---

# 现代Web技术（一）——WebAssembly

## 什么是WebAssembly
[MDN 官方解释](https://developer.mozilla.org/zh-CN/docs/WebAssembly)  
对于应用开发者，可以将WebAssembly看作一种IR（中间语言），类似于Java的字节码，C#的IL，LLVM IR，可以在浏览器中解释并运行。  
浏览器通过将WebAssembly编译成本地代码，提高性能的同时还能够跨平台。

## 为什么要使用WebAssembly
WebAssembly的性能比JavaScript更高，可以实现更复杂的计算任务。在传统场景中，复杂的计算任务一般交由JS worker线程处理，但worker不会提升性能，只是将任务从主线程中分离出来。
随着Web应用的发展，更多需求放在前端，使得Web应用更加复杂。WebAssembly可以提供更好的性能，使得Web应用可以处理更多的任务。  
同时，Emscripten等工具可以将C/C++等语言编译成WebAssembly，使得WebAssembly可以更好的与现有的代码库结合。通过这种方式，核心业务代码可以实现真正意义上的多端运行。  

## WebAssembly的缺点
WebAssembly相比于在宿主机的本地代码，内存布局大不相同。这种设计让Java和Python等GC语言更难以实现，性能没有多大提升。  
同时WebAssembly的运行环境没有操作系统的内核，所以没有系统调用，无法直接访问文件系统，网络等资源。  

## WebAssembly生态
[Emscripten](https://github.com/emscripten-core/emscripten) 基于LLVM编译器，可以将C/C++等语言编译成WebAssembly。  
[Wasmer WebAssembly Runtime](https://github.com/wasmerio/wasmer) 一个WebAssembly运行时，可以在多种平台上运行WebAssembly代码。  
[编译 Rust 为 WebAssembly](https://developer.mozilla.org/zh-CN/docs/WebAssembly/Rust_to_Wasm) Rust可以编译成WebAssembly。  

## WebAssembly的基本使用
在浏览器中使用WebAssembly，可以通过`WebAssembly.instantiate`和`WebAssembly.instantiateStreaming`方法加载WebAssembly模块。其中获取WebAssembly模块完全由开发者自行决定，浏览器接受BufferSource和Response对象或返回Response的Promise。
```typescript
/** [MDN Reference](https://developer.mozilla.org/docs/WebAssembly/JavaScript_interface/instantiate_static) */
function instantiate(bytes: BufferSource, importObject?: Imports): Promise<WebAssemblyInstantiatedSource>;
function instantiate(moduleObject: Module, importObject?: Imports): Promise<Instance>;
/** [MDN Reference](https://developer.mozilla.org/docs/WebAssembly/JavaScript_interface/instantiateStreaming_static) */
function instantiateStreaming(source: Response | PromiseLike<Response>, importObject?: Imports): Promise<WebAssemblyInstantiatedSource>;

const module = await instantiateStreaming(fetch('example.wasm'))
```
既然JavaScript和WebAssembly是两门独立的语言，那在实际操作中，必然会涉及到FFI（Foreign Function Interface）的问题。  
浏览器通过WAT格式中定义的导出函数，将WebAssembly模块中的函数暴露给JavaScript调用。同时JavaScript还可以通过直接访问WebAssembly的内存进行数据交互。  
```typescript
// 接上面的代码
console.log(module.instance.exports['add'](1, 2)) // 调用WebAssembly中的add函数
```
```c++
// C++代码
extern "C" int add(int a, int b) {
    return a + b;
}
```
```wat
(module
  (func $add (param i32 i32) (result i32)
    get_local 0
    get_local 1
    i32.add)
  (export "add" (func $add)))
```

## WebAssembly在实际项目中的应用
虽然浏览器API已经涵盖大部分JS和WebAssembly的交互需求，但是实际项目中，Emscripten提供的模拟环境能够实现更多的功能。
下面基于Emscripten自动生成的warpper代码，展示了如何在JavaScript中调用WebAssembly的函数。  
```javascript
// 加载WebAssembly模块
import wasmModule from './wasmModule.js' // 由Emscripten生成的JS模块

const wasm = await wasmModule() // Emscripten自动加载WebAssembly模块

console.log(wasm['_add'](1, 2)) // 调用WebAssembly中的add函数
```
```cpp
// C++代码，extern "C"表示以C的方式导出函数，EMSCRIPTEN_KEEPALIVE表示保留函数，避免被编译器优化
extern "C" int EMSCRIPTEN_KEEPALIVE add(int a, int b) {
    return a + b;
}
```

### 文件系统
WebAssembly无法直接访问文件系统，但Emscripten提供了一套文件系统API，可以在内存中模拟文件系统。  
```javascript
import wasmModule from './wasmModule.js'

const wasm = await wasmModule()

wasm.FS.writeFile('nums.txt', '1 2 3 4 5')

console.log(wasm['_add']('nums.txt'))
```
```cpp
#include <emscripten.h>
#include <fstream>
#include <iostream>
#include <string>

extern "C" int EMSCRIPTEN_KEEPALIVE add(const char* filename) {
    std::ifstream file(filename);
    int sum = 0;
    int num;
    while (file >> num) {
        sum += num;
    }
    return sum;
}
```

### HTTP请求
WebAssembly无法直接访问网络资源，但Emscripten提供了一套HTTP请求API，可以在内存中模拟网络请求。底层原理是将请求发送给JS，由XMLHTTPRequest完成请求。  
```cpp
#include <emscripten.h>
#include <string>
#include <iostream>

int main() {
    uint8_t* buf = nullptr;
    int buf_size, errorno;
    emscripten_wget_data("https://example.com/file.txt", &buffer, &buf_size, &errorno);
    std::string content(reinterpret_cast<char*>(buf), buf_size);
    std::cout << content << std::endl;
    return 0;
}
```

### Wasmer运行时
Wasmer是一个WebAssembly运行时，可以在多种平台上运行WebAssembly代码，同时兼容多种工具链编译得到的WebAssembly。  
以下是使用Emscripten编译的LLVM工具链，将C++代码编译成WebAssembly，并在Wasmer中运行的示例。  
```javascript
import { init, WASI } from 'https://esm.sh/@wasmer/wasi@1.1.2'
import Clang from './clang.js'
import Lld from './lld.js'

await init()

export const compileAndRun = async (mainC) => {
    const clang = await Clang()
    clang.FS.writeFile('main.cpp', mainC)
    await clang.callMain([
        '-c',
        'main.cpp'
    ])
    const mainO = clang.FS.readFile('main.o')

    const lld = await Lld()
    lld.FS.writeFile('main.o', mainO)
    await lld.callMain([
        '-flavor',
        'wasm',
        '-L/lib/wasm32-wasi',
        '-lc',
        '-lc++',
        '-lc++abi',
        '/lib/clang/18/lib/wasi/libclang_rt.builtins-wasm32.a',
        '/lib/wasm32-wasi/crt1.o',
        'main.o',
        '-o',
        'main.wasm',
    ])
    const mainWasm = lld.FS.readFile('main.wasm')

    const wasi = new WASI({})
    const module = await WebAssembly.compile(mainWasm)
    const instance = await WebAssembly.instantiate(module, {
        ...wasi.getImports(module)
    })

    wasi.start(instance)
    const stdout = await wasi.getStdoutString()
    return stdout
}
```
