# 最简单的编译器 - Windows版

上一节我实现了一个最简单的编译器，可以编译并执行`print("Hello world")`这样的Z语言程序。
但我的主力开发环境是Windows，所以还是需要花功夫实现Windows版的Z编译器。

这样做的缺点是未来任何新功能都要多实现一套编译codegen，测试工作也增加了；
但优点是方便了开发流程，我可以先开发Windows版，再移植到Linux版，去WSL内测试。

另一个好处，就是Z语言编译器从最开始就支持了跨平台，这是一个很好的开端。
从一开始就支持双平台的话，日常开发就必须考虑好跨平台的问题，
所以未来要再添加新的平台支持（例如龙芯或arm）的话，会容易得多。

## Windows下的汇编

Windows下的汇编器是`masm64`（Microsoft Assembly 64bit），它的语法和Linux下的`gas`有很大不同。
不过好在我用的`gas`语法选择了`intel_syntax`，所以具体到指令层面还是很像的。

Windows下的汇编我从来没接触过；实际上，windows开发我从来都没有接触过。所以找了不少资料才搞清楚基本的逻辑。

除了一些网页资料（我会记录在《脚踏实地汇编语言》一书里），我发现最有用的一本书就是《The Art of 64-Bit Assembly Language》。
这本书是经典的《The Art of Assembly Language》的64bit版本，作者是`Randall Hyde`。

经过两天的查找资料和学习，我大致了解了`masm64`的结构，以及与`gas64`的异同。

下面是我试探出来的Hello World汇编，它调用了`C`语言的`printf`函数，输出`Hello world`。

```asm
; Hello World for Windows x64 Assembly

; 导入C标准库。 msvcrt的意思是MicroSoft Visual C Runtime
includelib msvcrt.lib
; 新版的Visual Studio把printf系列函数嵌入到头文件里直接用C实现了，所以汇编语言没法从标准库里直接调用了，需要导入legacy_stdio_definitions.lib
includelib legacy_stdio_definitions.lib

; 数据段
.data
    msg db 'Hello, world!', 10, 0

; 代码段
.code
    ; 声明printf函数，它是由外部的库提供的。printf的实现实际上放在legacy_stdio_definitions.lib里
    externdef printf:proc

; 主函数
main proc
    ; Prologue，函数被调用时需要调整栈指针
    push rbp
    mov rbp, rsp
    ; 预留至少32字节的栈空间，作为Shadow Space
    sub rsp, 20h

    ; 把msg的地址传给rcx寄存器，作为printf的第一个参数
    lea rcx, msg ; lea: load the address of a variable into a register
    ; 调用printf
    call printf
    
    ; 归还Shadow Space
    add rsp, 20h
    ; Epilogue，恢复栈指针
    pop rbp

    ; 返回0
    mov rcx, 0
    ret

main endp

end
```

要运行这段程序，首先需要安装`Visual Studio`，并勾选上`C/C++开发`的模块，这样就会自动装上我们所需要的汇编器`ml64.exe`。

但是要运行这个汇编器，需要在命令行导入相应的基础库，而Visual Studio为我们提供了自动导入的工具：“Developer Command Prompt for VS 2022”。

在Windows的开始菜单里直接搜索`Develop`，一般就能直接找到这个工具。打开它，和普通的`cmd`命令行差不多，只不过多引入相关的环境变量。

在这个命令行中找到`hello.asm`所在的目录，然后执行：

```bash
> ml64 hello.asm
Microsoft (R) Macro Assembler (x64) Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

 Assembling: hello.asm
Microsoft (R) Incremental Linker Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/OUT:hello.exe
hello.obj
```

可以看到，编译器对hello.asm进行了汇编，得到两个文件：`hello.obj`和`hello.exe`，这时候我们只要直接运行`hello`，就可以看到结果：

```bash
> hello
Hello, world!
```

Windows下的`hello.asm`看起来要比Linux下的`hello.s`复杂，这其实是Windows开发环境的特点导致的。

