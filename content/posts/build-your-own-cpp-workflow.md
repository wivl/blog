---
title: "构建自己的现代 C++ 工作流程"
summary: "探讨如何基于 CMake + Conan2 构建高效的 C++ 工作流，包括构建和依赖管理。"
author: ["wivl"]
date: 2024-11-17
tags: ["cpp"]
weight: 1
ShowToc: true
draft: false
---

## 前言

与其他 "modern" 语言相比，C++ 是一个混乱的语言：不同平台的编译器实现不一致，不同版本的 C++ 标准差异显著，且缺少统一的官方工具链。尤其是在接触了 Rust 之后，我意识到有一个官方的、好用的项目管理器可以有多舒服。而这些正是所谓的 Modern C++ 缺少的东西。C++ 并没有什么标准规定项目的结构，这就导致开发者往往以自己的喜好来选择工具链和组织自己的项目。一方面构建这些项目需要用不同的方式让人十分恼火，另一方面，像我这样的初学者在尝试开发自己的 C++ 项目时，在选择自己的工作流程方面容易感到迷茫。我最近尝试找到一种合理的工作流程，于是有了这篇文章。

我将在这篇文章中讨论如何基于 CMake，充分利用 CMakePresets，配合 Conan2 包管理器，构建现代的、属于自己的工作流。

## 为什么选择 CMake

在讨论为什么选择 CMake 之前，先让我们看看市面上的其他类似工具。

