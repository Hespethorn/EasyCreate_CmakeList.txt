## 导言

在 Linux 下用 CMake 管理 C++ 项目时，你是否常遇到这些问题：手动写 CMakeLists.txt 繁琐、旧配置文件干扰编译、未使用参数警告刷屏、链接库不知从何下手？本文将带你打造一个「全能型 CMake 自动生成模板」，一键解决上述所有痛点，让项目构建效率翻倍。

## 一、模板核心功能清单

先看这个模板能帮我们做什么，避免重复造轮子：

-  **自动清理旧文件**：运行时自动删除 CMake 缓存、旧 Makefile 等冗余文件，杜绝配置冲突

-  **智能扫描源码**：递归识别当前目录及子目录下所有.cpp/.cc/.h/.hpp文件，无需手动列文件

-  **零警告编译**：默认抑制unused parameter（未使用参数）警告，同时保留关键编译检查

-  **规范输出目录**：可执行文件、库文件分别输出到build/bin和build/lib，源码目录不污染源

-  **灵活链接库**：预留动态库（.so）和静态库（.a）链接区域，示例清晰

-  **安全项目命名**：避免中文 / 特殊字符目录名导致的编译错误，支持手动自定义项目名

## 二、手把手实现模板脚本

整个模板的核心是一个 Bash 脚本（命名为create_cmake），我们分模块拆解实现逻辑，最后整合为完整脚本。

### 模块 1：自动清理旧 CMake 文件

每次生成新配置前，必须清理旧文件（比如CMakeCache.txt、CMakeFiles目录），否则残留配置会导致各种奇怪错误。

清理逻辑代码：

```
# 第一步：清理当前目录的CMake相关文件（避免旧配置干扰）
echo "🗑️ 正在清理CMake相关文件..."
# 删除核心配置文件
rm -f CMakeLists.txt
rm -f CMakeCache.txt
rm -f cmake_install.cmake
rm -f Makefile
# 删除CMake临时目录和构建目录
rm -rf CMakeFiles
rm -rf build
echo "✅ 清理完成，准备生成新配置"
```

### 模块 2：生成 CMakeLists.txt 核心配置

这部分是模板的灵魂，我们按 CMake 执行流程拆解关键配置：

#### 2.1 基础配置：指定版本、项目名、C++ 标准

```
# CMake最低版本要求（兼容大多数Linux发行版，如Ubuntu 20.04默认3.16）
cmake_minimum_required(VERSION 3.16)

# 项目名称：用安全默认名（避免中文目录问题），可手动修改为自定义名称
set(PROJECT_NAME "my_project")
project(${PROJECT_NAME} LANGUAGES CXX)  # 仅支持C++（如需C可加C）

# 设置C++标准：默认C++17（根据需求可改为11/14/20）
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)  # 强制启用指定标准，不降级
```

#### 2.2 自动扫描源码文件

用file(GLOB_RECURSE)递归扫描所有源文件，同时排除CMakeFiles/build/.git等非源码目录：

```
# 递归查找所有C++源文件（.cpp和.cc，覆盖常见后缀）
file(GLOB_RECURSE SOURCE_FILES
    ${PROJECT_SOURCE_DIR}/*.cpp
    ${PROJECT_SOURCE_DIR}/*.cc
)
# 排除非源码目录（关键！避免CMake自动生成的测试文件被误判）
list(FILTER SOURCE_FILES EXCLUDE REGEX ".*/CMakeFiles/.*")
list(FILTER SOURCE_FILES EXCLUDE REGEX ".*/build/.*")
list(FILTER SOURCE_FILES EXCLUDE REGEX ".*/.git/.*")

# 递归查找所有头文件（.h和.hpp）
file(GLOB_RECURSE HEADER_FILES
    ${PROJECT_SOURCE_DIR}/*.h
    ${PROJECT_SOURCE_DIR}/*.hpp
)
# 同样排除非源码目录
list(FILTER HEADER_FILES EXCLUDE REGEX ".*/CMakeFiles/.*")
list(FILTER HEADER_FILES EXCLUDE REGEX ".*/build/.*")
list(FILTER HEADER_FILES EXCLUDE REGEX ".*/.git/.*")
```

#### 2.3 自动添加头文件目录

提取所有头文件所在目录并去重，避免手动写include_directories：

```
# 遍历所有头文件，提取它们的目录
foreach(HEADER ${HEADER_FILES})
    get_filename_component(HEADER_DIR ${HEADER} DIRECTORY)
    list(APPEND INCLUDE_DIRS ${HEADER_DIR})
endforeach()
# 去重（避免重复添加同一目录）
list(REMOVE_DUPLICATES INCLUDE_DIRS)
# 将所有头文件目录添加到编译路径
include_directories(${INCLUDE_DIRS})
```

#### 2.4 编译配置：零警告 + 可执行文件生成

核心是抑制unused parameter警告（比如main函数的argc/argv未使用），同时保留关键警告：

