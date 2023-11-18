# 编译器与Z标准库

本节的目标是实现最简单的文件读写。

我们需要实现的功能函数分别是：

```z
// 读取并打印文件内容
fn read_file(path str)

// 写入文件
fn write_file(path str, content str)
```

由于Z语言现在还没有基本的逻辑操作语句，也没有局部存储的变量，
暂时没法用Z语言自己来实现读写的逻辑。

我选择的办法是先用C语言来实现这两个函数，Z语言只需要直接调用它就好，
就和调用`print`一样。

在这个过程中，我们实际上也会搭建出最简单的Z语言标准库。

这两个函数的C实现，未来会转换成用Z来实现。

## 标准库框架

我们先把C版本的标准库架子搭起来。

在工程的根目录下建立一个`lib`子目录，作为标准库的根目录。

我们先把标准库都放到一个叫`std.h`/`std.c`的文件里。

以后如果要分模块，再改成`std`目录，加上内部的各模块文件。

`std.h`的内容如下：

```c
// 读取文件
void read_file(const char *path);

// 写入文件
void write_file(const char *path, const char *content);
```

他们的实现放在`std.c`里：

```c
// 读取文件
void read_file(const char *path) {
    FILE *fp = fopen(path, "r");
    if (fp == NULL) {
        printf("Error: cannot open file %s\n", path);
        exit(1);
    }
    char *line = NULL;
    size_t len = 0;
    while (getline(&line, &len, fp) != -1) {
        printf("%s", line);
    }
    fclose(fp);
}

// 写入文件
void write_file(const char *path, const char *content) {
    FILE *fp = fopen(path, "w");
    if (fp == NULL) {
        printf("Error: cannot open file %s\n", path);
        exit(1);
    }
    fprintf(fp, "%s", content);
    fclose(fp);
}
```

接着给这个`std.c`添加一个测试用例：

```c
// test/test_std.c
#include "std.h"

int main(int argc, char **argv) {
    write_file("write_file_test.txt", "hello world\n");
    read_file("write_file_test.txt");
    return 0;
}
```

在`xmake.lua`中添加`std`库的编译目标：

```lua
target("std")
    set_kind("static")
    add_files("lib/*.c")
    add_files("src/util.c")
    add_includedirs("lib")
    add_includedirs("src")
```

注意，这里的编译类型是`static`，也就是静态链接库。

接着在添加`std`库的测试用例：

```lua
target("test_std")
    set_kind("binary")
    add_files("test/test_std.c")
    add_deps("std")
    add_includedirs("lib")
    add_includedirs("src")
    set_rundir(os.projectdir())
    add_tests("hello", {rundir = os.projectdir(), trim_output=true, pass_outputs="5\nhello\nhello world"})
    after_test(function (target, opt)
        os.rm("write_file_test.txt")
    end)
```

注意，我们测试的方式是先调用`write_file`写一个文件，再调用`read_file`读取并打印它的内容，
最后再和测试用例的`pass_outputs`进行对比。

这个过程中会生成一个临时文件`write_file_test.txt`，
所以需要在`xmake.lua`中添加一个`after_test`回调，
在完成测试后删掉它。

未来如果标准库里添加了`remove_file`函数，就可以用它来代替。

执行测试：

```bash
$ xmake test test_std/*
running tests ...
[100%]: test_std/hello .................................... passed 0.031s

100% tests passed, 0 tests failed out of 1, spent 0.031s
```

通过了！

添加一个Git标签：

```bash
$ git commit -a -m "步骤17：最基本的C版标准库"
$ git tag -a v0.0.17 -m "步骤17：最基本的C版标准库"
$ git push
```

接下来考虑如何用汇编调用这两个标准库函数。

## 汇编调用C函数

在汇编里调用自己写的C函数，和调用C标准库的函数差不多。

唯一的不同是需要在链接时加上`std.lib`。

我们先试试调用`read_file`函数，只要在以往调用`printf`的基础上进行修改就行了：

```asm
includelib msvcrt.lib
includelib legacy_stdio_definitions.lib
; 添加std.lib
includelib std.lib

.data
    ; 文件名称；注意这里是文件名，所以字符串后面不要加'\n'，即`,10`。
    fil db 'hello.z', 0

.code
    ; 声明外部函数
    externdef read_file:proc

main proc
    push rbp
    mov rbp, rsp
    sub rsp, 20h

    ; 填入第一个参数
    lea rcx, fil
    ; 调用函数
    call read_file
    
    add rsp, 20h
    pop rbp

    mov rcx, 0
    ret

main endp

end
```

把这段代码存入到到`work/read.asm`文件中，编译并运行：

```bash
$ ml64 read.asm /link ..\build\windows\x64\debug\std.lib
Microsoft (R) Macro Assembler (x64) Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

 Assembling: read.asm
Microsoft (R) Incremental Linker Version 14.37.32825.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/OUT:read.exe
read.obj
..\build\windows\x64\debug\std.lib
LINK : warning LNK4098: 默认库“LIBCMT”与其他库的使用冲突；请使用 /NODEFAULTLIB:library
```

