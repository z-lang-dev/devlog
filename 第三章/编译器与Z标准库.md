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