```
# 生成可执行文件（仅当存在源文件时才生成，避免报错）
if(SOURCE_FILES)
    add_executable(${PROJECT_NAME} ${SOURCE_FILES} ${HEADER_FILES})
    
    # 编译警告配置：关键！抑制未使用参数警告，保留其他有用警告
    if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        target_compile_options(${PROJECT_NAME} PRIVATE 
            -Wall          # 基础警告（如未定义变量）
            -Wextra        # 额外警告（如未使用变量）
            -Wpedantic     # 严格遵循C++标准警告
            -Werror=return-type  # 将返回值错误视为编译错误（强制规范）
            -Wno-unused-parameter  # 抑制未使用参数警告（解决main函数argc/argv警告）
        )
    endif()

# 无源码时提示警告（避免CMake配置失败）
else()
    message(WARNING "⚠️ 未找到任何.cpp或.cc源文件，生成的项目可能无法编译")
endif()
```

#### 2.5 预留库链接区域

为动态库（.so）和静态库（.a）预留链接位置，附带示例：

```
# -------------------------- 库链接区域 --------------------------
# 在此处添加需要链接的动态库或静态库，取消注释并修改路径即可
# 1. 链接系统动态库（如线程库pthread）
# target_link_libraries(${PROJECT_NAME} PRIVATE pthread)
# 2. 链接自定义动态库（指定.so文件路径）
# target_link_libraries(${PROJECT_NAME} PRIVATE /home/yourname/libs/mylib.so)
# 3. 链接静态库（指定.a文件路径）
# target_link_libraries(${PROJECT_NAME} PRIVATE /home/yourname/libs/utils.a)
# 4. 多库混合链接（先指定库目录，再链接库名）
# link_directories(/home/yourname/libs)  # 库文件所在目录
# target_link_libraries(${PROJECT_NAME} PRIVATE mylib utils)  # 库名（省略lib前缀和.so/.a）
# -------------------------------------------------------------------
```

#### 2.6 扫描结果提示

方便查看 CMake 扫描到的文件和目录，排查问题：

```
# 显示扫描结果（构建时在终端打印，方便确认）
message(STATUS "📁 项目目录: ${PROJECT_SOURCE_DIR}")
message(STATUS "🔍 找到源文件数量: ${CMAKE_ARGC}")
message(STATUS "🔍 找到头文件目录: ${INCLUDE_DIRS}")
```

### 模块 3：整合为完整 Bash 脚本

将上述所有逻辑整合，加上脚本执行提示，最终脚本如下：

```
#!/bin/bash

# 第一步：清理当前目录的CMake相关文件（避免旧配置干扰）
echo "🗑️ 正在清理CMake相关文件..."
rm -f CMakeLists.txt
rm -f CMakeCache.txt
rm -f cmake_install.cmake
rm -f Makefile
rm -rf CMakeFiles
rm -rf build
echo "✅ 清理完成，准备生成新配置"

# 第二步：生成完整的CMakeLists.txt模板
cat > CMakeLists.txt << EOF
cmake_minimum_required(VERSION 3.16)

# 项目名称（可手动修改为自定义名称，避免中文/特殊字符）
set(PROJECT_NAME "my_project")
project(\${PROJECT_NAME} LANGUAGES CXX)

# 设置C++标准（根据需求修改：11/14/17/20）
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 输出目录配置（统一管理编译产物，不污染源码）
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY \${CMAKE_BINARY_DIR}/bin)  # 可执行文件→build/bin
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY \${CMAKE_BINARY_DIR}/lib)  # 动态库→build/lib
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY \${CMAKE_BINARY_DIR}/lib)  # 静态库→build/lib

# -------------------------- 自动扫描文件 --------------------------
# 递归查找所有C++源文件（.cpp和.cc）
file(GLOB_RECURSE SOURCE_FILES
    \${PROJECT_SOURCE_DIR}/*.cpp
    \${PROJECT_SOURCE_DIR}/*.cc
)
# 排除非源码目录（关键！避免CMake临时文件干扰）
list(FILTER SOURCE_FILES EXCLUDE REGEX ".*/CMakeFiles/.*")
list(FILTER SOURCE_FILES EXCLUDE REGEX ".*/build/.*")
list(FILTER SOURCE_FILES EXCLUDE REGEX ".*/.git/.*")

# 递归查找所有头文件（.h和.hpp）
file(GLOB_RECURSE HEADER_FILES
    \${PROJECT_SOURCE_DIR}/*.h
    \${PROJECT_SOURCE_DIR}/*.hpp
)
# 排除非源码目录
list(FILTER HEADER_FILES EXCLUDE REGEX ".*/CMakeFiles/.*")
list(FILTER HEADER_FILES EXCLUDE REGEX ".*/build/.*")
list(FILTER HEADER_FILES EXCLUDE REGEX ".*/.git/.*")

# 自动添加头文件目录（无需手动写include_directories）
foreach(HEADER \${HEADER_FILES})
    get_filename_component(HEADER_DIR \${HEADER} DIRECTORY)
    list(APPEND INCLUDE_DIRS \${HEADER_DIR})
endforeach()
list(REMOVE_DUPLICATES INCLUDE_DIRS)
include_directories(\${INCLUDE_DIRS})

# -------------------------- 构建配置 --------------------------
if(SOURCE_FILES)
    # 生成可执行文件（名称=项目名）
    add_executable(\${PROJECT_NAME} \${SOURCE_FILES} \${HEADER_FILES})
    
    # 编译警告：抑制未使用参数，保留关键检查
    if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        target_compile_options(\${PROJECT_NAME} PRIVATE 
            -Wall 
            -Wextra 
            -Wpedantic 
            -Werror=return-type
            -Wno-unused-parameter  # 解决main函数argc/argv警告
        )
    endif()

    # -------------------------- 库链接区域 --------------------------
    # 示例1：链接系统动态库（pthread）
    # target_link_libraries(\${PROJECT_NAME} PRIVATE pthread)
    # 示例2：链接自定义动态库
    # target_link_libraries(\${PROJECT_NAME} PRIVATE /path/to/your/lib.so)
    # 示例3：链接静态库
    # target_link_libraries(\${PROJECT_NAME} PRIVATE /path/to/your/lib.a)
    # -------------------------------------------------------------------

else()
    message(WARNING "⚠️ 未找到任何.cpp或.cc源文件，请检查项目目录")
endif()

# 显示扫描结果（方便排查问题）
message(STATUS "📁 项目目录: \${PROJECT_SOURCE_DIR}")
message(STATUS "🔍 找到源文件数量: \${CMAKE_ARGC}")
message(STATUS "🔍 找到头文件目录: \${INCLUDE_DIRS}")
EOF

# 第三步：提示用户后续操作
echo "✅ 成功生成CMakeLists.txt！"
echo "💡 下一步操作指南："
echo "1. （可选）修改项目名：编辑CMakeLists.txt中的PROJECT_NAME变量"
echo "2. （可选）链接库：在「库链接区域」取消注释并修改路径"
echo "3. 编译项目：mkdir -p build && cd build && cmake .. && make"
echo "4. 运行程序：./build/bin/my_project（或自定义项目名）"
```

