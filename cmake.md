# CMake Quick Start Guide

Modern C++ project setup with CMake 3.28+ and C++23

---

## 1. Traditional g++ Compilation

```bash
# Single file
g++ -std=c++23 -o hello hello.cpp

# Multiple files
g++ -std=c++23 -o app main.cpp utils.cpp -I./include

# With optimization and warnings
g++ -std=c++23 -Wall -Wextra -O2 -o app main.cpp utils.cpp -I./include
```

---

## 2. Basic CMake Equivalent

**CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(HelloWorld VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(hello hello.cpp)
```

**Build commands:**
```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/hello
```

---

## 3. Project with Headers and Sources

**Project structure:**
```
project/
├── CMakeLists.txt
├── include/
│   └── math_utils.h
└── src/
    ├── main.cpp
    └── math_utils.cpp
```

**include/math_utils.h:**
```cpp
#pragma once

namespace math {
    int add(int a, int b);
    int multiply(int a, int b);
}
```

**src/math_utils.cpp:**
```cpp
#include "math_utils.h"

namespace math {
    int add(int a, int b) { return a + b; }
    int multiply(int a, int b) { return a * b; }
}
```

**src/main.cpp:**
```cpp
#include <iostream>
#include "math_utils.h"

int main() {
    std::cout << "5 + 3 = " << math::add(5, 3) << '\n';
    std::cout << "5 * 3 = " << math::multiply(5, 3) << '\n';
    return 0;
}
```

**CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(MathApp VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(math_app
    src/main.cpp
    src/math_utils.cpp
)

target_include_directories(math_app PRIVATE include)

# Enable warnings
target_compile_options(math_app PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -pedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)
```

---

## 4. Static Library + Application

**Project structure:**
```
project/
├── CMakeLists.txt
├── lib/
│   ├── CMakeLists.txt
│   ├── include/
│   │   └── mylib/
│   │       └── string_utils.h
│   └── src/
│       └── string_utils.cpp
└── app/
    ├── CMakeLists.txt
    └── main.cpp
```

**lib/include/mylib/string_utils.h:**
```cpp
#pragma once
#include <string>

namespace mylib {
    std::string to_upper(const std::string& str);
    std::string reverse(const std::string& str);
}
```

**lib/src/string_utils.cpp:**
```cpp
#include "mylib/string_utils.h"
#include <algorithm>

namespace mylib {
    std::string to_upper(const std::string& str) {
        std::string result = str;
        std::ranges::transform(result, result.begin(), ::toupper);
        return result;
    }

    std::string reverse(const std::string& str) {
        return std::string(str.rbegin(), str.rend());
    }
}
```

**app/main.cpp:**
```cpp
#include <iostream>
#include "mylib/string_utils.h"

int main() {
    std::string text = "hello";
    std::cout << mylib::to_upper(text) << '\n';
    std::cout << mylib::reverse(text) << '\n';
    return 0;
}
```

**Root CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(MyProject VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_subdirectory(lib)
add_subdirectory(app)
```

**lib/CMakeLists.txt:**
```cmake
add_library(mylib STATIC
    src/string_utils.cpp
)

target_include_directories(mylib PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

target_compile_features(mylib PUBLIC cxx_std_23)
```

**app/CMakeLists.txt:**
```cmake
add_executable(myapp main.cpp)

target_link_libraries(myapp PRIVATE mylib)
```

---

## 5. Header-Only Library

**Project structure:**
```
project/
├── CMakeLists.txt
├── vendor/
│   └── json.hpp  # single-header library
├── include/
│   └── config.h
└── src/
    └── main.cpp
```

**include/config.h:**
```cpp
#pragma once
#include "json.hpp"

using json = nlohmann::json;

json load_config(const std::string& path);
```

**CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(ConfigApp VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Header-only library interface
add_library(json_lib INTERFACE)
target_include_directories(json_lib INTERFACE vendor)

add_executable(config_app
    src/main.cpp
    src/config.cpp
)

target_include_directories(config_app PRIVATE include)
target_link_libraries(config_app PRIVATE json_lib)
```

---

## 6. With Unit Tests

**Project structure:**
```
project/
├── CMakeLists.txt
├── src/
│   ├── calculator.h
│   └── calculator.cpp
├── app/
│   └── main.cpp
└── tests/
    ├── CMakeLists.txt
    └── test_calculator.cpp
```

**src/calculator.h:**
```cpp
#pragma once

class Calculator {
public:
    static int add(int a, int b);
    static int subtract(int a, int b);
    static double divide(double a, double b);
};
```

**tests/test_calculator.cpp:**
```cpp
#include <catch2/catch_test_macros.hpp>
#include "calculator.h"

TEST_CASE("Addition works", "[calculator]") {
    REQUIRE(Calculator::add(2, 3) == 5);
    REQUIRE(Calculator::add(-1, 1) == 0);
}

TEST_CASE("Division works", "[calculator]") {
    REQUIRE(Calculator::divide(10.0, 2.0) == 5.0);
}

TEST_CASE("Division by zero throws", "[calculator]") {
    REQUIRE_THROWS(Calculator::divide(10.0, 0.0));
}
```

**Root CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(CalculatorProject VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Library
add_library(calculator STATIC
    src/calculator.cpp
)
target_include_directories(calculator PUBLIC src)

# Application
add_executable(calc_app app/main.cpp)
target_link_libraries(calc_app PRIVATE calculator)

# Tests
option(BUILD_TESTS "Build tests" ON)
if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

**tests/CMakeLists.txt:**
```cmake
include(FetchContent)

FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v3.5.0
)
FetchContent_MakeAvailable(Catch2)

add_executable(tests
    test_calculator.cpp
)

target_link_libraries(tests PRIVATE 
    calculator
    Catch2::Catch2WithMain
)

include(CTest)
include(Catch)
catch_discover_tests(tests)
```

**Run tests:**
```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure
```

---

## 7. External Dependencies (FetchContent)

**CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.28)
project(ModernApp VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

# spdlog for logging
FetchContent_Declare(
    spdlog
    GIT_REPOSITORY https://github.com/gabime/spdlog.git
    GIT_TAG v1.13.0
)

# Catch2 for testing
FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v3.5.0
)

# fmt for formatting (if not using spdlog's bundled fmt)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
)

FetchContent_MakeAvailable(spdlog Catch2 fmt)

# Main application
add_executable(myapp
    src/main.cpp
    src/processor.cpp
)

target_include_directories(myapp PRIVATE include)

target_link_libraries(myapp PRIVATE
    spdlog::spdlog
    fmt::fmt
)

# Tests
enable_testing()
add_executable(tests
    tests/test_processor.cpp
)

target_include_directories(tests PRIVATE include)

target_link_libraries(tests PRIVATE
    Catch2::Catch2WithMain
    spdlog::spdlog
)

include(CTest)
include(Catch)
catch_discover_tests(tests)
```

**src/main.cpp:**
```cpp
#include <spdlog/spdlog.h>
#include <fmt/core.h>

int main() {
    spdlog::info("Application started");
    
    auto message = fmt::format("Hello from C++{}", 23);
    spdlog::info(message);
    
    spdlog::warn("This is a warning");
    spdlog::error("This is an error");
    
    return 0;
}
```

**Alternative: Using CPM (CMake Package Manager):**
```cmake
include(cmake/CPM.cmake)

CPMAddPackage("gh:gabime/spdlog@1.13.0")
CPMAddPackage("gh:catchorg/Catch2@3.5.0")
CPMAddPackage("gh:fmtlib/fmt#10.2.1")
```

---

## Quick Reference

**Configure:**
```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
```

**Build:**
```bash
cmake --build build -j$(nproc)
```

**Test:**
```bash
ctest --test-dir build --output-on-failure
```

**Clean:**
```bash
cmake --build build --target clean
# or
rm -rf build
```

**Install:**
```bash
cmake --install build --prefix /usr/local
```
