# REPL与内置函数

## 内置函数

到现在为止，实际上Z语言还没有内置函数的概念。

我们虽然说`print`是一个内置函数，但并没有对它进行任何判断。

实际上，由于现在Z语言的全部函数调用都只是调用了`printf`，所以我们连`print`的函数签名都没有检查。

下面这个代码也会打印出信息：

```z
$ xmake run z repl
Z REPL v0.1
>>> abc("hello")
hello
```

看看解释器的代码就知道了，不论`ND_CALL`节点的`fname`内容是什么，我们都没有去检查它。
只要是`ND_CALL`节点，我们都直接调用`printf`。

```c
// interp.c
void execute(Node *expr) {
    // ...
    case ND_CALL:
        // 这里直接取出arg，调用`printf`，并没有检查`fname`的内容
        Node *arg = expr->as.call.arg; 
        if (arg->kind == ND_INT) {
            printf("%d\n", arg->as.num);
        } else {
            printf("%s\n", arg->as.str);
        }
        break;
    // ...
}
```

现在加上了更多的内置函数，函数名的检查就是必须的了。

## 解释器

首先，我们加上`print`、`pwd`、`ls`和`cd`这几个内置函数在解释器里的实现：

```c
// print
static void print(Node *arg) {
    if (arg->kind == ND_INT) {
        printf("%d\n", arg->as.num);
    } else {
        printf("%s\n", arg->as.str);
    }
}

// pwd
static void pwd() {
    char buf[1024];
    getcwd(buf, sizeof(buf));
    printf("%s\n", buf);
}

// ls
static void ls(char *path) {
    char cmd[1024];
    sprintf(cmd, "ls %s", path);
    system(cmd);
}

// cd
static void cd(char *path) {
    if (chdir(path) != 0) {
        perror("chdir");
    }
}
```

这里面，`print`、`pwd`和`cd`都可以直接调用C的标准库函数，但`ls`的库函数在Windows和Linux下有很大不同。
因此我先用更简单的`system`命令去替代标准库函数了。
未来有需要时，再实现一个跨平台的`ls`函数。

修改`execute`函数，根据`fname`选择内置命令：

```c
case ND_CALL:
    // 打印call.arg
    char *fname = expr->as.call.fname->as.str;
    if (strcmp(fname, "print") == 0) {
        print(expr->as.call.arg);
    } else if (strcmp(fname, "pwd") == 0) {
        pwd();
    } else if (strcmp(fname, "ls") == 0) {
        ls(expr->as.call.arg->as.str);
    } else if (strcmp(fname, "cd") == 0) {
        cd(expr->as.call.arg->as.str);
    } else {
        printf("Unknown builtin function: %s\n", fname);
    }
    break;
```

这样我们就可以在REPL里调用这几个命令了：

```z
$ xmake run z repl
Z REPL v0.1
>>> pwd()
D:\gitee\z-lang\zc\build\windows\x64\debug
>>> ls(".")
 Directory of D:\gitee\z-lang\zc\build\windows\x64\debug

2023.11.16 周四  01:34    <DIR>          .
2023.11.07 周二  16:09    <DIR>          ..
2023.11.14 周二  15:53           208,896 compile.test_compiler.pdb
2023.11.14 周二  15:53           217,088 compile.test_interp.pdb
2023.11.14 周二  15:53           217,088 compile.test_transpiler.pdb
2023.11.16 周四  01:34           176,128 compile.z.pdb
2023.11.14 周二  15:53           676,864 test_compiler.exe
2023.11.14 周二  15:53         4,145,248 test_compiler.ilk
2023.11.14 周二  15:53         7,270,400 test_compiler.pdb
2023.11.14 周二  15:53           676,864 test_interp.exe
2023.11.14 周二  15:53         4,146,368 test_interp.ilk
2023.11.14 周二  15:53         7,270,400 test_interp.pdb
2023.11.14 周二  15:53           677,888 test_transpiler.exe
2023.11.14 周二  15:53         4,148,760 test_transpiler.ilk
2023.11.14 周二  15:53         7,270,400 test_transpiler.pdb
2023.11.16 周四  01:34           725,504 z.exe
2023.11.16 周四  01:34         4,551,568 z.ilk
2023.11.16 周四  01:34         8,417,280 z.pdb
              16 File(s)     50,796,744 bytes
               2 Dir(s)  536,625,942,528 bytes free
>>> cd("..")
>>> pwd()
D:\gitee\z-lang\zc\build\windows\x64
```