Windows的实现语言主要是C++，汇编在操作系统中的地位主要是嵌入到C++代码中进行内敛汇编，以提高某些局部函数的性能，或者作为一个独立的模块，提供几个高效的函数供C++调用。

正是因为这样才导致新版的标准库里汇编都没法直接调用`printf`函数了，还得调用老版的库。

不过我们可以看出来，在`main`函数内部的代码，和Linux汇编还是比较近似的，都有`Prologue`和`Epilogue`，函数调用的方法也很类似，唯一的区别是调用时采用的寄存器不同。

## Hello World的win32版

在查找如何写出Hello World的过程中，我发现除了在命令行中调用`printf`之外，其实直接借助`win32`的系统调用，通过GUI弹窗的方式来输出`Hello World`也同样容易。

而在我以往的印象中，GUI库用起来肯定比命令行程序要复杂。果然Windows系统本身就更倾向于图形界面。

下面是一个调用`MessageBox`弹出消息框来展示`Hello World`的汇编实例：

```asm
; ASM Hello World Win64 MessageBox
; 编译命令: ml64.exe world.asm /link /subsystem:windows /entry:main

; 导入win32的标准库
includelib kernel32.lib
includelib user32.lib

; 声明MessageBox和ExitProcess这两个库函数
externdef MessageBoxA: proc 
externdef ExitProcess: proc

; 数据段
.data
  ttl db 'Win64 Hello', 0   ; 弹框标题
  msg db 'Hello World!', 0  ; 消息内容

; 代码段
.code
; 主函数
main proc
  ; 预留栈空间
  sub rsp, 28h     ; for Shadow Space

  ; 为MessageBoxA函数准备参数
  mov rcx, 0       ; hWnd = HWND_DESKTOP
  lea rdx, msg     ; LPCSTR lpText
  lea r8,  ttl     ; LPCSTR lpCaption
  mov r9d, 0       ; uType = MB_OK
  ; 调用弹窗
  call MessageBoxA

  ; 恢复栈指针
  add rsp, 28h  

  ; 获得MessageBoxA的返回值
  mov ecx, eax     ; uExitCode = MessageBox(...)
  ; 退出程序
  call ExitProcess
main endp

End
```

注意，和命令行的情况不同，这段代码在编译时制定了入口函数：`/entry:main`；
而上一个例子中程序的实际入口是`/entry:mainCRTStartup`，在`mainCRTStartup`中进行了C语言运行时`CRT`的初始化工作后再调用`main`函数的。
因此`main`函数中需要做`Prologue`和`Epilogue`的工作。
而本节实例的`main`函数本身就是入口函数，就不需要了。

从这一点来说，本节的例子其实比上一节流程上跟简单：准备一下参数，直接触发系统调用，就输出消息了。

把代码保存到文件`world.asm`，编译并运行：

```bat
> ml64 world.asm /link /subsystem:windows /entry:main
Microsoft (R) Macro Assembler (x64) Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

 Assembling: world.asm
Microsoft (R) Incremental Linker Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/OUT:world.exe
world.obj
/subsystem:windows
/entry:main
> world
```

这时会看到一个弹窗，内容是`Hello World`，如图所示：

![hello win32](pics/hello_win32.png)

当然，Z语言暂时并不会涉及到基于win32的GUI编程，所以这个例子只是为了展示一下Windows下汇编和Linux下汇编的异同。

接下来我们仍然回到命令行程序的实现。

## 从Z语言到masm64汇编

有了上一章中的`codegen`，这一步就很容易了，依葫芦画瓢即可。

我们需要把上一章实现的`codegen`改名为`codegen_linux`，而本章的汇编函数就叫`codegen_win`。


