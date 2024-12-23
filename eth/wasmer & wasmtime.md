## Wasmer & wasmtime

Wasmer 和 Wasmtime 是两个常用的 WebAssembly 运行时

​	•**Wasmer**：Wasmer 的目标是成为一个跨平台的 WebAssembly 运行时，允许在多种环境中运行 WebAssembly 模块，比如服务器、桌面应用、区块链等。它专注于通用性，能够运行在多个架构上，如 macOS、Linux、Windows 甚至移动设备。Wasmer 提供了丰富的 API 和插件，以便开发者在不同的环境中集成 Wasm。

​	•**Wasmtime**：Wasmtime 是由 Bytecode Alliance 开发，目标是一个轻量级、快速的 Wasm 运行时，主要专注于 WebAssembly System Interface (WASI) 标准，旨在支持模块化的安全应用程序。它更适合被嵌入在服务器端应用和基于 WASI 的环境中，例如云计算环境、插件系统等。

#### 兼容性与扩展性

​	•**Wasmer**：Wasmer 支持 Emscripten、WASI 和 WASM 标准，同时通过插件机制可以支持不同类型的 Wasm 模块。其设计使得 Wasmer 更适合需要灵活性、跨平台的应用场景，比如在不同架构的区块链节点中运行 Wasm 合约。

​	•**Wasmtime**：Wasmtime 专注于对 WASI 标准的支持，非常适合服务器端和容器化应用。Wasmtime 直接与 WASI 兼容，使其更适合实现高效的无服务器应用、云端插件和嵌入式场景中。

#### **执行性能**

​	•**Wasmer**：Wasmer 支持 AOT（提前编译）和 JIT（即时编译），使得它在不同场景中能够灵活切换编译模式。AOT 支持使得 Wasmer 能够实现更快的启动时间，对于需要重复执行的模块，AOT 可以大幅提升性能。

​	•**Wasmtime**：Wasmtime 主要依赖 Cranelift JIT 编译器，这使得它在代码优化和执行性能上表现出色。Cranelift 专为 Wasm 优化，使得 Wasmtime 的 JIT 模式非常高效，尤其适合高性能要求的 WebAssembly 应用。

#### **嵌入和 API 支持**

​	•**Wasmer**：提供了 C、Python、Go 和 Rust 等多种语言的 SDK，使得 Wasmer 能够在多种编程语言中被嵌入使用。它的 API 设计适合希望跨语言使用 Wasm 模块的场景，方便不同开发者集成 Wasmer。

​	•**Wasmtime**：Wasmtime 主要专注于 Rust 生态，提供了 Rust 和 C 的 SDK，虽然没有 Wasmer 那么多的语言支持，但在 Rust 中的集成体验非常好，尤其适合 Rust 环境中的高性能需求。

#### **应用场景**

​	•**Wasmer**：更适合需要跨平台、通用 Wasm 执行的场景，比如在区块链或不同操作系统上运行 Wasm 应用。Wasmer 支持更广泛的编程语言嵌入，因此适合多语言、多平台场景下的 Wasm 应用。

​	•**Wasmtime**：更适合注重 WASI 的应用场景，比如服务器端插件系统、无服务器应用等。由于其专注于轻量和高性能的 JIT 编译器，在需要快速执行和加载的场景中更具优势。



**总结**

​	•**Wasmer**：跨平台、灵活性强，适合多语言和跨架构的使用场景，比如区块链和移动设备。

​	•**Wasmtime**：专注于轻量和高效的 JIT 编译，适合服务器端、高性能、WASI 相关的应用场景。



## JIT && AOT 



**Just - in - Time（JIT）编译**

- **基本原理**
  - JIT 编译是一种在程序运行时将字节码（如 Java 字节码、WebAssembly 字节码等）转换为机器码的技术。在虚拟机（VM）环境中，当代码第一次被执行时，JIT 就会进行编译。例如，在 Java 虚拟机（JVM）中，方法第一次被调用时，JIT 编译器会分析字节码指令序列，将其编译为针对当前硬件平台的机器码。这个编译过程是动态的，根据代码的实际执行路径进行优化。
- **性能特点**
  - **启动速度相对较快**：JIT 编译的程序启动时不需要像 AOT 编译那样预先将所有代码编译为机器码，所以初始启动阶段可以更快地加载字节码并开始执行。以一个包含大量类和方法的 Java 应用为例，在启动时，JVM 只需要加载字节码，而不需要等待所有代码都编译完成，从而减少了启动时间。
  - **执行性能会逐渐优化**：随着程序的运行，JIT 编译器会收集代码的执行信息，如方法调用频率、循环执行次数等。根据这些信息，它可以对频繁执行的代码路径进行深度优化。例如，对于一个经常执行的循环体，JIT 编译器可以将循环内的字节码进行内联（将被调用的方法代码嵌入到调用处）、常量折叠（在编译时计算常量表达式的值）等优化，从而提高执行效率。不过，这种优化是在程序运行过程中逐步进行的。
