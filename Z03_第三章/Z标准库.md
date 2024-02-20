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
static Node *call(Parser *parser) {
  Node *fn = fname(parser);
  expect(parser, TK_LPAREN);
  ArgBuf *buf = args(parser);
  expect(parser, TK_RPAREN); 
  Node *node = malloc(sizeof(NodeKind) + sizeof(CallExpr) + buf->count * sizeof(Node *));
  node->kind = ND_CALL;
  node->as.call.fname = fn;
  node->as.call.argc = buf->count;
  for (int i = 0; i < buf->count; i++) {
    node->as.call.args[i] = buf->data[i];
  }
  trace_node(node);
  return node;
}
```

`call`的结构现在如下：

1. 调用`fname`解析函数名
2. 跳过`(`
3. 调用`args`解析参数列表，注意参数的个数并不是固定的，这里我们先假设不能超过4个。
4. 跳过`)`
5. 组装`Node`，其中最麻烦的部分在于`args`的组装。

由于`args`的参数在解析参数列表完成之前是不知道的，但我们的`CallExpr`的`args`字段采用的是`FAM`形式，必须在初始化时指定长度。
所以我在这里使用了一个`ArgBuf`结构体来临时存储参数，等解析完成后再把它们复制到`CallExpr`中。`ArgBuf`有最大长度，但好处是不用每次遇到函数调用都重新分配内存了，起到一个缓存的作用。

注意，这里的`ArgBuf`暂时不支持嵌套调用，即类似于
`pow(2, pow(2, 3))`这样的调用。等Z语言添加视野特性，支持嵌套调用时，我们需要更复杂的数据结构来支持。

现在语法分析器可以处理多个参数的函数调用了，但是后端还没有真正支持它。

对于所有用到`arg`的地方，都要从处理单个`arg`修改成遍历`args`数组，并处理其中每一个参数的逻辑。

## 解释器

由于解释器现在支持的`命令`，对应的函数参数要么是空要么是一个，暂时也不需要处理多个参数。但为了代码编译通过，以及给后续扩展留下余地，我还是对解释器的结构做了修改。

首先，之前只处理了`print`、`ls`、`pwd`、`cd`这几个内置函数，所以把他们挪出去成为单独的函数，叫`call_builtin`，

```c
bool call_builtin(Node *expr) {
    char *fname = expr->as.call.fname->as.str;
    if (strcmp(fname, "print") == 0) {
        print(expr->as.call.args[0]);
        return true;
    } else if (strcmp(fname, "pwd") == 0) {
        pwd();
        return true;
    } else if (strcmp(fname, "ls") == 0) {
        ls(expr->as.call.args[0]->as.str);
        return true;
    } else if (strcmp(fname, "cd") == 0) {
        cd(expr->as.call.args[0]->as.str);
        return true;
    } else if (strcmp(fname, "cat") == 0) {
        cat(expr->as.call.args[0]->as.str);
        return true;
    }
    return false;
}
```

注意这里的参数从之前的`expr->as.call.arg`变成了`expr->as.call.args[0]`。
我暂时忽略了参数数量的检查，因为打算之后支持自定义函数时，需要更多的参数时，统一检查。

接着再添加一个类似的处理函数`call_stdlib`，用来处理标准库函数。

```c
// 现在的标准库只有`read_file`和`write_file`两个函数
bool call_stdlib(Node *expr) {
    char *fname = expr->as.call.fname->as.str;
    if (strcmp(fname, "read_file") == 0) {
        read_file(expr->as.call.args[0]->as.str);
        return true;
    } else if (strcmp(fname, "write_file") == 0) {
        // 这里暂时没有检查参数个数，因为未来要做统一检查
        write_file(expr->as.call.args[0]->as.str, expr->as.call.args[1]->as.str);
        return true;
    }
    return false;
}
```

注意：由于我们的解释器是C写的，并且现在选择了最简单的方案：将`std`库直接静态链接到z解释器里，
因此只需要直接调用`read_file`和`write_file`就行了。

未来标准库扩容的时候，这种方案就不行了，到时候需要改成动态链接库的形式。
考虑到动态链接库牵涉到更多操作系统的差异，比静态链接要复杂不少，我打算等标准库扩容的时候再来实现。

现在解释器处理`ND_CALL`的逻辑就变得很清晰了：

```c
case ND_CALL:
    if (call_builtin(expr)) break;
    if (call_stdlib(expr)) break;
    printf("Unknown function: %s\n", expr->as.call.fname->as.str);
    break;
```

将来要加上对自定义函数的处理时，在这里在加一条`call_custom`的分支就行了。

这样的多分支查找逻辑对解释器很重要，
有了这个框架，未来我们还可以考虑在REPL和解释器里同时支持C、Python和JS的后端。

现在，我们就可以在解释器里调用`read_file`和`write_file`了。

```bash
$ xmake run -w work z repl
Z REPL v0.1
>>> read_file("hello.z")
print("Hello, world!")
>>> write_file("hello.tmp", "hello tmp!\n")
>>> read_file("hello.tmp")
hello tmp!
```

