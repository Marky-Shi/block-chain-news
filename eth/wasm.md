## 简介

webassembly/wasm 一个可移植、体积小、加载快并且兼容web的全新格式。 

> Tip：
>
> 在计算机发展早期，程序都是使用机器语言编写，性能方面虽然达到了极致，但是开发效率相对低下，随着技术的发展出现了汇编语言、C、C++，以及后续的java、python（语言越高级，开发效率虽然提升性能却下降，各有取舍。）
>
> 不论技术如何发展计算机唯一能够执行的还是机器语言。所以现代的高级编程语言要么是由编译器编译为机器代码（AOT  Ahead-of-time 预先编译），要么是由解释器直接解释执行（JIT just-in-time 即时编译）
>
> 现代的编译器大多数情况下则是按照模块化的方式设计与实现。典型的做法就是把编译器分为前端、中端、后端。
>
> - 前端：预处理、词法分析、语法分析、语义分析，生成便于后续处理的中间表示（IR ）
> - 中端：对IR进行分析和各种优化，如常量折叠、死代码消除、函数内联。
> - 后端：生成目标代码，把IR转换成与平台相关的汇编代码，最终由汇编器编译为机器代码。

- **高效**：webassembly有一套完整的语义，wasm是体积小且加载快的二进制格式，目标则是充分发挥硬件能力，达到原生执行效率。
- **安全**：webassembly 运行在一个沙箱化的执行环境中，甚至可以在现有的js vm中实现，不仅可以运行在浏览器中，也可以运行在非web环境中。
- **开放**：其设计了一个十分规整的文本格式用来调试、测试、编写程序等，以这种文本格式在web界面上查看wasm模块的源码。
- **不破坏网络**：wasm的设计原则是与其他网络技术和谐共处并保持向后兼容。

### 相关概念：

- **模块**：模块是wasm程序编译、传输和加载的单位，wasm规范定义了两种模块格式：二进制格式和文本格式。
  - 二进制格式：是wasm模块的主要编码格式，后缀为`.wasm` （可以通过高级语言编译，也可以通过wat编译生成）
  - 文本格式： 一般以`.wat` 为后缀，这种格式的文件开发者阅读代码。
  - 内存格式（特例）：wasm的实现通常会把二进制模块解码为内部形式（即内存格式），然后进行处理。
- **内存**：本质上是连续的字节数据，wasm的低级内存存取指令可以对其进行读写操作。
- **表格**：带类型的数组，大小可变。**表格中的存储项不能作为原始字节存储在内存里的对象的引用**
- **实例**：一个模块及其在运行时使用的所有状态，包括内存、表格和一系列导入值。
- **指令集**：类似于jvm，wasm采用了栈式虚拟机和字节码。
- 从语义上讲wasm二进制格式到最终被执行分为三个步骤：
  - 解码：将二进制模块解码为内存格式
  - 验证：对模块进行静态分析，确保模块的结构满足规范要求，且函数字节码不存在违规行为（如不可调用等）
  - 执行：又可以分为实例化和函数调用
  - `.wat`----编译---> `.wasm` --解码 --> `in memeory` -- 实例化-->`instance`




工具安装

默认都装有rust，golang

