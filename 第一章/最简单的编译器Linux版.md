# 最简单的编译器

上一节我们实现了最简单的解释器`interp`，本节我们实现`build`和`run`命令，把源码编译成可执行文件。


## 编译流程

编译器比解释器要复杂的多，在把源码解析成AST语法树之后，还要做几个步骤：

- 遍历AST，进行类型检查
- 将AST转换成中间代码（IR）
- 对IR进行多轮优化
- 将IR转换成目标代码
- 链接目标代码，生成可执行文件

有的编译器还不止一种IR，而是先转化为一种IR，进行某些检查和优化操作，再转化为另一种形式的IR，以便于进行更深入地优化，最终再转换成目标代码。

TODO: 画一个流程图

Z语言是一个循序渐进的语言，因此我们先忽略掉所有“锦上添花”的步骤，而只实现最简单的流程：

- 将AST转化为目标代码
- 调用外部工具进行链接并生成可执行文件

由于现在不需要考虑优化和跨平台，我们可以直接砍掉IR，未来再考虑加上。
我们选择一种比较常见的目标代码：GNU x86-64汇编，并借助gcc来进行链接和可执行文件的生成。

未来如果要扩充成严肃的编译器，可以有以下几个路径：

- 自研一套IR，自己写针对不同平台的汇编器和链接器。Go语言选择的是这个道路。
- 借助LLVM，这样就只需要把Z语言的AST翻译成LLVM IR，即可实现所有的优化操作和跨平台支持。Rust语言选择的是这个道路。
- 选择WASM作为目标代码，WASM有机会成为“真正的跨平台汇编”。现在所有的语言几乎都增加了wasm的选项。
- 不考虑代码生成，而以C作为目标代码，再借助gcc/clang等编译器实现所有平台的优化编译。V语言选择的是这个道路。

从经济性考虑的话，第二套方案显然更合理。第三套也可行，只是暂时还不太成熟。第四套本来就是Z语言三大转译目标之一，在后面会详细介绍。
但是从“学习”和“探索”的原则出发的话，显然第一套方案更有意义。因为我可以在自研IR和支持不同平台汇编的过程中学到更多东西。

所以我选择全部都要，或者至少保留所有方案的接口，这样以后大家有感兴趣的，可以来实现。

本章我们选择GNU x86-64汇编为目标代码，从zast直接生成它，再借用gcc命令来执行链接和生成可执行文件。
这样做有个缺点，就是只能Linux平台下跑。但作为第一次的尝试，并不是大问题。将来扩展编译器功能时，我会加上Windows的MASM64的支持；其他平台，例如移动端的ARM，新兴的RISC-V，国产的LoongArch，都可以加上。

#### 为什么不做类型检查？

最基本的类型检查可以在AST解析阶段就做好，而复杂一些的检查，往往是为了应付复杂的类型系统而产生的。现在的Z语言及其简单，可以说没有类型系统，所以这个问题暂时不用考虑。

我打算在添加了整数、浮点数和布尔类型之后，再做基本的类型检查和最简单的类型推导。

#### 为什么不做优化？

优化问题很重要，但是我认为它是纯纯的锦上添花的功能。过早地优化，会让编译流程变得非常复杂，并不适合Z语言循序渐进的“学习”和“探索”风格。
我打算把所有的优化工作都放在功能实现完成之后，即先完善整个语言的所有特性，再来考虑优化问题。

#### 为什么不做IR？

IR本身是为了跨平台和优化而生的。Z语言现在不需要考虑优化，平台也暂时只支持Linux和Windows。

另一方面，IR的设计也是个复杂的问题，我打算深入学习了LLVM IR、MIR、Go IR和WASM之后，再来考虑自研IR的问题。

所以，IR的问题也要放在语言特性完整之后，在添加LLVM或WASM的后端时再研究了。

#### 为什么不自己实现链接器？

链接器是一个有趣的问题，我以后会考虑学习并实现一套实验性质的链接器。
但链接器这个问题比较独立，和编译流程没有多少交集，且涉及非常多操作系统的细节，我把它的优先度放在了优化问题之后。

## GNU x86-64汇编

