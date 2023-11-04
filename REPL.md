# 最简单的交互器

这里说的交互器，就是我们平时所说的REPL。

输入一行代码，REPL会读取代码，进行计算，并打印结果；接着再输入一行代码，REPL再读取代码，进行计算，并打印结果；如此往复。

Read，Evaluate，Print，Loop。这就是REPL。

我最熟悉的REPL大概是Python的，尤其是IPython，它的出现重新定义了什么是优秀的REPL，并且后来又衍生出另一种交互体验：Jupyter Notebook。

我希望Z语言也能做到这样的交互体验。

我还希望可以在Z的REPL可以直接当做命令行来使用，类似于bash。

如果这两种功能结合起来，即bash+ipython，将会是一种美妙的组合。

当然，这样的交互器没那么容易实现，显然不适合放在本书的这个阶段。

所以我打算先从最基础的R-E-P-L循环开始，每次解释一行代码。

## REPL基本流程

我们在`main.c`的`repl`函数里先打出R-E-P-L的基本流程：

```c
// 交互式环境REPL
static void repl(void) {
    printf("Z REPL v0.1\n");

    // Loop
    for (;;) {
        // Read
        printf("-------------- \n");
        printf(">>> ");
        // Read
        char *line = NULL;
        size_t len = 0;
        int nread = getline(&line, &len, stdin);
        if (nread == -1) {
            printf("\n");
            break;
        }
        // 去掉行尾的换行符
        line[nread-1] = '\0';
        // 解析源码
        CallExpr *expr = parse_expr(line);

        // Evaluate & Print
        // 执行
        execute(expr);
    }
}
```

在这个`repl()`函数里：

- `Read`操作由`read_line()`和`parse_expr()`负责
- `Evaluate`操作由`execute()`负责
- `Print`操作，由于现在Z语言只有`print`函数，所以直接放在`execute()`里了，未来会独立出来。
- `Loop`操作用`for(;;)`循环实现

这四个操作合起来就是R-E-P-L，REPL。

这里的`parse_expr()`和`execute()`都在`interp`里实现过了，只有`getline()`是新的函数。

在Linux中，`getline()`是`<stdio.h>`库里的标准函数，我们不需要实现；但在Windows下，没有这个函数，我们需要自己实现。

我在网上找了个实现：

```c
typedef intptr_t ssize_t;

ssize_t getline(char **lineptr, size_t *n, FILE *stream) {
    size_t pos;
    int c;

    if (lineptr == NULL || stream == NULL || n == NULL) {
        errno = EINVAL;
        return -1;
    }

    c = getc(stream);
    if (c == EOF) {
        return -1;
    }

    if (*lineptr == NULL) {
        *lineptr = malloc(128);
        if (*lineptr == NULL) {
            return -1;
        }
        *n = 128;
    }

    pos = 0;
    while(c != EOF) {
        if (pos + 1 >= *n) {
            size_t new_size = *n + (*n >> 2);
            if (new_size < 128) {
                new_size = 128;
            }
            char *new_ptr = realloc(*lineptr, new_size);
            if (new_ptr == NULL) {
                return -1;
            }
            *n = new_size;
            *lineptr = new_ptr;
        }

        ((unsigned char *)(*lineptr))[pos ++] = c;
        if (c == '\n') {
            break;
        }
        c = getc(stream);
    }

    (*lineptr)[pos] = '\0';
    return pos;
}
```

这个实现是在<https://stackoverflow.com/a/47229318>找到的，大家可以自己去看看。

现在我们可以尝试运行REPL了：

```bash
$ xmake build
$ xmake run z repl
Hello from Z!
Z REPL v0.1
--------------
>>> print("Hello")
Parsing print("Hello")...
CallExpr {
  fn: print
  arg: "Hello"
}
Executing print(Hello)...
Hello
--------------
```

成功了！

至此我们可以再提交一个新的标签了。

```bash
$ git commit -a -m "步骤7：最简单的交互器"
$ git tag -a v0.0.7 -m "步骤7：最简单的交互器"
$ git push
```

## 小结

大家可能会觉得这一节内容太少了，我本来也想再加点东西，比如保存上一步的结果，或者记录历史命令啊等等。
但考虑之后我放弃了，因为这些东西都不属于“最简单的REPL”的范畴，而且会过早对编译器提出更复杂的需求。

我打算等完成下一大章，即完整的数学运算功能之后，再来丰富REPL的功能。