## 三、模板安装与使用教程

### 3.1 安装脚本（全局可用）

1. 将上述完整脚本保存为create_cmake文件，放在/usr/local/bin目录（全局可执行）：

```
sudo nano /usr/local/bin/create_cmake
```

1. 粘贴脚本内容后，按Ctrl+O保存，Ctrl+X退出。

1. 赋予脚本执行权限：

```
sudo chmod +x /usr/local/bin/create_cmake
```

### 3.2 实际项目使用演示

以一个简单 C++ 项目为例，目录结构如下（源码分散在子目录）：

```
2025年8月30日/
├── main.cpp          # 主函数
├── math/
│   ├── calculator.cpp  # 数学计算函数
│   └── calculator.h    # 头文件
└── utils/
    ├── log.cpp        # 日志函数
    └── log.h          # 头文件
```

#### 步骤 1：生成 CMake 配置

进入项目目录，执行create_cmake：

```
cd /home/hespethorn/day_by_day/2025年8月30日
create_cmake
```

此时会看到清理提示，随后生成CMakeLists.txt。

#### 步骤 2：（可选）链接库

如果需要链接pthread线程库，打开CMakeLists.txt，找到「库链接区域」，取消对应注释：

```
# 链接pthread动态库
target_link_libraries(${PROJECT_NAME} PRIVATE pthread)
```

#### 步骤 3：编译与运行

```
# 创建构建目录并进入（避免污染源码）
mkdir -p build && cd build
# 生成Makefile
cmake ..
# 编译（无警告刷屏）
make
# 运行程序（可执行文件在build/bin）
./bin/my_project
```

## 四、常见问题解决

### Q1：执行create_cmake提示权限不足？

A：确保脚本有执行权限，重新执行sudo chmod +x /usr/local/bin/create_cmake。

### Q2：编译时提示 “找不到动态库”？

A：如果是自定义动态库，需指定完整路径，或通过link_directories添加库目录，例如：

```
# 添加库目录
link_directories(/home/hespethorn/libs)
# 链接库（省略lib前缀和.so后缀）
target_link_libraries(${PROJECT_NAME} PRIVATE mylib)
```

### Q3：想修改 C++ 标准为 C++11？

A：打开CMakeLists.txt，将set(CMAKE_CXX_STANDARD 17)改为set(CMAKE_CXX_STANDARD 11)。

## 五、模板优势总结

对比手动写 CMakeLists.txt，这个模板的优势：

1. **效率提升**：5 秒生成配置，无需手动列文件、写头文件目录

1. **兼容性强**：支持中文目录、子目录源码、多种库链接场景

1. **零警告体验**：默认处理常见警告，编译输出更清爽

1. **新手友好**：预留示例注释，跟着改就能用，降低 CMake 学习成本


从此，Linux 下 C++ 项目构建不用再 “复制粘贴 CMake 配置”，一个create_cmake搞定所有基础工作，专注写代码即可！
