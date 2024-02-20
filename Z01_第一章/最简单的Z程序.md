# 最简单的Z程序

## Hello World?

一般来说，介绍语言特性时，第一个例子都是Hello World。

比如C:

```c
#include <stdio.h>

int main(int argc, char** argv) {
  printf("Hello, world!\n");
  return 0;
}
```

Python:

```python
print("Hello, world!")
```

JS:

```js
console.log("Hello, world!")
```

从这两个简单的例子我们也能看出C和Python/JS的巨大差别。

那么Z语言的Hello World应该是什么样的呢？可以和Python一样：

```z
print("Hello, world!")
```

也可以和C类似：

```z
use io.print

fn main {
  print("Hello, world!")
}
```

同样都是Z代码，为什么会有这么大的差别呢？真正的区别在于**场景**。

当作为脚本运行的时候，Z解释器会模仿Python，自动建立好初始的环境，并自动导入一些常用的库，比如io库。
而当作为静态编译程序的主入口时，Z编译器会要求程序明确地指定所有的依赖和程序的真正入口（`main`函数），这样才能达到最高的运行效率。

所以，前面那段代码相当于把`use`和`fn`语句自动补全了再运行的。

本章的内容，就是把这段最简单的Hello World程序，用最取巧的方法实现一遍。
顺便把Z编译器的框架搭建起来。


## 工具选择

工欲善其事，必先利其器。在开始编码之前，我们要先选择好工具。

我选择的开发环境如下：

- 操作系统：Win11/WSL2，这么做的好处是可以快速地验证Windows和Linux两套系统下的表现。而这两套系统也是Z语言主打的平台。
- 编辑器：VSCode，跨平台代码编辑器的不二之选。
- 代码仓库：gitee.com，快。
- 命令行工具：cmder/bash
- 开发语言：C。我选择C是因为Z语言最开始选定的目标语言就是C，它也是跨平台语言的最好选择之一。将来实现自举时，从C->Z的转化也会比较容易。
- 编译器：msvc/clang。Windows下直接用msvc，Linux下用clang。
- 构建工具：xmake。Z编译器不是大工程，不需要上cmake。xmake更简单小巧，且是国人开发的。

## 新建工程

选定了xmake，所以要先学习一下它怎么用。

我们先用xmake在上述的开发环境下新建一个工程。

第一步，在gitee.com上建立一个git仓库，并克隆到本地。目录是`zc`，它默认有个README.md文件。

第二步，进入`zc`目录，用xmake创建一个C语言工程:

```bash
xmake create -f -l c -P .
```

这里的参数`-f`表示强制新建工程，因为`zc`目录下已经有文件了，不添加这个参数就会报错；
`-l c`参数表示使用C语言，而`-P .`表示使用当前目录作为工程目录。

此时`xmake`会生成工程配置文件`xmake.lua`，以及`src`目录和`src/main.c`文件。

查看`src/main.c`文件，发现它是一个最简单的C语言程序：

```c
#include <stdio.h>

int main(int argc, char** argv)
{
    printf("hello world!\n");
    return 0;
}
```

把`printf`的内容修改为"Hello from Z!\n"，然后调用`xmake`编译。

我们可以调用`xmake run`来运行编译好的程序：

```bash
$ xmake run
Hello from Z!
```

至此最简单的工程建立好了！

我们可以提交一次代码，并打上tag："step1: new project"。

```bash
git commit -a -m "step1: new project"
git tag v0.0.1 -m "step1: new project"
git push
```

注意，如果要把tag也一起提交，应当配置`followTags`：

```bash
git config --global push.followTags true
```

或者在vscode里，修改settings，将"Git: Follow Tags When Sync"打钩。

## 编译器的输出内容

`Hello World`这样简单的程序，应该输出什么呢？

换句话说，Z编译器的输出应当是什么呢？

Z编译器既支持解释执行，也支持静态编译，因此我们有两个流程：

1. 解释执行：直接运行`z "print(\"Hello, world!\")"`，输出`Hello, world!`。或者直接运行`z`，在类似Python的交互环境中输入代码，获得解释执行的结果（这种模式也叫REPL）。
2. 编译执行：把Z源码写入`hello.z`文件，再执行`z build hello.z`得到可执行文件`hello.exe`，运行这个文件得到输出。

本节我们就先把这两个流程的框架搭建起来，而具体怎么解释和编译的过程先忽略掉。

## 基本编译命令

第一步，添加几个命令：

- `repl`：进入REPL环境
- `interp`：解释执行源码
- `build`：编译文件
- `run`：编译并运行文件，相当于先`build`，再运行生成的可执行文件

我们先把这几个命令的架子搭好，实现的细节先不管，直接输出“TODO”即可：

```c
static void repl(void) {
    printf("TODO: repl\n");
}

static void interp(char *code) {
    printf("TODO: interp %s\n", code);
}

static void build(char *file) {
    printf("TODO: building %s\n", file);
}

static void run(char *file) {
    printf("TODO: run %s\n", file);
}
```

可以看出，`repl`命令和其他三个不同，不需要参数；而`interp`的参数是一段代码；`build`和`run`的参数都是一个文件名。

接着在`main`函数里处理并调用对应的命令函数：

```c
// 第一个参数是命令名称：interp|repl|build|run
char *cmd = argv[1];

// 如果命令是repl，直接进入repl()交互环境
if (strcmp(cmd, "repl") == 0) {
    repl();
    return 0;
}

// 剩下的命令都需要提供内容（代码或文件名称）
if (argc < 3) {
    help();
    return 1;
}

// 根据命令执行不同的操作
if (strcmp(cmd, "interp") == 0) {
    interp(argv[2]);
} else if (strcmp(cmd, "build") == 0) {
    build(argv[2]);
} else if (strcmp(cmd, "run") == 0) {
    run(argv[2]);
} else {
    help();
}
```

