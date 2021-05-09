---
layout: post
title: 【操作指令】GNU开源项目总结
date: 2019-07-05
tags: 操作指令
---

# GNU开源项目编译总结

对于GNU开源项目，经常会看到几个核心文件：  

- README：包括基本的指引，一般需要先阅读这个文件。
- bootstrap（option）：一般是需要优先运行此脚本，做一些预处理操作，例如检测依赖的工具链是否完整、预先安装工具链、系统级的配置生成等，这个脚本，基本上意思是只执行一次，生成一些全局的东西。有一些bootstrap内部，是包含了configure的执行动作的，所以，也可能运行完这个之后，就无需继续执行configure了，当然，configure可以重复运行
- configure：用于生成基本配置，例如：c++编译选项等。如果没有bootstrap，可能它还承担了一些全局系统配置等工作。
- makefile：用于编译链接生成可执行文件
- 源代码：项目源码 
  
大部分情况，编译步骤如下：  

```bash
# 执行预处理操作
./bootstrap

# 新建临时目录
mkdir build
cd build

# 运行configure
../configure

# 编译和安装
make && make install
```

# CMakeLists.txt的编写

对于熟悉visual studio这套工具的开发而言，solution、project、C/C++编译设置（include目录、编译优化选项、使用的动态静态运行时库）、链接设置（lib目录、链接的lib、输出的）、工程源码等，会非常熟悉，也是一套软件研发所需要关注的基本概念。切换到Linux下开发，则通常会比较陌生，不太适应makefile等新的编译模式。因此，这里总结一下这俩到对应关系，以便后续开源项目的查看。  

## solution对比

### vs中的solution

vs中的solution（也就是xxx.sln文件），会包含各个project的位置，以及一些版本信息，如下：

- vs版本
- 包含的project
- solution编译平台（debug、release、32位、64位）
- 针对每个project的编译平台（是否编译debug、release、32位、64位）
- 项目之间的依赖编译

编译整个solution，就会针对所有的工程依次进行编译

### CMake中的solution

linux下的工程管理，是按照目录来管理的，对于solution，实际就是一个目录，包括一个CMakeLists.txt文件。对于每个工程而言，都是一个目录中包含CMakeLists.txt。  

![png](/images/post/gnu/solution.png)

其中包括都如下基本信息：
- cmake_minimum_required(VERSION 3.15)：包含最小使用都cmake版本，后面需要使用cmake来进行make文件生成
- project(solution)：当前工程名称。实际上，这里的solution本质也是一个project，只是它不包含实际代码，而是引用其他工程，等价于
- add_subdirectory(helloworld)：在这里的作用，类似增加project，其中的文件夹，就是一个工程文件夹，这个工程文件夹中必须包含CMakeLists.txt

为了能同时编译多个solution，只需要在solution顶层运行如下命令：

```bash
mkdir build
cd build

# 对于debug编译
mkdir debug
cd debug
cmake -DCMAKE_BUILD_TYPE=Debug ../../
cmake --build .
cd ..

# 对于release编译
mkdir release
cd release
cmake _DCMAKE_BUILD_TYPE=Release ../../
cmake --build .
```

## project对比

### vs中的project

vs中的project，包含几个工程文件，其中包括主工程文件.vcxproj、工程结构文件.vcxproj.filter、属性模板文件.props。  
对于cmake编译环境，上述介绍到的CMakeLists.txt，就是每个工程文件。按照属性对比，来依次看看他们的区别。  

这里工程常用的一些设置，包括：

- 常规 - 可执行文件输出目录、临时文件pdb等输出目录（设置一些全局变量outdir）
- 常规 - SDK版本
- 常规 - 编译平台工具集合
- 常规 - C++语言标准
- 高级 - 字符集（unicode还是别的）
- 调试 - 命令行及参数
- VC++目录 - 包含Windows SDK相关的库的头文件包含目录
- C/C++ - 常规 - 自身及第三方库文件包含目录
- C/C++ - 常规 - 警告等级
- C/C++ - 优化 - 优化等级
- C/C++ - 预处理器 - 一堆预处理的define定义
- C/C++ - 代码生成 - 运行时库是静态还是动态
- C/C++ - 高级 - 函数调用约定
- 链接器 - 常规 - 可执行文件输出全路径
- 链接器 - 常规 - 第三方库目录
- 链接器 - 输入 - 依赖的lib库
- 链接器 - 清单文件 - MANIFEST文件，包括UAC和默认是否管理员启动等
- 链接器 - 调试 - 是否生成pdb
- 链接器 - 系统 - 默认运行子系统
- 链接器 - 系统 - 堆和堆栈默认大小设置
- 生成事件 - 编译前、中、后执行脚本等