- **资源占用情况**
  - **内存占用动态变化**：JIT 编译器在运行时会占用一定的内存来存储编译后的机器码和编译过程中的中间数据。随着程序的运行和更多代码被编译，内存占用会逐渐增加。但是，由于它可以根据实际运行情况选择性地编译代码，对于一些不经常执行的代码部分，就不需要占用额外的内存来存储其机器码。
  - **CPU 资源消耗在编译阶段较高**：在代码的编译阶段，JIT 编译器需要消耗 CPU 资源来进行字节码到机器码的转换和优化。不过，这种编译过程通常是在后台线程或者在代码执行的空闲间隙进行的，尽量减少对程序主要执行流程的影响。
- **适用场景**
  - **开发和调试阶段**：在开发过程中，开发人员需要频繁地修改代码并快速看到结果。JIT 编译的快速启动特性使得开发人员可以迅速运行应用程序，检查代码的功能是否正确。而且，由于不需要频繁地重新编译整个程序，调试周期可以更短。
  - **长期运行且代码执行路径复杂多变的应用**：对于一些服务器端应用，如 Web 应用服务器或者消息队列系统，它们需要长时间运行，并且根据用户请求和业务逻辑，代码的执行路径可能会有很大变化。JIT 编译可以在运行过程中根据实际的执行情况对代码进行优化，适应这种复杂多变的执行模式。

**Ahead - of - Time（AOT）编译**

- 基本原理
  - AOT 编译是在程序运行之前，将高级编程语言（如 Java、C# 等）或者中间表示语言（如 WebAssembly）的代码直接编译为目标机器码。这个过程是在开发阶段或者部署之前完成的。例如，在一些移动应用开发中，使用 AOT 编译器将 Java 或者 Kotlin 代码编译为 Android 设备能够直接执行的机器码。在这种情况下，生成的机器码是固定的，在运行时不再进行字节码到机器码的转换。
- 性能特点
  - **启动后的执行性能稳定高效**：由于代码已经预先编译为机器码，程序在启动后可以直接执行机器码，避免了运行时编译的开销。对于一些对性能要求极高的应用，如游戏开发或者实时控制系统，AOT 编译可以提供更稳定、可预测的性能。以一个 3D 游戏为例，游戏的核心渲染和物理计算逻辑通过 AOT 编译后，可以在启动游戏后以最快的速度运行，减少卡顿现象。
  - **整体性能在某些情况下可能更高**：对于一些简单、执行路径固定的程序，AOT 编译可以在编译阶段进行全面的优化，因为它可以看到整个程序的代码结构。例如，对于一个简单的数学计算库，AOT 编译器可以针对特定的数学运算进行向量化优化（利用 CPU 的 SIMD 指令集），从而在运行时获得更高的性能。
- 资源占用情况
  - **启动时文件体积可能较大**：AOT 编译会生成完整的机器码文件，这个文件的大小可能会比字节码文件大很多。例如，一个包含大量功能的 Java 应用，经过 AOT 编译后，机器码文件可能会占用较多的磁盘空间。这在存储资源有限的设备上可能会是一个问题。
  - **运行时内存占用相对稳定**：与 JIT 编译不同，AOT 编译后的程序在运行时不需要额外的内存来存储编译过程中的中间数据或者编译后的机器码（除了正常的程序运行数据），内存占用相对比较稳定，便于在资源受限的环境中进行内存管理。
- 适用场景
  - **对启动速度和性能要求极高的应用**：如一些嵌入式系统或者物联网设备上的应用，这些设备的资源有限，并且对应用的启动速度和执行效率有严格要求。通过 AOT 编译，可以确保应用在这些设备上能够快速启动并高效运行。
  - **代码结构相对固定、执行路径可预测的应用**：例如一些工具软件或者命令行工具，它们的功能和执行流程相对固定。AOT 编译可以在编译阶段对这些固定的代码结构进行优化，提高整体性能。

**JIT 的灵活性和动态优化**：

​	•由于 JIT 是运行时编译的，它有机会基于实际执行情况对热点代码（常用路径）进行优化，像内联、消除不必要的循环等，这在运行较长时间的服务中尤其有利。此外，一些 JIT 编译器可以通过采样来检测瓶颈并重新优化，这在需要适应性强的系统中很有用。

**AOT 的稳定性和低延迟优势**：

​	•AOT 编译由于在执行前就已生成了机器码，因此启动时间相对固定，内存占用较为可控，特别适合启动时间和响应速度要求很高的场景。AOT 编译器可以进行静态优化，但可能无法利用运行时信息进行进一步的性能调优。

**两者的混合模式**：

​	•现代虚拟机中有时会采用 AOT + JIT 的混合模式。部分代码可在 AOT 编译阶段预编译为机器码，以加快初始执行速度，而在运行过程中，通过 JIT 编译对热点路径进行优化，进一步提升性能。这种方式在一些需要启动速度和执行效率兼备的应用中非常有效。