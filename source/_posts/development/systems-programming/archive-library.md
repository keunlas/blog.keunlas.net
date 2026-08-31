---
title: C/C++ 库文件打包与使用
date: 2026-09-01 02:24:46
tags:
  - c
  - cpp
  - compiler
  - linker
  - gcc
  - ar
categories:
  - Development
  - Systems Programming
---

库文件是将多个 `.o` 目标文件打包后的产物，供程序在编译链接时使用。根据链接方式的不同，分为静态库（`.a`）和动态库（`.so`）：

- **静态库**：在编译链接时，将库中的代码完整地复制到可执行文件中。生成的可执行文件不再依赖该库。
- **动态库**：在编译链接时，只记录库的引用；程序运行时，由动态链接器加载。多个程序可以共享同一份动态库。

# 静态库

## 打包静态库

### 源文件

```cpp
// myadd.h
int add(int a, int b);
```

```cpp
// myadd.cpp
#include "myadd.h"
#define N 0

int add(int x, int y) {
    return x + y + N;
}
```

### 编译与打包

首先将源文件编译成机器码，再使用 `ar` 命令打包成静态库。

```shell
# 编译成机器码
g++ -c myadd.cpp -o myadd.o
# 打包成静态库
ar rcs libmyadd.a myadd.o
```

静态库的命名约定为 `lib<名称>.a`。

## 使用静态库

```cpp
// main.cpp
#include <iostream>
#include "myadd.h"

int main() {
    int sum = add(1, 2);
    std::cout << "add(1, 2) = " << sum << std::endl;
    return 0;
}
```

```shell
g++ main.cpp -static -L. -lmyadd && ./a.out
```

```
add(1, 2) = 3
```

- `-L.`：在当前目录 `./` 中寻找库文件。
- `-lmyadd`：链接名为 `libmyadd` 的库。
- `-static`：以静态方式链接所有库，生成的可执行文件不再依赖任何外部库。

### 目录结构

```
test/
├── a.out
├── main.cpp
├── myadd.cpp
├── myadd.h
├── myadd.o
└── libmyadd.a
```

## `ar` 命令详解

`ar` 用于创建和管理静态库。完整语法如下：

```shell
ar [-][dmpqrtx][cfosSuvV] [a<成员>] [b<成员>] [i<成员>] [count] [备存文件] [成员文件]
```

### 常用示例

```shell
# 将 test.o 加入 libtest.a，已存在则替换；c 创建库，s 建立符号表
ar rcs libtest.a test.o

# 将 test.o 追加到 libtest.a 末尾
ar qcs libtest.a test.o
```

### 操作参数

`dmpqrtx` 为操作参数，只能且必须使用其中一个。以下将备存文件称为**库**，成员文件称为**模块**。

- `d`：删除库中指定模块。使用 `v` 选项可列出被删除的模块。
- `m`：变更库中模块的次序。当多个模块有相同的符号定义时，成员的顺序很重要。默认将指定成员移到最后，可用 `a`/`b`/`i` 指定位置。
- `p`：将库中模块的内容输出到标准输出。使用 `v` 选项可在输出前显示模块名。不指定模块名则输出全部。
- `q`：快速追加。将新模块追加到库末尾，不检查是否需要替换。`a`/`b`/`i` 对此操作无效，模块总是追加到末尾。使用 `v` 选项可列出每个模块。追加后符号表未更新，需用 `ar s` 或 `ranlib` 刷新。
- `r`：插入模块（已存在则替换）。当模块名已存在时替换同名模块。默认插入到末尾，可用 `a`/`b`/`i` 指定位置。
- `t`：显示库中的模块清单，一般只显示文件名。
- `x`：从库中提取模块。不指定模块名则提取全部。

### 修饰参数

修饰参数可与操作参数结合使用。

- `a<成员>`：在已存在的成员**之后**插入新文件。
- `b<成员>`：在已存在的成员**之前**插入新文件。
- `i<成员>`：等同于 `b`，在已存在的成员之前插入新文件。
- `c`：创建库。不论库是否存在都会创建。
- `f`：截短过长的文件名，以兼容其他系统的 `ar` 实现。
- `N`：与 `count` 配合使用，指定同名文件提取或删除的个数。
- `o`：提取成员时保留原始日期。不指定则标为提取时间。
- `P`：文件名匹配时使用全路径。`ar` 创建库时不能使用全路径（不符合 POSIX 标准），但某些工具可以。
- `s`：写入或更新库的目标文件索引。等同于 `ranlib`。
- `S`：不创建目标文件索引，加快大库的创建速度。
- `u`：仅在 `r` 操作时有效，只插入比库中同名文件更新的成员。
- `v`：显示详细信息。
- `V`：显示 `ar` 版本。