这里有两个问题：

- 使用函数调用的形式，写起来确实比命令行命令麻烦
- REPL开启后默认的`pwd`是`z`程序所在的目录，这里由于是开发版本，所以跑到工程下的`debug`目录了。实际上，我们希望默认的`pwd`是当前用户所在的目录。

要解决第一个问题，需要再`repl`里单独判断`<cmd> <arg>`这种形式的命令，把它们转化为`<cmd>(<arg>)`的函数调用形式。

## REPL的内置命令

```c
// REPL内置的命令
char *commands[] = {
    "print",
    "pwd",
    "ls",
    "cd",
    NULL
};

// 把命令转换成Z函数
static char *command(char *line) {
    for (int i = 0; ; i++) {
        char *cmd = commands[i];
        if (cmd == NULL) return line;
        if (starts_with(line, cmd)) {
            size_t len = strlen(cmd);
            char *res = calloc((int)strlen(line) + 16, sizeof(char));
            if (line[len] == '\0') { // 空命令，例如`ls`或者`pwd`
                if (strcmp(cmd, "ls") == 0) {
                    return "ls(\".\")";
                }
                sprintf(res, "%s()", cmd);
                return res;
            } else if (line[len] == ' ') { // 带参数的命令
                // 取得参数
                char *arg = substr(line, len + 1, strlen(line));
                if (!is_digit(arg[0])) { 
                    // 命令参数可以不带引号，但转换成Z函数时需要加上引号
                    sprintf(res, "%s(\"%s\")", cmd, arg);
                } else {
                    sprintf(res, "%s(%s)", cmd, arg);
                }
                printf("Got command: %s\n", res);
                return res;
            } else {
                return line;
            }
        }
    }
}
```

我们在`repl.c`里添加一个`char *command(char *line)`函数，
它可以把`cd ..`这样的命令转化为`cd("..")`内置函数调用形式。

现在REPL支持的内置命令有：`print`、`pwd`、`ls`和`cd`。

现在我们再试试：

```bash
>>> pwd
D:\gitee\z-lang\zc\build\windows\x64\debug
```

的确可以用命令行的格式了。

但`z repl`的默认目录还是不对，我们需要在REPL启动的时候，设置一下当前运行目录。

## 工作目录

经过一番调查，我发现上面讲到的`debug`目录问题，实际上是`xmake`的设置。

在`xmake`中，可能是为了不污染源码的目录，`xmake run`默认的工作目录是生成的`.exe`文件所在的目录。
在Windows中，这个目录是`build/windows/x64/debug`。

而如果我们直接在`zc`根目录下运行`z.exe`，会发现`pwd`的结果是就是启动时所在的`zc`根目录了：

```z
$ cd gitee/z-lang/zc
$ .\build\windows\x64\debug\z.exe repl
Z REPL v0.1
>>> pwd
D:\gitee\z-lang\zc
```

也就是说，如果是正常运行`.exe`文件，默认的工作目录是符合我们的预期的。

所以，如果想使用`xmake run`的话，按照它的官方文档，用`-w <working-dir>`参数来运行，就没问题了：

```bash
$ cd gitee/z-lang/zc
$ xmake run -w . z repl
Z REPL v0.1
>>> pwd
D:\gitee\z-lang\zc
```

因此，我们不需要在REPL启动时修改工作目录了。

## 小结

至此我们已经实现了几个解释器的内置函数，并在REPL中关联了对应的内置命令。

这是函数的第一个应用，未来我们会给解释器和REPL添加更多方便使用的内置函数。

下一节，我们探讨一下如果在编译器中调用C标准库函数或系统调用来完成更多的功能。

这一章我们涉及的主要是内置函数，下一章就要讨论Z语言的标准库了。