汇编是很复杂的体系，即使把目光聚焦到只看Linux下的GNU X86汇编，仍然要做出选择：

- 语法风格：AT&T vs Intel
- 寄存器表示：prefix vs noprefix
- 指令集：x86 vs x86-64
- 工具链：gcc/gas vs nasm vs clang
- 操作系统：Windows vs Linux
- 是否引用C标准库：不引用 vs 引用

这几个指标的每个选择，都会产生某些区别。

我根据自己的喜好，每项都选择了最后一个：Intel语法、noprefix、x86-64、clang、Linux、引用C标准库。

至于为什么要用Linux汇编而不是Windows汇编，是因为我之前在做Z语言前期探索时学的是Linux汇编，并已经实现过一个简单版本了，所以今天就先用这个方案了。
等下一步有空再学习Windows下的汇编`masm64`和相应的工具，然后再来实现Windows版本的编译器。

考虑到我的开发环境是Win11+WSL/Ubuntu，支持这两种平台的汇编已经足够了。

这样选好之后，先尝试手写一个Hello World的汇编代码，大概是这个样子：

```asm
  .intel_syntax noprefix
  .text
  .global main

main:
  push rbp
  mov rbp, rsp

  lea rdi, [rip+msg]
  call puts

  pop rbp
  xor eax, eax
  ret

msg:
  .asciz "Hello, world!"
```

这段代码存放到`hello.s`文件。

这段代码的含义是：