```c

// 将AST编译成汇编代码：windows/masm64
static void codegen_win(CallExpr *expr) {
    // 打开输出文件
    FILE *fp = fopen("app.asm", "w");
    // 导入标准库
    fprintf(fp, "includelib msvcrt.lib\n");
    fprintf(fp, "includelib legacy_stdio_definitions.lib\n");
    // 要打印的信息参数
    fprintf(fp, ".data\n");
    fprintf(fp, "    msg db '%s', 10, 0\n", expr->arg);
    fprintf(fp, ".code\n");
    // 声明printf函数
    fprintf(fp, "    externdef printf:proc\n");

    // main函数

    fprintf(fp, "main proc\n");
    // prolog
    fprintf(fp, "    push rbp\n");
    fprintf(fp, "    mov rbp, rsp\n");
    // reserve stack for shadow space
    fprintf(fp, "    sub rsp, 20h\n");

    // 准备printf参数
    fprintf(fp, "    lea rcx, msg\n");
    fprintf(fp, "    call printf\n");

    // restore stack
    fprintf(fp, "    add rsp, 20h\n");
    // epilog
    fprintf(fp, "    pop rbp\n");
    // 返回
    fprintf(fp, "    mov rcx, 0\n");
    fprintf(fp, "    ret\n");

    // 结束
    fprintf(fp, "main endp\n");
    fprintf(fp, "end\n");

    // 保存并关闭文件
    fclose(fp);
}
```

这里和`codegen_linux`类似，也是一行行输出hello.asm的示例代码，只是在需要填入`msg`内容的地方填入编译器从`hello.z`里获得的字符串。

现在`build`函数需要根据操作系统来判断是调用`codegen_linux`还是`codegen_win`：

```c
static void build(char *file) {
    printf("Building %s\n", file);
    // 读取源码文件内容
    char *code = read_src(file);
    // 解析出AST
    CallExpr *expr = parse_expr(code);
    // 输出汇编代码
#ifdef _WIN32
    codegen_win(expr);
#else
    codegen_linux(expr);
#endif
}
```

这样就OK了。

编译`zc`项目：

```bash
$ xmake build
[ 25%]: compiling.release src\main.c
[ 50%]: linking.release z.exe
[100%]: build ok, spent 0.562s
```

运行`z build`：

```bash
$ xmake run -w work z build hello.z
Hello from Z!
Building hello.z
Parsing print("Hello, world!")...
CallExpr {
  fn: print
  arg: "Hello, world!"
}
```

这时候会看到`work`目录里已经生成了一个`app.asm`汇编文件，内容也和我们手写的`hello.asm`一样。

调用`ml64`进行编译：

```bash
$ cd work
$ ml64 app.asm
Microsoft (R) Macro Assembler (x64) Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

 Assembling: app.asm
Microsoft (R) Incremental Linker Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/OUT:app.exe
app.obj
``````

得到了`app.exe`，运行它：

```bash
$ app
Hello, world!
```

成功了！

如果切回到`WSL`的`Ubuntu`，再执行类似的编译和运行，也能得到同样的结果。

不过考虑到Linux底下要调用`clang`，Windows要调用`ml64`，而`ml64`必须在微软提供的`Development Console`里进行，
所以我就把这个流程独立出来了。现在再Linux下，`z build hello.z`也只会生成`app.s`，后面需要读者自己调用`clang`来编译他：

```bash
$ cd work
$ clang -o app.exe app.s
$ ./app.exe
Hello, world!
```

## 小结

现在我们已经拥有了一个最简单的，但是跨平台的编译器了！这也算是一个小成就了。

再看一看`build`函数：

```c
static void build(char *file) {
    printf("Building %s\n", file);
    // 读取源码文件内容
    char *code = read_src(file);
    // 解析出AST
    CallExpr *expr = parse_expr(code);
    // 输出汇编代码
#ifdef _WIN32
    codegen_win(expr);
#else
    codegen_linux(expr);
#endif
}
```

多么简洁，多么美妙！

现在我们又可以提交一个新的标签了：

```bash
$ git commit -a -m "步骤5：Windows下的最简编译器"
$ git tag v0.0.5 -m "步骤5：Windows下的最简编译器"
$ git push
```

至此，解释器和编译器都做好了。

接下来的工作是转译器：z2c、z2py、z2js。

虽然有三份工作，不过考虑到我们只需要实现最简单的`print`函数，而且这仨都是高级语言，理论上比生成汇编要更容易，所以不用担心。

做完这三个转译器之后，我们再把前面所有的工作整合到一起，做出一个可以互动的REPL来。

