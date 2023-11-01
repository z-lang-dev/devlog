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