# 动态库

## 创建动态库

### 源文件

```cpp
// myadd.h
int add(int a, int b);
```

```cpp
// myadd.cpp
#include "myadd.h"
#define N 0

int add(int x, int y) {
    return x + y + N;
}
```

### 编译

```shell
# 一步到位
g++ -fPIC -shared myadd.cpp -o libmyadd.so

# 或分两步
g++ -c -fPIC myadd.cpp -o myadd.o
g++ -shared myadd.o -o libmyadd.so
```

- `-fPIC`：生成位置无关代码（Position Independent Code），使代码可以被加载到任意内存地址。
- `-shared`：生成共享对象（即动态库）。

动态库的命名约定为 `lib<名称>.so`。

## 隐式链接

隐式链接在编译时完成，使用方式与静态库类似，但代码不会被复制到可执行文件中。

### 编译链接

```cpp
// main.cpp
#include <iostream>
#include "myadd.h"

int main() {
    int sum = add(1, 2);
    std::cout << "add(1, 2) = " << sum << std::endl;
    return 0;
}
```

```shell
g++ main.cpp -L. -lmyadd && ./a.out
```

```
./a.out: error while loading shared libraries: libmyadd.so: cannot open shared object file: No such file or directory
```

编译链接成功，但运行时找不到动态库。原因是编译链接期与运行期查找动态库的路径不同：

| 阶段       | 搜索路径                                                 |
| ---------- | -------------------------------------------------------- |
| 编译链接期 | `-L` 指定的路径、`LIBRARY_PATH`、`/usr/lib`、`/lib`      |
| 运行期     | `LD_LIBRARY_PATH`、`/etc/ld.so.conf`、`/usr/lib`、`/lib` |

### 临时解决：设置 `LD_LIBRARY_PATH`

```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/test/
./a.out
```

```
add(1, 2) = 3
```

### 永久解决：设置 `RUNPATH`

程序的**动态节**（dynamic section）中可以嵌入库的搜索路径。使用 `readelf -d` 查看：

```shell
readelf -d ./a.out
```

动态节中有两个与搜索路径相关的条目：`RPATH` 和 `RUNPATH`。动态链接器按以下顺序搜索：

1. `RPATH`（已废弃）
2. `LD_LIBRARY_PATH`
3. `/etc/ld.so.conf`
4. `RUNPATH`

`RPATH` 优先级最高且无法被环境变量覆盖，因此已被废弃。推荐使用 `RUNPATH`。

在链接时通过 `-Wl` 向链接器传递参数来设置：

```shell
# 原命令
g++ main.cpp -L. -lmyadd -o a.out

# 添加 RUNPATH
g++ main.cpp -L. -Wl,-R,'$ORIGIN' -lmyadd -o a.out

# 上面那条指令是书中的，我在我的 Archlinux 上无法正确进行操作
# 我在 Archlinux 上使用下面这条指令成功完成操作
g++ main.cpp -L. '-Wl,-R${ORIGIN}' -lmyadd -o a.out
```

- `-Wl`：将后续参数传递给链接器。
- `-R`：链接器参数，用于设置 `RUNPATH`。
- `$ORIGIN`：表示可执行文件所在的目录（而非当前工作目录）。

## 显式链接

显式链接在运行时通过 `dlopen` 等函数动态加载库，无需在编译时链接。

### 源文件

```cpp
// myadd.cpp

#define N 0

extern "C" int add(int x, int y) {
    return x + y + N;
}
```

显式链接需要 `extern "C"`，否则 C++ 的 name mangling 会导致 `dlsym` 找不到函数。

### 编译动态库

```shell
g++ -fPIC -shared myadd.cpp -o myadd.so
```

### 加载与调用

```cpp
// main.cpp
#include <dlfcn.h>
#include <iostream>

int main() {
    // 加载动态库
    // RTLD_NOW：立即加载所有符号
    // RTLD_LAZY：延迟加载，调用时才解析
    auto *handle = dlopen("./myadd.so", RTLD_LAZY);
    if (handle == nullptr) {
        std::cerr << "error: dlopen" << std::endl;
        return 1;
    }

    // 获取函数指针
    using MYADD_TYPE = int (*)(int, int);
    auto add = (MYADD_TYPE)dlsym(handle, "add");
    if (add == nullptr) {
        std::cerr << "error: dlsym" << std::endl;
        dlclose(handle);
        return 1;
    }

    // 调用函数
    int sum = add(1, 2);
    std::cout << "add(1, 2) = " << sum << std::endl;

    // 关闭动态库
    dlclose(handle);
    return 0;
}
```

```shell
g++ main.cpp && ./a.out
```

```
add(1, 2) = 3
```