这里的warning可能是由于我们使用的`std.lib`和`msvcrt.lib`都包含了`printf`函数导致的，
暂时不管它。

运行：

```bash
$ cd work
$ .\read.exe 
print("Hello, world!")
```

可以看到，正确读取了`hello.z`文件的内容。

如果要调用`write_file`，就需要准备两个参数了：

```asm
includelib msvcrt.lib
includelib legacy_stdio_definitions.lib
; 添加std.lib
includelib std.lib

.data
    ; 要打印的消息，后面的`10`代表换行符
    msg db 'Hello, world!', 10, 0
    ; 文件名称；注意这里是文件名，所以字符串后面不要加'\n'
    fil db 'hello.tmp', 0

.code
    ; 声明外部函数
    externdef write_file:proc

main proc
    push rbp
    mov rbp, rsp
    sub rsp, 20h

    ; 填入第一个参数
    lea rcx, fil
    ; 填入第二个参数
    lea rdx, msg
    ; 调用函数
    call write_file
    
    add rsp, 20h
    pop rbp

    mov rcx, 0
    ret

main endp

end
```

注意，这里第一个参数是`fil`，存入`rcx`；第二个参数是`msg`，存入`rdx`寄存器。

Windows的函数调用惯例是前四个参数用`rcx`、`rdx`、`r8`、`r9`传递，如果有更多的参数就需要存放到栈上了。具体参看[Windows Calling Convention](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170#parameter-passing)。

而如果参数是浮点数，那么前四个参数就换成`xmm0`、`xmm1`、`xmm2`、`xmm3`这4个浮点数专属的寄存器了。

未来我们实现浮点数的时候需要单独处理这种情况。

本章我们只实现简单的函数调用，即不超过4个参数的情况。

综合上面两个示例，我们可以修改`codegen.c`，给它添加处理标准库函数的能力。

不过在这之前，我们还要看一看Linux下的汇编是不是类似的情况。

查找一番资料后，我发现Linux下的汇编和windows基本相同，区别在于：

- 不需要`includelib`，也不需要`externdef`，直接假装函数已经存在就行，直接调用。
- 调用惯例不一样。Linux下，前6个参数用寄存器传递，之后用栈。

注意：Linux的6个参数传递寄存器分别是：`rdi`、`rsi`、`rdx`、`rcx`、`r8`、`r9`。
这一点和Windows不同。

为了减少代码复杂度，我们取最简单的情况，即4个参数作为当前Z语言函数参数个数的上限。

下面是Linux下写入并读取一个文件的示例：

```asm
    # config for intel syntax and no prefix
    .intel_syntax noprefix
    # text section stores code
    .text
    # export main function to global space
    .global main

# definition of main function
main:
    # prologue
    push rbp
    mov rbp, rsp
    # load address of fil into rdi
    lea rdi, [rip+fil]
    # load address of msg into rsi
    lea rsi, [rip+msg]
    # call C function `write_file`
    call write_file
    # load address of fil into rdi
    lea rdi, [rip+fil]
    # call C function `read_file`
    call read_file
    # epilogue
    pop rbp
    # return 0
    xor eax, eax
    ret


fil:
    .asciz "hello.tmp"
msg:
    .asciz "Hello, world!\n"
```

注意，这里在调用完`call write_file`后，`rdi`和`rsi`寄存器的值会消失，因为他们默认是`volatile`的，因此在调用`call read_file`之前，需要再次把`fil`的地址填入到`rdi`寄存器中。否则会报`Segmentation fault`错误。

要运行它，也需要把我们写的`std`标准库引入。在Linux下，生成的标准库是`libstd.a`：

```bash
$ cd work
$ clang -o read_write.exe read_write.s ../build/linux/x86_64/release/libstd.a
$ ./read_write.exe
Hello, world!
```

我们查看目录，发现确实生成了临时文件`hello.tmp`。

有了这个示例代码，在编写`codegen.c`时就可以对照参考了。

## 支持多个参数

要支持`read_file`和`write_file`，我们的语法树也要做出修改了。

这是因为`write_file`有两个参数，而之前我们默认所有的函数都只有一个参数。

因此现在需要把`CallExpr`结构体的`Expr *arg`改成`Expr *args[]`数组，并加上`argc`表示参数个数。

```c
struct CallExpr {
    Node *fname; // 函数名
    uint8_t argc; // 参数个数
    Node *args[]; // 支持多个参数。 这个模式被称为Flexible Array Member(FAM)
};
```

`CallExpr`修改后，所有涉及到它的地方都要做出相应的修改。

最重要的修改是`parser.c`中解析函数调用的地方，即`call()`函数：

```c
```

## 代码生成

在`codegen.c`中，我们需要添加对`read_file`和`write_file`函数的处理。


