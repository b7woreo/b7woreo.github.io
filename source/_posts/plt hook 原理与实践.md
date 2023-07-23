---
title: 'plt hook 原理与实践'
date: 2023-07-09
tags:
- Android
---

# 前言

在 Android 应用开中, 针对于 native 的 hook 方案主要分为两种:

- __PLT Hook:__ 基于地址无关代码(PIC, Position-independent Code)原理设计的 hook 方案, 可用于 hook 对外部函数的调用, 代表项目: [bhook](https://github.com/bytedance/bhook).
- __Inline Hook:__ 通过在代码段中插入新的指令实现 hook 的方案, 代表项目: [android-inline-hook](https://github.com/bytedance/android-inline-hook).

本文主要对 PLT Hook 的原理进行探索和实践.

# 基本原理

## ELF 文件

![](/images/202307231659.png)

ELF 全称: Executable and Linkable Format, 是 Unix 和类 Unix 系统的标准二进制文件格式. 常见的 ELF 文件有:

- __可重定位文件(Relocatable File):__ 这类文件包含了代码和数据, 可以被用来链接成可执行文件或共享目标文件.
- __可执行文件(Executable File):__ 这类文件包含了可以直接执行的程序, 它们一般都没有扩展名.
- __共享目标文件(Shared Object File):__ 这种文件包含了代码和数据, 可以在以下两种情况下使用: 一种时链接器可以使用这种文件跟其他的可重定位文件和共享目标文件链接, 产生新的目标文件; 第二种是动态链接器可以将几个这种共享目标文件与可执行文件结合, 做为进程映像的一部分来运行.

ELF 文件主要是由各类 __段__ 组成, 在不同的视图模型下 __段__ 会有不同的含义.

### 链接视图

在 ELF 文件被加载到内存之前, 编译器主要是通过读取 `Section header table` 中的信息, 以 `Section` 为单元操作修改 ELF 文件.  
下面是进行 PLT Hook 所需要涉及的 `Section` 类型, 具体用法在后续章节讲解.

- `.dynamic`: 保存了动态链接器所需要的基本信息, 比如动态链接的符号表的位置, 动态链接重定位表的位置等.
- `.dynsym`: 与动态链接相关的符号, 不包括模块内部的符号.
- `.dynstr`: 动态链接时的符号字符串表.
- `.rel.dyn / .rela.dyn`: 用于数据重定位, 重定位的位置位于 `.got`.
- `.rel.plt / .rela.plt`: 用于函数重定位, 重定位的位置位于 `.got.plt`.
- `.got`: 外部数据跳转表.
- `.got.plt`: 外部函数跳转表.
- `.plt`: 外部函数调用中间表, 用于实现函数延迟绑定.

### 执行视图

当 ELF 文件被加载时(无论是可执行文件还是动态链接库), 系统内核会读取 `Program header table` 中的信息, 以 `Fragment` 为单元使用页映射的方式将 ELF 文件加载到内存中.     
执行视图相较于链接视图, 会将多个拥有相同内存权限的 `Section` 合并到一个的 `Fragment` 中.

## 动态库加载

PLT Hook 主要涉及的是共享目标文件, 下面将介绍动态链接库加载时会使用到的两个关键技术.

### 地址无关代码(PIC)

动态链接相较于静态链接最大的区别在于符号的地址信息在被加载时才知道, 于是编译器会把符号的处理分成两类:

- 模块内部符号: 在编译期可以知道相对地址, 在运行期使用相对地址进行访问.
- 模块外部符号: 编译期在 `.got` 中预留地址空间, 在动态链接库被加载时由动态链接器负责填入正确的地址.

于是动态链接库中 `.data` 和 `.text` 的内存数据可以在多个进程中共享, 每个进程又会有自己的独享的 `.got` 内存数据. 当动态链接库调用外部函数时流程:

![](/images/20230709234507.png)

### 延迟绑定(PLT)

PIC 有一个缺点, 就是在动态链接库被加载时需要完成对所有符号的处理工作, 这会影响程序的启动速度. 于是在 PIC 的基础上衍生出了 PLT 技术. 当动态链接库调用外部函数时流程:

![](/images/20230709234522.png)

相较于 PIC, PLT 多出了 `.plt` 这个中间跳转表, `.plt` 中的大致逻辑:

1. 判断函数符号在 `.got.plt` 中的地址是否已经初始化, 如已初始化跳转至步骤 `3`, 否则执行步骤 `2`;
2. 使用 `_dl_runtime_resolve` 初始化 `.got.plt` 中的函数符号地址;
3. 读取 `.got.plt` 中的对应函数符号地址, 跳转至该地址完成函数执行.

## 原理总结

动态链接库使用到外部符号时, 一定会通过 `.got` 或 `.got.plt` 确定符号运行时的地址, 所以只要在运行时修改 `.got` 和 `.got.plt` 中的符号地址信息, 就能实现 hook 操作.

# 代码实践

在知晓 PLT Hook 的基本原理后, 下面将在 Linux 操作系统中动手进行实践: 

1. 编写一个动态库, 动态库中会使用到 `printf` 函数;
2. 通过 `dl_iterate_phdr` 在程序运行时获取动态库的信息, 并完成对 `printf` 的 hook 操作.



## 编写动态库

文件: lib.h
```c
void hello_world();
```

文件: lib.c
```c
#include "lib.h"
#include <stdio.h>

void hello_world(){
  printf("hello world\n");
}
```

通过执行 `gcc -fPIC --shared lib.c -o lib.so` 命令得到可用于动态链接的 `lib.so` 文件.

## 编写主程序

文件: main.c

```c
#define _GNU_SOURCE
#include "lib.h"
#include <link.h>
#include <stdio.h>
#include <string.h>
#include <stdarg.h>

void my_printf(const char *str) {
  printf("before printf: %s\n", str);
  printf("%s\n", str);
  printf("after printf: %s\n", str);
}

int main() {
  hook();
  hello_world();
  return 0;
}

void hook() { 
  // 见: "编写 hook 操作" 章节
}
```

通过执行 `gcc main.c lib.so -Wl,-rpath ./ -o main` 命令得到可用于执行的 `main` 文件, `main` 在运行时会动态链接 `lib.so` 文件.   
`-Wl,-rpath ./` 指定了运行时共享目标文件查找目录, 所以运行时要保证 `main` 和 `lib.so` 文件在同一个目录下.

## 确认基本信息

### 符号名

```bash
$ objdump -d lib.so

...

0000000000001050 <puts@plt>:
    1050:       f3 0f 1e fa             endbr64 
    1054:       f2 ff 25 bd 2f 00 00    bnd jmpq *0x2fbd(%rip)        # 4018 <puts@GLIBC_2.2.5>
    105b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

...

0000000000001119 <hello_world>:
    1119:       f3 0f 1e fa             endbr64 
    111d:       55                      push   %rbp
    111e:       48 89 e5                mov    %rsp,%rbp
    1121:       48 8d 3d d8 0e 00 00    lea    0xed8(%rip),%rdi        # 2000 <_fini+0xed0>
    1128:       e8 23 ff ff ff          callq  1050 <puts@plt>
    112d:       90                      nop
    112e:       5d                      pop    %rbp
    112f:       c3                      retq   
```

通过 `objdump` 命令可以看到, `hello_world` 中的 `printf` 函数调用被替换成了 `puts@plt` 函数, 而 `puts@plt` 函数最终会调用 `puts@GLIBC_2.2.5` 函数. 因此接下来将需要修改 `.got.plt` 中 `puts` 函数的地址. 

### 重定位表结构

```bash
$ readelf -d lib.so

Dynamic section at offset 0x2e20 contains 24 entries:
  Tag        Type                         Name/Value
 0x0000000000000014 (PLTREL)             RELA

...

```

在 ELF 文件中, 有两种格式的重定位表: `rel` 和 `rela`, 为了简化下面的 hook 逻辑这里先通过 `readelf` 读取 `.dynamic` 段中 `PLTREL` 的信息得知 `lib.so` 中使用的是 `rela` 格式, 因此在下面进行 hook 操作时将进行一些简化, 不再对重定位表的格式进行判断.

### 内存权限

```bash
$ readelf -l lib.so

Elf file type is DYN (Shared object file)
Entry point 0x1060
There are 11 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000530 0x0000000000000530  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x000000000000013d 0x000000000000013d  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x00000000000000cc 0x00000000000000cc  R      0x1000
  LOAD           0x0000000000002e10 0x0000000000003e10 0x0000000000003e10
                 0x0000000000000218 0x0000000000000220  RW     0x1000
  DYNAMIC        0x0000000000002e20 0x0000000000003e20 0x0000000000003e20
                 0x00000000000001c0 0x00000000000001c0  RW     0x8
  NOTE           0x00000000000002a8 0x00000000000002a8 0x00000000000002a8
                 0x0000000000000020 0x0000000000000020  R      0x8
  NOTE           0x00000000000002c8 0x00000000000002c8 0x00000000000002c8
                 0x0000000000000024 0x0000000000000024  R      0x4
  GNU_PROPERTY   0x00000000000002a8 0x00000000000002a8 0x00000000000002a8
                 0x0000000000000020 0x0000000000000020  R      0x8
  GNU_EH_FRAME   0x000000000000200c 0x000000000000200c 0x000000000000200c
                 0x000000000000002c 0x000000000000002c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002e10 0x0000000000003e10 0x0000000000003e10
                 0x00000000000001f0 0x00000000000001f0  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.property .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt 
   01     .init .plt .plt.got .plt.sec .text .fini 
   02     .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.gnu.property 
   06     .note.gnu.build-id 
   07     .note.gnu.property 
   08     .eh_frame_hdr 
   09     
   10     .init_array .fini_array .dynamic .got 
```

在 `Section to Segment mapping` 中可以找到 hook 时需要修改的 `.got.plt` 位于 `03` 段中, 而在 `Program Headers` 中找到 `03` 段的信息可以看到 `Flags` 是 `RW`, 因此可以在运行时直接对 `.got.plt` 中的信息进行修改.

## 编写 hook 操作

基于 PLT Hook 的基本原理, `hook` 函数的实现主要分为以下几个步骤:

1. 通过 `dl_iterate_phdr` 遍历当前进程中所有 `.so` 的信息, 找到 `lib.so` 的地址;
2. 遍历 `.dynamic` 段中的信息, 找到 `.rela.plt` 的地址;
3. 遍历 `.rela.plt`段中的信息, 找到 `.got.plt` 段中 `puts` 函数的地址;
4. 将 `.got.plt` 段中 `puts` 函数的地址改为 `my_printf` 函数的地址.

```c
int callback(struct dl_phdr_info *info, size_t size, void *data) {
  // 判断当前动态库是否是 `./lib.so` 的动态库的信息, 不如不是则跳过
  if (strcmp("./lib.so", info->dlpi_name) != 0) {
    return 0;
  }

  // 待从 .dynamic 段中查找出的信息
  int pltrelsz = 0; // .rela.plt 段大小
  char *dynstr = NULL; // .dynstr 段起始地址
  ElfW(Rela) *rela_plt = NULL; // .rela.plt 段起始地址
  ElfW(Sym) *dynsym = NULL; // .dynsym 段起始地址

  // 遍历当前动态库中所有段的信息
  for (size_t i = 0; i < info->dlpi_phnum; i++) {
    const ElfW(Phdr) *phdr = &info->dlpi_phdr[i];

    // 如果不是 .dynamic 段则跳过
    if (phdr->p_type != PT_DYNAMIC) {
      continue;
    }

    int dynEntryCount = phdr->p_memsz / sizeof(ElfW(Dyn));
    ElfW(Dyn) *dyn = (ElfW(Dyn) *)(phdr->p_offset + info->dlpi_addr);

    // 遍历获取 .dynamic 段中的信息
    for (int j = 0; j < dynEntryCount; j++) {
      ElfW(Dyn) *entry = &dyn[j];
      switch (dyn->d_tag) {
      case DT_PLTRELSZ: {
        pltrelsz = dyn->d_un.d_val;
        break;
      }
      case DT_JMPREL: {
        rela_plt = (ElfW(Rela) *)(info->dlpi_addr + dyn->d_un.d_ptr);
        break;
      }
      case DT_STRTAB: {
        dynstr = (char *)(info->dlpi_addr + dyn->d_un.d_ptr);
        break;
      }
      case DT_SYMTAB: {
        dynsym = (ElfW(Sym) *)(info->dlpi_addr + dyn->d_un.d_ptr);
        break;
      }
      }
      dyn++;
    }
  }

  // 遍历 .rela.plt 段中的信息
  int relaEntryCount = pltrelsz / sizeof(ElfW(Rela));
  for (int i = 0; i < relaEntryCount; i++) {
    ElfW(Rela) *entry = &rela_plt[i];

    // 获取 .dynsym 中的索引
    int r_sym = ELF64_R_SYM(entry->r_info);

    // 获取 .dynstr 中的索引
    int st_name = dynsym[r_sym].st_name;

    // 获取当前的符号名称
    char *name = &dynstr[st_name];

    // 如果不是 `puts` 符号, 则跳过
    if (strcmp("puts", name) != 0) {
      continue;
    }

    // 计算 `puts` 符号在 .got.plt 表中的位置
    uintptr_t hook_point = (uintptr_t)(info->dlpi_addr + entry->r_offset);

    // 修改 `puts` 符号的跳转地址为: my_printf
    *(void **)hook_point = my_printf;
  }

  return 0;
}


void hook() { dl_iterate_phdr(callback, NULL); }
```

## 最终效果

```bash
$ ./main
before printf: hello world
hello world
after printf: hello world
```

通过 `./main` 命令运行主程序, 将会得到下面的输出, 可以看到在 `hello world` 前后多了两行输出, 说明 lib.so 中的 `printf` 函数调用已经被替换成了 `my_printf`.

# 参考资料

- <程序员的自我修养 -- 链接, 装载与库>
- [Android PLT hook 概述](https://github.com/caikelun/caikelun.github.io/blob/master/site/blog/2018-05-01-android-plt-hook-overview.md)
- [字节跳动开源 Android PLT hook 方案 ByteHook](https://github.com/caikelun/caikelun.github.io/blob/master/site/blog/2021-08-19-bytedance-open-source-bytehook.md)
- [dl_iterate_phdr](https://man7.org/linux/man-pages/man3/dl_iterate_phdr.3.html)
- [Dynamic Section](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-42444.html)
- [Symbol Table Section
](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-79797.html#scrolltoc)
- [Relocation Sections](https://docs.oracle.com/cd/E23824_01/html/819-0690/chapter6-54839.html#scrolltoc)