### CMake中的project

对于CMake项目，project就是包含CMakeLists.txt的所在目录。为了针对上述vs中的概念形成对比记忆，这里就用一整套工程来表述。  
相关命令解释：<https://cmake.org/cmake/help/v3.17/manual/cmake-commands.7.html>  

#### 常规写法

```bash
# helloworld/CMakeLists.txt

# 描述了cmake的最小版本3.10
cmake_minimum_required(VERSION 3.10)

# set the project name and version。基本没什么作用，后面的version，会生成全局变量：PROJECT_VERSION_MAJOR和PROJECT_VERSION_MINOR
project(HelloWorld VERSION 1.0)

# 指定C++标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# 输通过config.h.in，动态生成config.h。这个的主要目的，是通过预编译动态生成一些配置项目。在这里，可以使用一些CMake的全局变量，例如：PROJECT_BINARY_DIR、PROJECT_SOURCE_DIR等
configure_file(config.h.in config.h)

option(USE_MYLIB "Use My Lib Implement" ON)

if(USE_MYLIB)
    # MyLib本身是一个lib库，所以需要进行编译
    add_subdirectory(../MyLib libdir)

    # 两个列表，用于lib和额外的include，类似c++的lib库输入和c++的include
    list(APPEND EXTRA_LIBS MyLib)
    list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MyLib")
endif()

add_executable(helloworld main.cpp)

# 类似于vs中是c/c++的lib包含
target_link_libraries(helloworld PUBLIC ${EXTRA_LIBS})

# 类似于vs中的c/c++包含目录，主要用于头文件包含的搜索路径
target_include_directories(HelloWorld PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           ${EXTRA_INCLUDES}
                           )
```

```bash
# MyLib/CMakeLists.txt
add_library(MyLib mymath.cpp)
```

#### Lib库中添加include

当使用interface时，会写入interface_include_directories环境变量，外层使用target_link_libraries时，会主动扫描lib库自身当包含目录，这样外层引用lib库的主体，就不不再需要单独使用target_include_directories了。

```bash
# helloworld/CMakeLists.txt

# 描述了cmake的最小版本3.10
cmake_minimum_required(VERSION 3.10)

# set the project name and version。基本没什么作用，后面的version，会生成全局变量：PROJECT_VERSION_MAJOR和PROJECT_VERSION_MINOR
project(HelloWorld VERSION 1.0)

# 指定C++标准
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# 输通过config.h.in，动态生成config.h。这个的主要目的，是通过预编译动态生成一些配置项目。在这里，可以使用一些CMake的全局变量，例如：PROJECT_BINARY_DIR、PROJECT_SOURCE_DIR等
configure_file(config.h.in config.h)

option(USE_MYLIB "Use My Lib Implement" ON)

if(USE_MYLIB)
    # MyLib本身是一个lib库，所以需要进行编译
    add_subdirectory(../MyLib libdir)

    # 两个列表，用于lib和额外的include，类似c++的lib库输入和c++的include
    list(APPEND EXTRA_LIBS MyLib)
endif()

add_executable(helloworld main.cpp)

# 类似于vs中是c/c++的lib包含
target_link_libraries(helloworld PUBLIC ${EXTRA_LIBS})

# 类似于vs中的c/c++包含目录，主要用于头文件包含的搜索路径
target_include_directories(HelloWorld PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```

```bash
# MyLib/CMakeLists.txt

add_library(MyLib mymath.cpp)

# 其中，CMAKE_CURRENT_SOURCE_DIR表示的是MyLib当前的源码目录。使用INTERFACE就可以将该目录加入到interface_include_dir，任何想要add这个lib的，都会自动搜索这个变量，自动加入所有的目录到搜索路径。
target_include_directories(MyLib
        INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
        )
```

### vs中的概念与CMake中一一对应

- 常规 - 可执行文件输出目录、临时文件pdb等输出目录（设置一些全局变量outdir） ----> 如下：

```bash
# 针对 add_executable 类型的，使用 EXECUTABLE_OUTPUT_PATH
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/../bin)

# 针对 add_library 类型的，使用 LIBRARY_OUTPUT_PATH
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/../lib)
```

- 常规 - SDK版本 ----> 这个由于是平台相关的SDK版本，CMake本质是跨平台的（Make是平台相关的），所以是没有对应关系
- 常规 - 编译平台工具集合 ----> 这个相当于CMake版本，如下：

```bash
cmake_minimum_required(VERSION 3.15)
```

- 常规 - C++语言标准 ----> 如下：