[xmake](https://xmake.io/) 是由中国开发者开发的构建工具。它使用 Lua 作为其构建脚本（build scripts）的描述语言。xmake 附带一个包管理器 [xrepo](https://xrepo.xmake.io/)。xmake 也可以作为项目管理器。基本上可以认为 **xmake = 项目管理器 + 构建工具 + 包管理器**，相当于属于 C++ 自己的 cargo。我曾经有一段时间一直在使用 xmake，在**终端 + nvim** 的工作流程里体验很不错。但是虽然 xmake 在大部分编辑器里都有插件支持，在其他编辑器/IDE 里用 xmake 的体验还是很割裂，第三方工具对 xmake 的支持并不好。

[premake5](https://premake.github.io/) 似乎在国外开发者社区里具有一定的流行度，它同样使用 Lua 作为构建脚本描述语言，提供了类似 CMake 的生成多平台构建配置的功能。premake 的构建脚本要比 CMakeLists 简洁得多，但是它支持的平台类型数量远不如 CMake。

市面上还有其他体量足够大的 C++ 构建工具，如 [Meson](https://mesonbuild.com/) 和 [Bazel](https://bazel.build/)。它们都在社区中足够受欢迎，并且相较于 CMake 都有自己的优势。对于这些构建工具还请读者自己去探索，我不想把所有的构建工具都列出来评价，因为其中有一些构建工具我从来没有上手使用过。

老实说，这些构建工具相对于 CMake 都多多少少有一定的可取之处。既然这样，我为什么要使用 CMake？理由很简单：因为大多数 C++ 开发者都在使用 CMake。当你在 Github 里浏览的 C++ 项目，你会发现无论是库还是可执行文件，它们很有可能使用的就是 CMake。换言之 CMake 具有最庞大的社区，当所有人都在使用 CMake 时，我们最好也使用 CMake，以获得最好的社区支持。

但同时我们不得不承认 CMake 存在一些问题。首先和 C++ 一脉相承的是，CMake 是一个具有历史底蕴的构建工具，不同的项目，哪怕都使用 CMake 作为构建工具，它们的实现方法和项目文件的组织方式可能完全不同。我试图在这些项目的组织方法中找到一种方便的、现代的方式来管理源文件、第三方依赖，建立自己的工作流。

## 项目架构

假设我们要创建一个光线追踪渲染器。这个渲染器是一个命令行工具，实现了从命令行读取场景文件，渲染场景并保存为图片的功能。看起来这是一个中型的项目，所以我们最好在开始写代码之前就考虑好项目的结构。一种可行的办法是：我先实现渲染器的各种功能，例如读取场景文件、导入模型、渲染和输出最后的图片等，然后将这些功能（实际上是一系列的类和函数）封装成库。然后再使用这些库，写一个命令行工具。

类似这样的架构我把它称为 `Core-App Architecture`，即把重要的功能实现在一个库里，然后 app 基于这个库实现。我们的项目遵循这样的架构，适用于中/大型的项目。项目的树结构如下：

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

每个文件夹的作用如下：

1. `app` 文件夹存放应用程序的源文件

2. `core` 文件夹存放实现的所有库，其中库又可以有多个，在这里称为 `module`。以 core 里只包含一个 module 为例（module1），module1 下有 `include` 和 `src` 子文件夹。分别存放头文件和源文件，头文件和源文件在两个文件夹的结构应该一致。注意这里的两个文件夹分别都套上了一个和 module 名字相同的文件夹 `module1`，这是为了在当 include 这个 module 头文件的时候，允许包含模块名 module1：

```cpp
#include <module1/header.hpp>
```

否则，include 语句应该这样：

```cpp
#include <header.hpp>
```

这样写增加了代码的可读性。

3. `cmake` 文件夹存放了项目可能需要的 `.cmake` 文件，例如后面要用到的 Conan 相关的脚本。

值得注意的是，据我观察有许多项目并不会把 app 和 core 文件夹放在项目根目录，而是在根目录创建一个 `src` 文件夹，然后把 `app` 和 `core` 文件夹放在 `src` 文件夹里。我个人喜好不遵循这样的模式，因为我觉得这样做没有什么好处，只是增加了文件夹的深度，并且需要一个额外的 `CMakeLists.txt`。

## 编写 CMakeLists.txt

我们首先给项目添加源文件。模板使用了一个很简单的例子：`app` 文件夹里只有一个 `main.cpp`，`core` 文件夹包含一个 `module1`，分别在 `include` 和 `src` 文件夹里创建 `header.hpp` 和 `source.cpp`。

现在我们给项目添加 `CMakeLists.txt`。具体需要添加的位置和项目现在的结构如下：

```
│  CMakeLists.txt
├─app
│      CMakeLists.txt
│      main.cpp
└─core
    │  CMakeLists.txt
    └─module1
        │  CMakeLists.txt
        ├─include
        │  └─module1
        │          header.hpp
        └─src
            └─module1
                    source.cpp
```

### 根目录

我在项目里使用了多层文件夹，很容易想到用 `sub_directory()` 的方式来组织，在这里也不例外。我需要在根目录里创建一个 `CMakeLists.txt`，内容很简单:

```cmake
cmake_minimum_required(VERSION 3.25)
project(app-dev)

add_subdirectory(core)
add_subdirectory(app)
```

注意到我在这里将项目命名为 `app-dev`，之所以这个名字有一个 `-dev` 后缀，是因为一个项目除了源码，往往还包含测试、文档等内容。这些东西只有开发者在乎，而和用户没有关系。

### core 

对于 `core` 文件夹，首先我们需要在 `core` 的根目录添加一个 `CMakeLists.txt`。内容很简单，添加所有 module 为 subdirectory。并且由于在这个例子中 `core` 文件夹里只有一个 `module1`，所以整个 `CMakeLists.txt` 只有一行：

```cmake
# core/CMakeLists.txt
add_subdirectory(module1)
```

现在添加 `module1` 的 `CMakeLists.txt`：

```cmake
# core/module1/CMakeLists.txt
project(module1)

# 声明一个库，并指定源文件
add_library(${PROJECT_NAME}
    src/module1/source.cpp
)
# 给库取别名 core::module1 增加了可读性，稍后在 app 文件夹 CMakeLists.txt 体现出
add_library(core::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

# 设置库的头文件路径
target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)

target_compile_features(${PROJECT_NAME}
    PRIVATE
        cxx_std_17
)
```

其中的这段代码可能会让初学者感到疑惑：

```cmake
# 设置库的头文件路径
target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)
```

`PUBLIC` 部分，声明了安装该库和构建该库的条件下，编译器查找头文件的文件夹为 `module1` 文件夹下的 `include` 文件夹。

`PRIVATE` 部分，允许 `module1` 内部从 `include` 文件夹作为根目录开始查找头文件。在这个例子中，`source.cpp` 实现了 `header.hpp` 定义的函数：

```cpp
// core/module1/include/module1/header.hpp
#pragma once

namespace module1 {
    int sum(int a, int b);
}


// core/module1/include/module1/source.cpp
#include "../../include/module1/header.hpp"

namespace module1 {
    int sum(int a, int b) {
        return a + b;
    }
}
```

注意 `source.cpp` 中的 `include` 语句。如果我们的 `CMakeLists.txt` 里包含 `PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}`，这个 include 语句可以这样写：

```cpp
#include <include/module1/header.hpp>
```

### app

当我们完成了 core 文件夹的所有 CMakeLists.txt 的编写，app 的 CMakeLists.txt 就很简单了。

```cmake
# app/CMakeLists.txt
project(app)

add_executable(${PROJECT_NAME}
    main.cpp
)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
    core::module1
)
```

这个 CMakeLists.txt 很简单，简单到没什么需要解释的。还记得我们在 `core/module1` 里的 CMakeLists.txt 给库取的别名吗，现在体现在 `target_link_libraries` 里了，这样用户就知道 module1 是在 core 里定义的库而不是其他地方。

## CMake Presets

### 为什么需要 CMake Presets

到这一步，我们已经完成了 CMakeLists.txt 的编写了，现在应该可以使用类似:

```
cmake -Bbuild .
cmake --build build
```
这样的命令构建项目。但是往往 CMake 命令不会这么简单。

假设我们使用的编辑器不是那么智能，我们可能需要更复杂的 CMake 命令生成 `compile_commands.json` 文件来让编辑器提供更好的辅助功能（这种情况 vim 用户可能经常遇到）。那么 CMake 命令可能会变成这样：

```
cmake -Bbuild . -DCMAKE_EXPORT_COMPILE_COMMANDS=1
```

或者你可能需要指定 CMake 生成指定平台的构建文件，那么你的 CMake 命令可能会变成这样：

```
cmake -Bbuild_arm64_iphones -GXcode -DCMAKE_BUILD_TYPE=Release \
	-DCONAN_PROFILE_BUILD=default \
  	-DCONAN_PROFILE_HOST=$(pwd)/.profiles/ios-arm64-iphoneos \
	-DCMAKE_SYSTEM_NAME=iOS \
  	-DCMAKE_OSX_DEPLOYMENT_TARGET=9.0 \
  	-DCMAKE_OSX_ARCHITECTURES=arm64
```

可以看到，有时候我们需要指定 CMake 生成构建文件时的 `cache variable`，使得 CMake 命令变得很长很难维护。这个问题有很多解决办法，CMake 提供了官方开箱即用的方法：CMake Presets。

CMake 3.19 引入了 CMake Presets。CMake Presets 解决了使用 CMake 命令时的参数传递问题，允许指定常用配置选项并于他人共享。开发者可以在项目里维护两个 json 文件：`CMakePresets.json`（全局、共享） 和 `CMakeUserPresets.json`（本地）。里面可以定义 `configure`、`build`、`test` 和 `package` 四个阶段的配置。并且提供了 `workflow`，开发者可以定义自己的工作流程。

### 一个简单的 CMakePresets.json

CMakePreset.json 里可以定义的内容很多，我不想再这里列举出很多写法，读者感兴趣可以自行了解。

本项目使用的 CMakePresets.json 相对简单，定义了 CMake 生成 Ninja 构建工具的配置文件，其他设置几乎使用默认：

```json
{
    "version": 6,
    "cmakeMinimumRequired": {
        "major": 3,
        "minor": 25,
        "patch": 0
    },
    "configurePresets": [
        {
            "name": "default",
            "displayName": "Default Config",
            "description": "Default build using Ninja generator",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "generator": "Ninja",
            "cacheVariables": {
                "CMAKE_EXPORT_COMPILE_COMMANDS": true,
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_VERBOSE_MAKEFILE": "ON"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "windows",
            "displayName": "Windows x86_64 Default",
            "configurePreset": "default"
        },
        {
            "name": "darwin",
            "displayName": "macOS Default",
            "configurePreset": "default"
        },
        {
            "name": "linux",
            "displayName": "linux x86_64 Default",
            "configurePreset": "default"
        }
    ],
    "workflowPresets": [
        {
            "name": "windows",
            "steps": [
                {
                    "type": "configure",
                    "name": "default"
                },
                {
                    "type": "build",
                    "name": "windows"
                }
            ]
        }
    ]

}
```

#### Configure Presets

包含了多个配置预设对象。每个对象都定义了一组 CMake 配置选项。也就是说，configure presets 定义了 CMake 如何生成指定构建工具需要的配置文件。以上面的 configure presets 为例，定义了名为 `default` 的配置。指定了配置文件的生成目录，指定了构建工具为 Ninja，以及执行 CMake 命令时候的 cache variables。

```json
"configurePresets": [
    {
        "name": "default",
        "displayName": "Default Config",
        "description": "Default build using Ninja generator",
        "binaryDir": "${sourceDir}/build/${presetName}",
        "generator": "Ninja",
        "cacheVariables": {
            "CMAKE_EXPORT_COMPILE_COMMANDS": true,
            "CMAKE_BUILD_TYPE": "Debug",
            "CMAKE_VERBOSE_MAKEFILE": "ON"
        }
    }
]
```

然后执行

```
cmake --preset default
```

生成 Ninja 构建工具需要的配置文件。

#### Build Presets

Build presets 定义了 CMake 如何使用已经生成好的构建工具配置文件构建 target。如果在 configure 步骤指定 generator 为 Ninja，那么在这个阶段就是指定 Ninja 如何编译 target，如指定编译器、平台架构等。

在这个项目里其实在 build 阶段没有什么特殊需求，所以只是使用了 default configure，由 Ninja 自己决定如何构建 target。如果你需要更详细的 build preset 可以自行了解。

```json
"buildPresets": [
    {
        "name": "windows",
        "displayName": "Windows x86_64 Default",
        "configurePreset": "default"
    },
    {
        "name": "darwin",
        "displayName": "macOS Default",
        "configurePreset": "default"
    },
    {
        "name": "linux",
        "displayName": "linux x86_64 Default",
        "configurePreset": "default"
    }
]
```

这一步之后可以执行

```
cmake --build --preset windows
```

构建 target。

#### Workflow Presets

我们的项目不涉及到 package 阶段，test 也暂时先跳过。先让我们定义自己的 workflow。workflow 只是将上述的几个步骤组合起来，构建一个工作流，这样我们就不需要分开执行 configure、build 等步骤了。定义一个 workflow 简洁直观，只需要在 steps 字段里声明每一步使用哪一个 preset。

```json
"workflowPresets": [
    {
        "name": "windows",
        "steps": [
            {
                "type": "configure",
                "name": "default"
            },
            {
                "type": "build",
                "name": "windows"
            }
        ]
    }
]
```

现在我们可以使用 workflow 运行项目的整个构建流程:

```
cmake --workflow --preset windows
```


## 使用 Conan2 管理第三方依赖

Conan2 是一个受欢迎的 C++ 包管理器。Conan2.0 提供了简单好用的 CMake 支持。

### CMake 结合 Conan2

为什么我要强调 Conan2？因为 Conan2 相对于 Conan1.x 提供了更加方便的 CMake 工具链的结合方案，详细可以参考 [Conan 官方的 Meeting C++ 分享视频](https://www.youtube.com/watch?v=s0q6s5XzIrA)。这里使用的就是官方推荐的方案，具体步骤和解释如下。

1. 我们使用 Conan 官方的 [cmake-conan](https://github.com/conan-io/cmake-conan) 仓库里提供的工具。从仓库里下载 `conan_provider.cmake` 到项目根目录的 `cmake` 文件夹里。

2. 然后在根目录创建一个 `conanfile.txt` 文件，用来声明我们希望 Conan 安装的第三方库。`conanfile.txt` 的内容如下，沿用官方的例子，声明了 `fmt` 库：

```
[requires]
fmt/9.1.0

[layout]
cmake_layout

[generators]
CMakeDeps
```

3. 修改 `CMakeLists.txt` 和 `CMakePresets.json`。还记得我们在第一步从 `cmake-conan` 仓库拷贝过来的 `conan_provider.cmake` 文件吗。这是一个 CMake 模块文件，里面定义了 Conan 包管理器与 CMake 交互的各种宏和函数。简单来说，`conan_provider.cmake` 重新定义了 CMake 的 `find_package()` 函数。当用户在 `CMakeLists.txt` 里面使用 `find_package()` 函数的时候会调用 Conan 的实现。Conan 会在这里施一些“魔法”。使得开发者不需要显式执行 Conan 命令。要做到这一点，只需要定义 `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` 为 `conan_provider.cmake`。由于我们的项目使用了 CMakePresets，这个步骤就很方便。修改 `CMakePresets.json` 里的 `configurePresetsa` 为：

```json
"configurePresets": [
    {
        "name": "conan",
        "hidden": true,
        "cacheVariables": {
            "CMAKE_PROJECT_TOP_LEVEL_INCLUDES": "${sourceDir}/cmake/conan/conan_provider.cmake"
        }
    },
    {
        "name": "default",
        "displayName": "Default Config",
        "description": "Default build using Ninja generator",
        "inherits": [
            "conan"
        ],
        "binaryDir": "${sourceDir}/build/${presetName}",
        "generator": "Ninja",
        "cacheVariables": {
            "CMAKE_EXPORT_COMPILE_COMMANDS": true,
            "CMAKE_BUILD_TYPE": "Debug",
            "CMAKE_VERBOSE_MAKEFILE": "ON"
        }
    }
],
```

这里的实现方式是新建了名为 `conan` 的 configure preset，并且使 default 继承这个 preset。之后 Conan 包管理器就可以运行了。在这个例子中，我需要在 `app/main.cpp` 文件里用 `fmt` 库。我需要修改 `app/CMakeLists.txt` 只需添加一行 `find_package()` 然后 link `fmt` 库：

```cmake
project(app)

add_executable(${PROJECT_NAME}
    main.cpp
)

# 添加 find_package()
find_package(fmt REQUIRED)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
    core::module1
    fmt::fmt # link fmt 库
)
```

之后我们输入命令

```
cmake --workflow --preset windows
```

如果上面的设置无误的话每次运行 CMake 工作流应该都会同步第三方库。我们不需要执行任何一条 `conan install .` 命令！


## 总结

创建项目是开始一个项目的第一步，这篇文章也是我的博客的第一篇。研究这个课题花费了我将近一周的时间，收集了多方的资料，尝试了多种工具链，最后找到了适合自己的工具链。但是这篇博客其实还没有完成。我在创建 CMake 工作流的时候跳过了测试的部分，但是我在研究这个课题的时候经常可以听见那些有经验的开发者强调编写测试的重要性。但是很不幸的是，测试是我从来没有接触过的内容。我应该会在之后补充测试相关的内容。

在发布这篇文章的同时，我也将我在文章中使用的模板分享到 [GitHub](https://github.com/wivl/cpp-template) 上了，读者可以参考源码，或者直接使用这个模板。


## References

[1][How to start a modern C++ project - Mikhail Svetkin - Meeting C++ 2023](https://www.youtube.com/watch?v=UI_QayAb9U0)

[2][CMake and Conan: past, present and future - Diego Rodriguez-Losada - Meeting C++ 2023](https://www.youtube.com/watch?v=s0q6s5XzIrA)

[3][How to Properly Setup C++ Projects](https://www.youtube.com/watch?v=5glH8dGoeCA)

[4][cmake-conan GitHub repository](https://github.com/conan-io/cmake-conan)

[5][cmake-presets Document](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html)

[6][How to Use Modern CMake for an App + Lib Project](https://rvarago.github.io/2018-08-20-how-to-use-modern-cmake-for-an-app-p-lib-project/)