wabt [WebAssembly/wabt: The WebAssembly Binary Toolkit (github.com)](https://github.com/WebAssembly/wabt)

编译可以参照git上的提示。

需要配置到环境变量中的工具：

- wat2wasm  读取wasm文本格式的文件，检查错误，并将其转化为wasm二进制格式。
  - wat2wasm  test.wat 解析并对xx.wat 文件进行类型检查（.wat文件则是wasm的文本格式，此处只是个例，并无其他意义）
  - wat2wasm test.wat -o test.wasm  解析wat文件并将其输出为 test.wasm 二进制文件。
  - wat2wasm name.wat -v  解析name.wat 并将结果详细输出到终端。
- wasm-strip 裁剪wasm文件
  - wasm-strip name.wasm
- wasm-objdump 打印有关wasm二进制文件内容的信息
  - wasm-objdump test.wasm
    - -d   wasm-objdump -d test.wasm  拆解函数体
    - -x   wasm-objdump -x test.wasm  显示部分详细信息
    - -h   wasm-objdump -h test.wasm  显示头部信息
- wasm2wat  读取webassembly 的二进制格式文件，并将其转换为webassembly 文本格式 即  xx.wasm ===> xx.wat
  - wasm2wat test.wasm -o test.wat 
  - wasm2wat test.wasm --no-debug -o test.wat   无debug信息
  - wasm2wat test.wasm --fold-exprs -o test.wat  尽可能的折叠表达式

其他的工具根据业务需求自行配置，不再赘述。

## 二进制格式

### 结构

看rust一段代码：

```shell
cargo new name --lib
```

`cargo.toml` 配置

```toml
[lib]
crate-type=["cdylib"]
```

```rust
extern "C" {fn random_i32()->i32;}
#[no_mangle]
pub extern "C" fn discard (){
    unsafe {let _=random_i32();}
}
```

将这段源码编译之后

```shell
cargo build -target wasm32-unknown-unknown --release
```

接着执行

```shell
wasm-objdump -x name.wasm


...
Section Details:

Type[2]:
 - type[0] () -> i32
 - type[1] () -> nil
Import[1]:
 - func[0] sig=0 <random_i32> <- env.random_i32
Function[1]:
 - func[1] sig=1 <discard>
Table[1]:
 - table[0] type=funcref initial=1 max=1
Memory[1]:
 - memory[0] pages: initial=16
Global[3]:
 - global[0] i32 mutable=1 <__stack_pointer> - init i32=1048576
 - global[1] i32 mutable=0 <__data_end> - init i32=1048576
 - global[2] i32 mutable=0 <__heap_base> - init i32=1048576
Export[4]:
 - memory[0] -> "memory"
 - func[1] <discard> -> "discard"
 - global[1] -> "__data_end"
 - global[2] -> "__heap_base"
Code[1]:
 - func[1] size=9 <discard>
 ...
```

从上述二进制文件的简略信息中可以看出，该文件的组成部分：`Type` `Import` `Function` `Table` `Memory` `Global` `Export` `Code`,这些都是wasm 二进制文件的项目中的一部分。

wasm规范一共定义了12个段，每个段都分配了ID(0--11)

完整的wasm二进制文件的项目划分见下表：

| magic number  version | 以魔数和版本号开头                                           |
| --------------------- | ------------------------------------------------------------ |
|                       | 自定义段 ID 0 这段是给编译器工具使用的，可以存放函数名等调试信息，或其他附加信息。 |
| TypeSec               | 类型段 ID 1  本例中为`Type[2]` random_i32()->i32             |
| ImprotSec             | 导入段 ID 2  本例中为`Import[1]  ` <random_i32> <- env.random_i32 |
| FuncSec               | 函数段 ID 3  本例中为 `Func[1]` discard                      |
| TableSec              | 表段 ID 4   本例中为`type=funcref initial=1 max=1` 列出了模块内定义定义的所有的表，wasm规定最多只能导入和定义一张表 |
| MemroySec             | 内存段 ID 5 本例中为`pages: initial=16`列出模块定义需要的内存空间 |
| GlobalSec             | 全局段 ID 6 本例中为`Global[3]:` 包含模块内定义的全局变量信息，包含值类型、可变性、初始值。 |
| ExportSec             | 导出段  ID 7 本例中`Export[4]:` discard 方法， 至于memory等则是rust 内部实现的。 |
| StartSec              | 起始段 ID 8  给出模块起始函数索引，不同于其他的段，该段只能有一个项目，主要功能：1）在模块加载后进行一些初始化的工作 2） 将模块变成可执行模块。 |
| ElEMSec               | 元素段 ID 9  对应Table elem 表示表内初始化的数据             |
| CodeSec               | 代码段 ID 10 本例中为`func[1] size=9 <discard>`  存储内部函数局部变量的信息和字节码，与函数段一一对应 |
| DataSec               | 数据段 ID 11 列出内存初始化的数据。                          |



### 实体类型

#### 值类型

wasm定义了四种基本的数值类型 32位整数、64位整数、32位浮点数、64位浮点数。高级语言所支持的类型，如布尔类型、指针、结构体等，都是被编译器翻译成这四种基础的值类型。，不同于高级语言，wasm值类型并不会区分符号

#### 函数类型

即函数的签名，描述函数的参数数量和类型，以及返回值和类型

#### 限制类型

描述表元素的数量和内存页数的上下限

#### 内存类型

描述内存的页数限制

#### 表类型

表的元素类型以及元素限制

#### 全局变量

描述全局变量的类型和可变性

#### 结果类型

函数或表达式的执行结果

#### 导出类型

函数类型、表类型、内存类型和全局变量类型的集合

 

### 指令集

和汇编代码一样，wasm二进制模块中的代码也是由多条的指令构成。

wasm的指令包含两部分的信息：操作码和操作数。操作码是指指令的ID，决定指令将执行的操作；操作数相当于指令的参数，决定指令的执行结果。

操作数又分为：

- 静态操作数：直接编码在指令里边，跟在操作码后边
- 动态操作数：在运行时从操作数栈获取

#### 操作码

wasm的操作码固定一个字节，因此指令集最多有256条指令。wasm规范一共定义了178条指，按照功能可以分为5大类：

1. 控制指令：共13条
2. 参数指令： 共2条
3. 变量指令：共5条
4. 内存指令：共25条
5. 数值指令：共133条

#### 数值指令

格式为前缀操作码 + 子操作码（立即数），在所有的数值指令中，只有4条常量指令和饱和截断指令有立即数：

- i32.const （opcode 0x41） 带一个s32的立即数，使用LEB128有符号编码格式
  - LEB128 编码格式： 是一种变长的编码格式，对于32位的整数来说，编码之后可能变为1--5字节，64位整数可能变为1--10字节.越小的整数，编码之后占用的字节数越小。
    - 采用小端编码方式，低位在前，高位在后
    - 采用128进制，每7bit为一组，由第一个字节的第7位承载，空出来的最高位则是标志位，1表示还有后续字节，0表示没有。
- i64.const (opcode 0x42) 带一个s64的立即数，使用LEB128有符号编码格式
- f32.const  (opcode 0x43) 带一个f32的立即数，固定占用4字节
- f64.const (opcode 0x44) 带一个f64类型的立即数，固定占用8字节
- trunc_sat (opcode 0xFC) 带一个单字节立即数，将浮点数截断为整数

例：

```wat
(module
	(f32.const 1.32) (f32.const 3.1415) (f32.add)
	(f32.trunc_sat_f32_2) (drop)
)
```

将wat文件编译为二进制文件

```shell
wat2wasm name.wat -o name.wasm
```

借助wasm-objdump 工具查看wasm文件的详细信息：

```shell
wasm-objdump -d name.wasm

Code Disassembly:

000016 func[0]:
000017: 43 a4 70 9d 3f             | f32.const 0x1.3ae148p+0
00001c: 43 56 0e 49 40             | f32.const 0x1.921cacp+1
000021: 92                         | f32.add
000022: fc 00                      | i32.trunc_sat_f32_s
000024: 1a                         | drop
000025: 0b                         | end
```

在wat文件中设置了两个f32的常量，并将两个数相加，随后进行截断，根据解析的wasm文件可以看出  `f32.const `---> `0x43`  `turnc_ast`--->` 0xfc`  `end` ---> `0x0b`  `add` -->`0x92`  `drop`--> `0x1a`

同理使用xxd 命令也可以看出相应的操作

```shell
xxd -g -u 1 name.wasm

00000000: 00 61 73 6D 01 00 00 00 01 04 01 60 00 00 03 02  .asm.......`....
00000010: 01 00 0A 12 01 10 00 43 A4 70 9D 3F 43 56 0E 49  .......C.p.?CV.I
00000020: 40 92 FC 00 1A 0B                                @.....
```

#### 变量指令

变量指令共5条，其中3条用于读写局部变量，剩余的2条是用来读写全局变量，立即数都是变量的索引。

例：

```
(module
	
   (global $g1 (mut i32)(i32.const 15 ))  ;; 全局变量
   (global $g2 (mut i32)(i32.const 20 ))


   (func(param $a i32) (param $b i32)  ;;声明函数 获取全局变量，以及函数中的局部变量
       (global.get $g1) (global.set $g2)  
       (local.get $a) (local.set $b)

   )

)
```

编译之后，使用wasm-objdump 工具对二进制文件进行分析

```shell
Code Disassembly:

000025 func[0]:
000026: 23 00                      | global.get 0   // 获取去全局变量
000028: 24 01                      | global.set 1   // 设置全局变量
00002a: 20 00                      | local.get 0    // 获取局部变量
00002c: 21 01                      | local.set 1    // 设置局部变量
00002e: 0b                         | end
```

xxd 命令同上

```shell
00000000: 00 61 73 6D 01 00 00 00 01 06 01 60 02 7F 7F 00  .asm.......`....
00000010: 03 02 01 00 06 0B 02 7F 01 41 0F 0B 7F 01 41 14  .........A....A.
00000020: 0B 0A 0C 01 0A 00 23 00 24 01 20 00 21 01 0B     ......#.$. .!..
```

#### 内存指令

内存指令共25条，其中14条用于将内存数据加载到操作数栈，9条存储指令，将操作数栈顶的数据写回内存，加载，存储的这些指令带有两个操作数：对齐提示、内存偏移量。

剩余的指令用于获取和扩展内存页数，操作数为内存索引，由于wasm规范规定，模块只能导入和定义一块内存，因此内存索引，目前只起到占位的作用，index ==0。

> tip：内存可以在限制范围内动态增长，增长必须以页为单位，一页是 **64KB** 即 65536 字节。内存页数总数不能超过65536，也就是一个wasm的内存不能超过**4GB**

例：

```shell
   (memory 1 8)
   ;; 将memory 设置初始page = 1  max = 8   此处可以先将max 设置为65537 观察终端中的错误信息。
   ;; error: max pages (65537) must be <= (65536) 
   ;; (memory 1 65537) 
   
   
   (data (offset (i32.const 100)) "hello")
   (func

       (i32.const 1) (i32.const 2)
       (i32.load offset=100)
       (i32.store offset=100)
       (memory.size) (drop)
       (i32.const 4 ) (memory.grow) (drop)
   )
```

编译之后使用wasm-odjdump 命令查看二进制文件

```shell
wasm-objdump -d name.wasm


Code Disassembly:

00001c func[0]:
00001d: 41 01                      | i32.const 1
00001f: 41 02                      | i32.const 2
000021: 28 02 64                   | i32.load 2 100
000024: 36 02 64                   | i32.store 2 100
000027: 3f 00                      | memory.size 0
000029: 1a                         | drop
00002a: 41 04                      | i32.const 4
00002c: 40 00                      | memory.grow 0
00002e: 1a                         | drop
00002f: 0b                         | end
```

#### 结构化控制指令

结构控制指令共13条，包括结构化控制指令、跳转指令、函数调用指令等。

结构化控制指令：

- block  opcode 0x02
- loop   opcode 0x03
- if     opcode 0x04

以上的三个指令必须搭配 `end (0x0B)`指令。 若if 语句存在分支，则中间有else (0x05)指令分隔。

例：

```shell
(module

(func (result i32)
       (block (result i32)
        (i32.const 1)
        (loop (result i32)
        (if (result i32)(i32.const 2 )
           (then (i32.const 3 ))
           (else (i32.const 4))
        )

       )
       (drop)
       )

   )
  )
```

编译之后，使用wasm-objdump 对二进制文件进行解析

```shell
000017 func[0]:
 000018: 02 7f                      | block i32
 00001a: 41 01                      |   i32.const 1
 00001c: 03 7f                      |   loop i32
 00001e: 41 02                      |     i32.const 2
 000020: 04 7f                      |     if i32
 000022: 41 03                      |       i32.const 3
 000024: 05                         |     else
 000025: 41 04                      |       i32.const 4
 000027: 0b                         |     end
 000028: 0b                         |   end
 000029: 1a                         |   drop
 00002a: 0b                         | end
 00002b: 0b                         | end
```

xxd 检查二进制代码，就能找到起始和终止的命令

```shell
xxd -u -g 1 name.wasm

00000000: 00 61 73 6D 01 00 00 00 01 05 01 60 00 01 7F 03  .asm.......`....
00000010: 02 01 00 0A 17 01 15 00 02 7F 41 01 03 7F 41 02  ..........A...A.
00000020: 04 7F 41 03 05 41 04 0B 0B 1A 0B 0B              ..A..A......
```

#### 跳转指令

跳转指令一共4条

- br opcode 0x0c 进行无条件跳转 立即数是目标的标签索引
- br_if  opcode 0x0d 进行有条件的跳转，立即数也是目标标签索引
- br_table opcode 0x0e 进行查表跳转，立即数是目标标签索引和默认标签索引
- return opcode 0x0f 直接跳出到最外层的函数，导致整个函数返回。

例：

```shell
  (module
  
  (func
       (block (block (block
           (br 1)
           (br_if 2 (i32.const 100) )
           (br_table 0 1 2 3 )
           (return)
       )))

   )
   )
```

编译后，使用wasm-objdump 进行检查

```shell
000016 func[0]:
 000017: 02 40                      | block
 000019: 02 40                      |   block
 00001b: 02 40                      |     block
 00001d: 0c 01                      |       br 1
 00001f: 41 e4 00                   |       i32.const 100
 000022: 0d 02                      |       br_if 2
 000024: 0e 03 00 01 02 03          |       br_table 0 1 2 3
 00002a: 0f                         |       return
 00002b: 0b                         |     end
 00002c: 0b                         |   end
 00002d: 0b                         | end
 00002e: 0b                         | end
```



#### 函数调用指令

wasm支持直接调用和间接调用两种方式

- call opcode 0x10 根据立即数指定的函数索引来调用函数
- call_indirect 0x11 函数的签名索引由立即数指定，但是直到运行时才知道调用的函数。

例：

```shell
(module

	(type $ft1 (func))
    (type $ft2 (func))
    (table funcref (elem $f1 $f1 $f1))
    (func $f1
        (call $f1)
        (call_indirect (type $ft2) (i32.const 2))
    )


)
```

编译后，使用wasm-objdump 进行检查

```shell
00002b func[0]:
 00002c: 10 00                      | call 0
 00002e: 41 02                      | i32.const 2
 000030: 11 01 00                   | call_indirect 1 0
 000033: 0b                         | end
```



### 文本格式

wasm 文本格式采用S- 表达式描述模块（Lisp）

和二进制格式的差异

- 二进制格式是以段为单位组织数据，文本格式则是以域为单位，wat编译器会把相同类型的域收集起来，合并成二进制段
- 二进制格式中，除了自定义的段，其他段的ID必须是递增的顺序排序，文本格式中要求并没有那么严格，但是导入域必须在函数域、表域、内存域和全局域之前。
- 文本格式的域和二进制文件中的段基本上是一一对应的。但有两种特例：
  - 文本格式没有单独的代码域，只有函数域，wat编译器会把函数收集起来，分别生成代码段和函数段
  - 文本格式没有自定义域
- 文本格式提供了多种内联的写法

#### 类型域

```shell
(module
	(type $ft1 (func (param i32 i32) (result i32)))
)
```

定义了一个函数 入参两个i32类型，输出i32结果

#### 导入导出域

外部导入

```shell
(module
   (import "evn" "f1" (func $f1 (type $ft1)))

   (import "env" "f2" (func $f2 (param f64) (result f64 f64)))  ;; inline func type 
    
   (import "env" "t1" (table $t 1 8 funcref))

   (import "env" "m1" (memory $m 4 16))

   (import "evn" "g1" (global $g1 i32))   ;; immutable

   (import "evn" "g2" (global $g2 (mut i32))) ;; mutable

)
```

导出域

```shell
(module  
    (export "f1" (func $f1))

    (export "m1" (memory $m1))
    
    (export "g1" (global $g1))
    
)
```