```bash
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```

- 高级 - 字符集（unicode还是别的） ----> 如下：

```bash
# 只有UNICODE和其他类型，这个是为了让平台SDK使用UNICODE版本还是单字节字符串版本
add_definitions(-DUNICODE -D_UNICODE)
```

- 调试 - 命令行及参数  ----> 如下：
![png](/images/post/gnu/clion_debug.png)
- VC++目录 - 包含Windows SDK相关的库的头文件包含目录 ----> 本质应该是通过搜索路径，使用find_path，但是不确定
- C/C++ - 常规 - 自身及第三方库文件包含目录 ----> 如下：
```bash
# 向目标helloworld中添加路径，其中，PUBLIC表示添加路径后，允许自身和第三方引用方使用；PRIVATE表示添加自身路径仅允许自身使用；INTERFACE表示添加供他人使用
target_include_directories(helloworld
        PUBLIC "${PROJECT_BINARY_DIR}"
        )
```
- C/C++ - 常规 - 警告等级 ----> 实际使用的是gcc的[编译警告](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html)，如下：

```bash
# 两种方式设置，第一种是针对所有编译器的，包括c和c++：
if( CMAKE_BUILD_TYPE STREQUAL "Debug" )
    add_definitions(-Wall)
endif()

# 第二种，精细化，可以只包括C++
if( CMAKE_BUILD_TYPE STREQUAL "Debug" )
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -g -Wall -Wno-unused-variable -pthread")
elseif( CMAKE_BUILD_TYPE STREQUAL "Release" )
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -O2 -pthread -fopenmp")
endif()
```

- C/C++ - 优化 - 优化等级 ----> 实际使用的是gcc的[编译优化](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)，如下：

```bash
# 第一种，针对所有编译器而言：

# 关闭编译器优化
add_compile_options(-O0)

# 使用优化等级
add_compile_options(-Os)

# 第二种，精细化，可以只包括C++

# 关闭编译器优化，注意：-fno-elide-constructors这个选项，只针对C++
set(CMAKE_CXX_FLAGS "-fno-elide-constructors ${CMAKE_CXX_FLAGS}")
```

- C/C++ - 预处理器 - 一堆预处理的define定义  ----> 如下：

```bash
add_definition("-D宏")
```

- C/C++ - 代码生成 - 运行时库是静态还是动态 ----> 待定？？？
- C/C++ - 高级 - 函数调用约定
- 链接器 - 常规 - 第三方库目录 ----> 这里如果想调用第三方lib库，需要几个步骤：

```bash
# 添加库头文件包含目录
target_include_directories(helloworld
        PUBLIC "${PROJECT_SOURCE_DIR}/../MyLib"
        )

# 添加库依赖
target_link_libraries(helloworld PUBLIC MyLib.dylib)
```

- 链接器 - 输入 - 依赖的lib库 ----> 如下：

```bash
# 可以隐式指定，它会搜索MyLib.dylib、libMyLib.a
target_link_libraries(helloworld PUBLIC MyLib)

# 也可以显示指定是动态库还是静态库（根据名字）
target_link_libraries(helloworld PUBLIC MyLib.dylib)
```

- 链接器 - 清单文件 - MANIFEST文件，包括UAC和默认是否管理员启动等 ----> 使用的是chmod更改权限，主要分为root权限和其他权限
- 链接器 - 调试 - 是否生成pdb ----> gcc编译的程序，即使未开启优化生成了pdb，也是跟可执行程序一起的，如果需要调试就不太方便，猜测应该是在发布的时候，同时生成release（已优化）和debug（未优化）可执行程序文件，并将debug的可执行程序的符号表分离出来。参考<https://stackoverflow.com/questions/866721/how-to-generate-gcc-debug-symbol-outside-the-build-target>
- 链接器 - 系统 - 默认运行子系统 ----> Windows下，兼容console、win system、linux system系统，linux下没有这么复杂，就一套系统运行
- 链接器 - 系统 - 堆和堆栈默认大小设置 ----> gcc编译选项可以设置，参考<https://gcc.gnu.org/onlinedocs/gcc-9.3.0/gcc/Option-Summary.html#Option-Summary>中，搜索stack
- 生成事件 - 编译前、中、后执行脚本等 ----> 这里可以在CLion中debug配置中编译工具，默认配置了build，可以自定义前置添加执行命令，和后置执行命令

# 更多教程

CMake全命令，非常好的教程：<https://www.cnblogs.com/ZY-Dream/p/11232779.html>  
CMake输出控制：<https://www.cnblogs.com/tangxin-blog/p/8283460.html>