- `.intel_syntax noprefix`：配置代码格式：使用Intel语法，且不需要加前缀。
- `.text`：代码段，后面的内容会存放在可执行文件的代码段中。
- `.global main`：声明`main`函数是全局函数。这样其他函数就可以调用它了。实际上可执行文件的入口函数是`_start`，gcc/clang会自动生成`_start`，初始化好环境（主要是C标准库），然后调用`main`函数。
- `push rbp; mov rbp, rsp;`：在汇编里，任何函数调用开始时都需要保存现场，即把当前的`rbp`（base pointer）压栈，以便之后恢复；再把盏指针的值赋给`rbp`，这样函数内部的代码就可以访问栈上的局部变量了。
- `lea rdi, [rip+msg]; call puts`：先把`msg`的地址放到`rdi`寄存器里，再调用`puts`函数。在x86_64汇编中，函数的第一个参数从`rdi`寄存器找，第二个参数从`rsi`寄存器找，以此类推。具体可以参考[这篇文章](http://6.s081.scripts.mit.edu/sp18/x86-64-architecture-guide.html)。我会在将来实现多参数函数时详细介绍。
- `pop rbp`：函数调用结束了，恢复之前的`rbp`。
- `xor eax, eax`：把`eax`寄存器清零，作为函数的返回值。
- `ret`：函数返回。返回值即`eax`寄存器中的值，现在为0。
- `msg: .asciz "Hello, world!"`：定义一个常量，内容是`Hello, world!`。`.asciz`表示这是一个以0结尾的字符串。

它对应的C程序是这样的：

```c
#include <stdio.h>

int main(void) {
    puts("Hello, world!");
    return 0;
}
```

实际上，如果想知道一段C程序生成的汇编是什么样子，有个非常好用的在线工具[godbolt compiler explorer](https://godbolt.org/)，可以直接输入C程序，选择编译工具链，查看生成的汇编代码。
有兴趣做编译器开发的同学一定要研究一下这个网站。非常赞！

要在Linux系统中编译上面的`hello.s`，可以用`clang`或`gcc`：

```bash
$ clang -o hello.exe hello.s
$ ./hello.exe
Hello, world!
```

clang（或gcc）会自动调用汇编器（llvm-as或gas），将`hello.s`文件转换为二进制格式（相当于.o文件），
然后调用系统提供的链接器（ld），将这个二进制文件和C标准库的库文件链接起来，生成可执行文件`hello.exe`。

实际上，clang/gcc在编译C程序时，只是比编译汇编文件多一步而已：先把C代码编译为汇编代码，后面就一样了。

所以我们的Z编译器，做的其实是clang/gcc中将C语言编译为汇编代码的这个模块相同的工作。

下一步，我们想办法把Z语言的源码转换为上面的这个汇编。

我们要用的Z代码和上一节相同：

```z
print("Hello, world!")
```

## 从Z到汇编

和解释器一样，从Z语言编译为汇编的过程中，先要把Z语言代码解析成zast语法树。
这个工作我们已经在上一节的`interp`解释器中做过了，也就是`parse_expr()`函数。
这个函数的输出是zast语法树，具体而言是一个`CallExpr`节点，因为现在整个语言只有这一种类型的AST节点。
所以本节的编译器，任务是把一个`CallExpr`节点转化为对应的`.s`汇编文件

我们需要需要一个`codegen`函数：

```c
// 将AST编译成汇编代码
static void codegen(CallExpr *expr) {
    // 打开输出文件
    FILE *fp = fopen("app.s", "w");
    // 首行配置
    fprintf(fp, "    .intel_syntax noprefix\n");
    fprintf(fp, "    .text\n");
    fprintf(fp, ".global main\n");
    fprintf(fp, "main:\n");
    // prolog
    fprintf(fp, "    push rbp\n");
    fprintf(fp, "    mov rbp, rsp\n");
    // 调用puts函数。注：这里还没有做好内置函数print和C标准库函数puts的映射，所以先直接用puts。未来会在内置函数功能中添加这种映射。
    fprintf(fp, "    lea rdi, [rip+msg]\n");
    fprintf(fp, "    call %s\n", "puts");
    // epilog
    fprintf(fp, "    pop rbp\n");
    // 返回
    fprintf(fp, "    xor rax, rax\n");
    fprintf(fp, "    ret\n");
    // 设置参数字符串
    fprintf(fp, "msg:\n");
    fprintf(fp, "    .asciz \"%s\"\n", expr->arg);

    // 保存并关闭文件
    fclose(fp);
}
```

可以看到，这个函数生成的内容和上一节手写的hello.s应当是一致的，唯一的不同，是如果Z源码里的字符串传入其他内容时，生成的`.s`文件里的msg字段内容也会跟着变化。

我们在`build`函数里调用`codegen`：

```c
static void build(char *file) {
    printf("Building %s\n", file);
    // 读取源码文件内容
    char *code = read_src(file);
    // 解析出AST
    CallExpr *expr = parse_expr(code);
    // 输出汇编代码
    codegen(expr);
    // 调用clang生成可执行文件
    system("clang -o app.exe app.s");
}
```

先调用`read_src`函数读取源码，再调用之前写好的`parse_expr`解析出zast，接着调用`codegen`生成`.s`文件，最后调用`clang`生成可执行文件。

由于现在用的GNU x86_64汇编只在Linux系统中支持，所以这个编译器暂时只能在Linux环境下跑起来。

我们在Win11中安装好`wsl2`，进入Ubuntu系统，安装`clang`，再用`git`下载好最新的Z编译器代码，就可以尝试编译运行了：

我们在编译器目录下新建一个`work`目录，用做编译器的运行目录，并把`hello.z`文件放到这个目录下。

```bash
$ xmake build
[ 25%]: cache compiling.release src/main.c
[ 50%]: linking.release z
[100%]: build ok, spent 0.361s
$ xmake run -w work z build hello.z
Hello from Z!
Building hello.z
Parsing print("Hello, world!")...
CallExpr {
  fn: print
  arg: "Hello, world!"
}
```

注意，`xmake run -w work z`的意思就是把`work`目录当做工作目录来执行`z`程序。

此时`clang`生成了可执行文件`app.exe`，我们可以直接运行：

```bash
$ cd work
$ ./app.exe
Hello, world!
```

成功了！

现在我们就可以提交代码并打上第4个标签了：

```bash
$ git commit -a -m "step4: 最简单的编译器Linux版"
$ git tag v0.0.4 -m "step4: 最简单的编译器Linux版"
$ git push
```

## 小结

本节实现了Linux下的最简单的GNU x86_64汇编的编译功能，并且调用`clang`生成了可执行文件。

我已经可以宣布，做出了最简单的编译器了！

但是，我的主开发环境是Windows，所以下一节还要把这个编译器移植到Windows上来。也就是说，我需要去学习Windows下的汇编语言`masm64`，以及配套的编译工具。