编译并运行：

```bash
$ xmake build
$ xmake run z repl
Hello from Z!
TODO: repl
```

这里`xmake run z`可以直接运行`z.exe`可执行文件，相当于执行`build\windows\x64\release\z.exe`。
其他几个命令也如此。

现在我们可以提交代码并打上第二个标签了：

```bash
git commit -a -m "step2: add basic commands"
git tag v0.0.2 -m "step2: add basic commands"
git push
```

接下来我们一个个实现。先从最简单的`interp`开始。

## 实现`interp`

`interp`的作用是将`code`参数作为一行代码，解析成AST语法树（我们叫它zast），再执行它，得到结果。

例如，如果执行：`z interp "1+1"`，结果应当是`2`。

我们本章用最简单的办法实现`z interp "print(\"Hello, world!\")"`，输出`Hello, world!`。

我们先来定义AST的结构。这里的`print("Hello, world!")`是一个函数调用的表达式，我们取名为`CallExpr`，在最简单的情况下，它的结构如下：

```c
// zast.h
struct CallExpr {
    char *fn; // 函数名称
    char *arg; // 参数
};
```

`fn`就是函数名，在上述例子中就是`print`；而现在Z语言只需要支持一个参数，因此用一个字符串类型的`arg`字段来表示就可以了，上述例子中`arg`的值应当解析为`"Hello, world!"`。
除了这两个字段，表达式里还有几个东西：括号`()`、引号`""`，但这些都是为了方便阅读和避免歧义而设置的分隔符，在解析完成之后，就不需要在AST中存放了。

我们接下来的工作就是把`"print(\"Hello, world!\")"`解析成`CallExpr`结构，再想办法执行它。

考虑到现在Z语言只支持一个函数`print`，且参数只有一个字符串，我们可以用最笨的办法来简单地解析它：

- 从代码开头到左括号`'('`之前的内容就是函数名`print`；
- 两个引号`'"'`之间的内容就是参数字符串`Hello, world!`。

因此我们只需要找到`'('`的位置`index_lparen`即可，这样，函数名称就是`0~index_lparen`的子串，
而参数字符串就是`(index_lparen+2)`到`strlen(code)-2`的子串。
注意这里之所以是`+2`和`-2`，是因为我们还要剔除掉引号`"`。

```c
// 解析表达式
static CallExpr *parse_expr(char *code) {
    printf("Parsing %s...\n", code);
    // 解析源码
    CallExpr *expr = calloc(1, sizeof(CallExpr));
    // 从代码开头到'('之间的就是函数名称
    int index_lparen = index_of(code, '(');
    char *fn = substr(code, 0, index_lparen);
    expr->fn = fn;
    // 从'"'到'"'之间的就是参数
    char *arg = substr(code, index_lparen + 2, strlen(code)-2);
    expr->arg = arg;
    // 打印出AST
    print_ast(expr);
    return expr;
}
```

这里用到的`index_of`和`substr`都是很简单的字符串操作，直接参考源码即可。

有了这个解析函数，`interp()`的内容就很简单了：

```c
// 解释并执行代码
static void interp(char *code) {
    printf("Interpreting %s...\n", code);
    // 解析源码
    CallExpr *call = parse_expr(code);
    execute(call);
}
```

执行函数`execute()`现在也很简单，直接调用C标准库的`printf`即可：

```c
// 执行AST
static void execute(CallExpr *call) {
    printf("Executing %s(%s)...\n", call->fn, call->arg);
    // 打印call.arg
    printf("%s\n", call->arg);
}
```

我们试试效果：

```bash
$ xmake run z interp "print(\"Hello world!\")"
Hello from Z!
Interpreting print("Hello world!")...
Parsing print("Hello world!")...
CallExpr {
  fn: print
  arg: "Hello world!"
}
Executing print(Hello world!)...
Hello world!
```

看最后一行输出，`Hello world!`已经打印出来了！

换个信息打印：

```bash
$ xmake run z interp "print(\"Byebye from Z!\")"
Hello from Z!
Interpreting print("Byebye from Z!")...
Parsing print("Byebye from Z!")...
CallExpr {
  fn: print
  arg: "Byebye from Z!"
}
Executing print(Byebye from Z!)...
Byebye from Z!
```

这样，最简单的`interp`就实现好了。

我们提交代码并打上第3个标签：

```bash
git commit -a -m "step3: implement simple interpreter"
git tag v0.0.3 -m "step3: implement simple interpreter"
git push
```

这个实现有不少问题：

- 现在Z语言只支持一句话，而且只能是一个`print("msg")`的调用。
- 我们虽然解析了`print`函数名，但实际上并没有用到它，随便换个名字，例如`no_print`，程序还是会打印出输入的信息。这显然是不对的，我们需要检查函数名是否是语言所支持的，并输出错误信息。
- 如果源码是`print(1)`，打印一个整数，得不到正确的结果，这是因为解析函数认定了`""`。

而循序渐进地改进这些问题，就能逐步做出一个完整的解释起来。


## 小结

本节我们做出了最简单的Z解释器，并能够执行Hello World程序。

下一节我们仍然只处理Hello World，但不同的是要实现一个最简单的编译器，可以执行`build`和`run`命令。
