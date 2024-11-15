---
title: "构建自己的现代 C++ 工作流程"
summary: "讨论如何基于 CMake + Conan2 构建自己的 C++ 工作流程，包括构建、依赖管理、测试"
author: ["wivl"]
date: 2024-11-13
tags: ["cpp"]
weight: 1
ShowToc: true
draft: true
---

> 未完成

## 前言

与其他 "modern" 语言相比，cpp 是一个混乱的语言：不同平台的编译器的实现不同；不同版本 cpp 标准之间的差距大；缺少官方工具链。尤其是在接触了 rust 之后，我意识到有一个官方的、好用的项目管理器可以有多舒服。而这些正是 cpp 缺少的东西。cpp 标准并没有规定项目的结构。这就导致开发者往往以自己的喜好来选择工具链和组织自己的项目。一方面构建这些项目需要用不同的方式让人十分恼火，另一方面，像我这样的初学者在尝试开发自己的 cpp 项目时，在选择自己的工作流程方面容易感到迷茫。我尝试找到一种合理的工作流程，于是有了这篇文章。

我将在这篇文章中讨论如何基于 CMake，充分利用 CMakePresets，配合 vcpkg，构建现代的、属于自己的工作流。

## 为什么选择 CMake

在讨论为什么选择 CMake 之前，先让我们看看市面上的其他类似工具。

xmake 是由中国开发者开发的构建工具。它使用 Lua 作为其构建脚本（build scripts）的描述语言。xmake 附带一个包管理器 xrepo。xmake 也可以作为项目管理器。基本上可以认为 xmake = 项目管理器 + 构建工具 + 包管理器，相当于属于 cpp 自己的 cargo。我曾经有一段时间一直在使用 xmake，在终端 + nvim 的工作流程里体验很不错。但是虽然 xmake 在大部分编辑器里都有插件支持，在其他编辑器/IDE 里用 xmake 的体验还是很割裂，第三方工具对 xmake 的支持并不好。

premake5 似乎在国外开发者社区里具有一定的流行度，它同样使用 Lua 作为构建脚本描述语言，提供了类似 CMake 的生成多平台构建配置的功能。premake 的构建脚本要比 CMakeLists 简洁得多，但是它支持的平台类型数量远不如 CMake。

市面上还有其他体量足够大的 cpp 构建工具，如 Meson 和 Bazel。它们都在社区中足够受欢迎，并且相较于 CMake 都有自己的优势。对于这些构建工具还请读者自己去探索，我不想把所有的构建工具都列出来评价，因为其中有一些构建工具我从来没有上手使用过。

老实说，这些构建工具相对于 CMake 都多多少少有一定的可取之处。既然这样，我为什么要使用 CMake？理由很简单：因为大多数 cpp 开发者都在使用 CMake。当你在 Github 里浏览的 cpp 项目，你会发现无论是库还是可执行文件，它们很有可能使用的就是 CMake。换言之 CMake 具有最庞大的社区，当所有人都在使用 CMake 时，我们最好也使用 CMake，以获得最好的社区支持。

但同时我们不得不承认 CMake 存在一些问题。首先和 cpp 一脉相承的是，CMake 是一个具有历史底蕴的构建工具，不同的项目，哪怕都使用 CMake 作为构建工具，它们的实现方法和项目文件的组织方式可能完全不同。我试图在这些项目的组织方法中找到一种方便的、现代的方式来管理源文件、第三方依赖和测试，建立自己的工作流。

## 项目架构

假设我们要创建一个光线追踪渲染器。渲染器是一个命令行工具，实现了从命令行读取场景文件并渲染为图片的功能。看起来这是一个中型的项目，所以我们最好在开始写代码之前就考虑好项目的结构。一种可行的办法是：我首先实现渲染器的各种功能，例如读取场景文件、导入模型、渲染和输出最后的图片等，然后将这些功能（实际上是一系列的类和函数）封装成库。然后再使用这些库，写一个命令行工具。

类似这样的架构我把它称为 Core-App Architecture，即把重要的功能实现在一个库里，然后 app 基于这个库实现。我们的项目遵循这样的架构，适用于中/大型的项目。项目的树结构如下：

```
├─app
├─cmake
└─core
    └─module1
        ├─include
        │  └─module1
        └─src
            └─module1
```

1. `app` 文件夹存放应用程序的源文件

2. `core` 文件夹存放库的头文件和源文件，其中库又可以有多个，在这里命名为 `module`。以一个 module 为例，module 分为 `include` 和 `src` 文件夹。分别存放头文件和源文件，头文件和源文件在两个文件夹的结构应该一致。注意这里的两个文件夹分别都套上了一个和 module 名字相同的文件夹 `module1`，这是为了在当 include 这个 module 头文件的时候，允许包含模块名 module1：

```cpp
#include <module1/header.hpp>
```

否则，include 语句应该这样：

```cpp
#include <header.hpp>
```

这样写增加了代码的可读性。

3. `cmake` 文件夹存放了项目可能需要的 `.cmake` 文件，例如后面要用到的 conan 相关的脚本。

值得注意的是，据我观察有许多项目并不会把 app 和 core 文件夹放在项目根目录，而是在根目录创建一个 src 文件夹，然后把 app 和 core 文件夹放在 src 文件夹里。我个人喜好不遵循这样的模式，因为我觉得这样做没有什么好处，只是增加了文件夹的深度，并且需要一个额外的 `CMakeLists.txt`。

## 编写 CMakeLists

## 拥抱 cmake-presets

### CI

CMake 3.19 引入了 cmake-presets。cmake-presets 解决了使用 CMake 命令时的参数传递问题，允许指定常用配置选项并于他人共享。开发者可以在项目里维护两个 json 文件：CMakePresets.json（全局、共享） 和 CMakeUserPresets.json（本地）


## 使用 Conan2 管理第三方依赖


## 添加测试

## 将工作流程适配到编辑器


## References

[1][How to start a modern C++ project - Mikhail Svetkin - Meeting C++ 2023](https://www.youtube.com/watch?v=UI_QayAb9U0)

[2][CMake and Conan: past, present and future - Diego Rodriguez-Losada - Meeting C++ 2023](https://www.youtube.com/watch?v=s0q6s5XzIrA)

[3][How to Properly Setup C++ Projects](https://www.youtube.com/watch?v=5glH8dGoeCA)

[4][cmake-conan GitHub repository](https://github.com/conan-io/cmake-conan)

[5][cmake-presets Document](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)

[6][How to Use Modern CMake for an App + Lib Project](https://rvarago.github.io/2018-08-20-how-to-use-modern-cmake-for-an-app-p-lib-project/)