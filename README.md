# C++ Style Guide for Robotics Systems
## A Comprehensive Guide for Safe, Modern C++ Development

---

## Table of Contents

### Part I: Foundations
1. [Core Philosophy](#1-core-philosophy)
2. [Development Environment Setup](#2-development-environment-setup)
3. [Language Subset and Safety](#3-language-subset-and-safety)
4. [Error Handling with std::expected](#4-error-handling-with-stdexpected)
5. [Memory Management](#5-memory-management)

### Part II: Core Libraries
6. [Essential Libraries Overview](#6-essential-libraries-overview)
7. [String Formatting with fmt](#7-string-formatting-with-fmt)
8. [Logging with spdlog](#8-logging-with-spdlog)
9. [Testing with Catch2](#9-testing-with-catch2)
10. [Linear Algebra with Eigen](#10-linear-algebra-with-eigen)

### Part III: Concurrent Programming
11. [Threading Fundamentals](#11-threading-fundamentals)
12. [Thread Safety Patterns](#12-thread-safety-patterns)
13. [Lock-Free Programming](#13-lock-free-programming)
14. [Real-Time vs Non-Real-Time Code](#14-real-time-vs-non-real-time-code)

### Part IV: Communication
15. [Inter-Thread Communication](#15-inter-thread-communication)
16. [CycloneDDS for ROS2](#16-cyclonedds-for-ros2)
17. [Message Design and Serialization](#17-message-design-and-serialization)

### Part V: User Interface
18. [ImGui Fundamentals](#18-imgui-fundamentals)
19. [Real-Time Visualization](#19-real-time-visualization)

### Part VI: Production
20. [Build System and Tooling](#20-build-system-and-tooling)
21. [Static Analysis and Sanitizers](#21-static-analysis-and-sanitizers)
22. [Debugging and Profiling](#22-debugging-and-profiling)
23. [Complete Robot Example](#23-complete-robot-example)

### Appendices
- [A. Quick Reference](#appendix-a-quick-reference)
- [B. Common Pitfalls](#appendix-b-common-pitfalls)
- [C. Migration from Python](#appendix-c-migration-from-python)

---

## Part I: Foundations

## 1. Core Philosophy

### 1.1 Guiding Principles

Our approach to C++ prioritizes **safety**, **clarity**, and **performance** in that order:

1. **Make invalid states unrepresentable** - Use the type system to prevent errors
2. **Fail at compile time, not runtime** - Catch errors early
3. **Explicit is better than implicit** - No hidden behavior
4. **Use proven libraries** - Don't reinvent the wheel
5. **Separate RT from non-RT** - Clear boundaries for real-time code

### 1.2 The Modern C++ Subset

We use a carefully chosen subset of C++23 that provides safety without complexity:

```cpp
// ✅ EMBRACE these features
std::expected<T, E>      // Error handling without exceptions
std::unique_ptr<T>        // Single ownership
std::span<T>             // Safe array views
std::array<T, N>         // Fixed-size arrays
std::string_view         // Non-owning string views
[[nodiscard]]            // Enforce error checking
constexpr                // Compile-time computation
enum class               // Type-safe enums
std::thread              // Platform-independent threading
std::atomic<T>           // Lock-free primitives

// ❌ AVOID these features
throw/try/catch          // Use std::expected instead
new/delete               // Use smart pointers
std::shared_ptr          // Prefer single ownership
auto everywhere          // Be explicit about types
template metaprogramming // Keep templates simple
virtual inheritance      // Use composition
operator overloading     // Except for specific cases
```

### 1.3 Code Organization Philosophy

```
project/
├── src/
│   ├── core/           # Core functionality
│   ├── control/        # Control algorithms
│   ├── drivers/        # Hardware interfaces
│   ├── comm/           # Communication (DDS, etc.)
│   ├── ui/             # User interface
│   └── main.cpp
├── include/            # Public headers
├── tests/              # Unit and integration tests
├── tools/              # Build and analysis tools
├── docs/               # Documentation
└── third_party/        # External dependencies
```

---

## 2. Development Environment Setup

### 2.1 Required Tools

```bash
# Ubuntu 22.04/24.04 setup
sudo apt update
sudo apt install -y \
    build-essential \
    cmake ninja-build \
    clang-16 clang-tidy-16 \
    gcc-13 g++-13 \
    gdb valgrind \
    python3-pip \
    libfmt-dev \
    libspdlog-dev \
    libeigen3-dev \
    libimgui-dev

# Install additional tools
pip3 install conan cpplint

# Install Catch2
git clone https://github.com/catchorg/Catch2.git
cd Catch2
cmake -Bbuild -H. -DBUILD_TESTING=OFF
sudo cmake --build build/ --target install

# Install CycloneDDS
git clone https://github.com/eclipse-cyclonedds/cyclonedds.git
cd cyclonedds
cmake -Bbuild -H. -DCMAKE_INSTALL_PREFIX=/usr/local
sudo cmake --build build/ --target install
```

### 2.2 VS Code Configuration

```json
// .vscode/settings.json
{
    "C_Cpp.default.cppStandard": "c++23",
    "C_Cpp.default.intelliSenseMode": "linux-gcc-x64",
    "C_Cpp.clang_format_style": "file",
    "C_Cpp.codeAnalysis.runAutomatically": true,
    "C_Cpp.codeAnalysis.clangTidy.enabled": true,
    "cmake.configureSettings": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
    },
    "files.associations": {
        "*.h": "cpp",
        "*.hpp": "cpp"
    }
}
```

### 2.3 Clang-Format Configuration

```yaml
# .clang-format
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
PointerAlignment: Left
AlignAfterOpenBracket: Align
AllowShortFunctionsOnASingleLine: Empty
AlwaysBreakTemplateDeclarations: Yes
BinPackParameters: false
BreakBeforeBraces: Attach
IndentCaseLabels: true
SortIncludes: true
IncludeBlocks: Regroup
IncludeCategories:
  - Regex: '^<.*>'
    Priority: 1
  - Regex: '^".*"'
    Priority: 2
```

---

## 3. Language Subset and Safety

### 3.1 Feature Usage Guidelines

#### Memory Management
```cpp
// ✅ GOOD: Clear ownership with unique_ptr
class MotorController {
    std::unique_ptr<MotorDriver> driver;
public:
    explicit MotorController(int motor_id) 
        : driver(std::make_unique<MotorDriver>(motor_id)) {}
};

// ✅ GOOD: Observer pattern for non-owning references
class ControlLoop {
    MotorController* motor;  // Non-owning, must outlive ControlLoop
public:
    explicit ControlLoop(MotorController& m) : motor(&m) {}
};

// ❌ BAD: Shared ownership complicates lifetime
class BadDesign {
    std::shared_ptr<Resource> resource;  // Who owns this?
};
```

#### Type Safety
```cpp
// ✅ GOOD: Strong types prevent errors
enum class MotorID : uint8_t { 
    MOTOR_1 = 0, MOTOR_2, MOTOR_3, MOTOR_4, MOTOR_5, MOTOR_6 
};

enum class JointID : uint8_t { 
    BASE = 0, SHOULDER, ELBOW, WRIST_1, WRIST_2, WRIST_3 
};

// Can't accidentally mix motor and joint IDs
void move_motor(MotorID motor, float position);
void move_joint(JointID joint, float angle);

// ❌ BAD: Weak typing allows errors
void bad_move(int id, float value);  // Is id a motor or joint?
```

### 3.2 RAII and Resource Management

```cpp
// Every resource follows RAII pattern
class SerialPort {
    int fd = -1;
    
public:
    explicit SerialPort(const std::string& device) {
        fd = ::open(device.c_str(), O_RDWR | O_NOCTTY);
        if (fd < 0) {
            throw std::runtime_error("Failed to open " + device);
        }
    }
    
    ~SerialPort() {
        if (fd >= 0) {
            ::close(fd);
        }
    }
    
    // Delete copy, allow move
    SerialPort(const SerialPort&) = delete;
    SerialPort& operator=(const SerialPort&) = delete;
    SerialPort(SerialPort&& other) noexcept : fd(std::exchange(other.fd, -1)) {}
    SerialPort& operator=(SerialPort&& other) noexcept {
        if (this != &other) {
            if (fd >= 0) ::close(fd);
            fd = std::exchange(other.fd, -1);
        }
        return *this;
    }
};
```

---

## 4. Error Handling with std::expected

### 4.1 Error Code Design

```cpp
// error_codes.h
#pragma once
#include <expected>
#include <string_view>

namespace robot {

enum class ErrorCode : int32_t {
    // Success
    OK = 0,
    
    // Hardware errors (100-199)
    HW_NOT_CONNECTED = 100,
    HW_INIT_FAILED = 101,
    HW_COMM_TIMEOUT = 102,
    HW_INVALID_RESPONSE = 103,
    
    // System errors (200-299)
    SYS_NOT_INITIALIZED = 200,
    SYS_ALREADY_RUNNING = 201,
    SYS_INVALID_STATE = 202,
    SYS_RESOURCE_BUSY = 203,
    
    // Parameter errors (300-399)
    PARAM_OUT_OF_RANGE = 300,
    PARAM_INVALID = 301,
    PARAM_NULL = 302,
    
    // Critical errors (400+)
    CRITICAL_SAFETY_VIOLATION = 400,
    CRITICAL_EMERGENCY_STOP = 401,
};

constexpr std::string_view to_string(ErrorCode code) {
    switch (code) {
        case ErrorCode::OK: return "OK";
        case ErrorCode::HW_NOT_CONNECTED: return "Hardware not connected";
        case ErrorCode::HW_INIT_FAILED: return "Hardware initialization failed";
        // ... complete for all codes
        default: return "Unknown error";
    }
}

// Type aliases for cleaner code
template<typename T>
using Result = std::expected<T, ErrorCode>;

using Status = std::expected<void, ErrorCode>;

// Helper for creating success results
template<typename T>
Result<T> Ok(T&& value) {
    return Result<T>(std::forward<T>(value));
}

inline Status OkStatus() {
    return Status();
}

// Helper for creating error results
template<typename T>
Result<T> Err(ErrorCode code) {
    return std::unexpected(code);
}

inline Status ErrStatus(ErrorCode code) {
    return std::unexpected(code);
}

} // namespace robot
```

### 4.2 Error Handling Patterns

```cpp
// motor_controller.cpp
#include "error_codes.h"
#include <fmt/core.h>

namespace robot {

class MotorController {
    bool initialized = false;
    float position = 0.0f;
    float max_position = 6.28f;
    
public:
    [[nodiscard]] Status initialize() {
        if (initialized) {
            return ErrStatus(ErrorCode::SYS_ALREADY_RUNNING);
        }
        
        // Hardware initialization
        if (!hw_init()) {
            return ErrStatus(ErrorCode::HW_INIT_FAILED);
        }
        
        initialized = true;
        return OkStatus();
    }
    
    [[nodiscard]] Result<float> get_position() const {
        if (!initialized) {
            return Err<float>(ErrorCode::SYS_NOT_INITIALIZED);
        }
        return Ok(position);
    }
    
    [[nodiscard]] Status move_to(float target_position) {
        if (!initialized) {
            return ErrStatus(ErrorCode::SYS_NOT_INITIALIZED);
        }
        
        if (std::abs(target_position) > max_position) {
            return ErrStatus(ErrorCode::PARAM_OUT_OF_RANGE);
        }
        
        // Perform movement
        position = target_position;
        return OkStatus();
    }
};

// Using the error handling
void example_usage() {
    MotorController motor;
    
    // Method 1: Immediate checking
    if (auto status = motor.initialize(); !status) {
        fmt::print("Init failed: {}\n", to_string(status.error()));
        return;
    }
    
    // Method 2: Monadic chaining
    auto result = motor.get_position()
        .and_then([&motor](float pos) {
            return motor.move_to(pos + 0.1f);
        })
        .or_else([](ErrorCode err) {
            fmt::print("Operation failed: {}\n", to_string(err));
            return ErrStatus(err);
        });
    
    // Method 3: Value or default
    float current_pos = motor.get_position().value_or(0.0f);
    
    // Method 4: Pattern matching on error
    if (auto move_result = motor.move_to(10.0f); !move_result) {
        switch (move_result.error()) {
            case ErrorCode::PARAM_OUT_OF_RANGE:
                fmt::print("Position {} out of range\n", 10.0f);
                break;
            case ErrorCode::SYS_NOT_INITIALIZED:
                fmt::print("Motor not initialized\n");
                break;
            default:
                fmt::print("Unexpected error: {}\n", 
                         to_string(move_result.error()));
        }
    }
}

} // namespace robot
```

---

## 5. Memory Management

### 5.1 Ownership Patterns

```cpp
// ownership_patterns.h
#pragma once
#include <memory>
#include <vector>

namespace robot {

// Pattern 1: Single ownership with unique_ptr
class RobotArm {
    std::unique_ptr<MotorController> motors[6];
    
public:
    RobotArm() {
        for (int i = 0; i < 6; ++i) {
            motors[i] = std::make_unique<MotorController>(i);
        }
    }
    
    // Move-only semantics
    RobotArm(RobotArm&&) = default;
    RobotArm& operator=(RobotArm&&) = default;
    
    // No copy
    RobotArm(const RobotArm&) = delete;
    RobotArm& operator=(const RobotArm&) = delete;
};

// Pattern 2: Observer pattern for shared access
class Logger {
public:
    void log(std::string_view msg);
};

class Component {
    Logger* logger;  // Non-owning observer
    
public:
    explicit Component(Logger& log) : logger(&log) {}
    
    void do_work() {
        if (logger) {
            logger->log("Working...");
        }
    }
};

// Pattern 3: Value semantics for small objects
struct Pose {
    float x, y, z;
    float roll, pitch, yaw;
    
    // Pass by value for small structs
    Pose transform(const Pose& other) const {
        return Pose{
            x + other.x, y + other.y, z + other.z,
            roll + other.roll, pitch + other.pitch, yaw + other.yaw
        };
    }
};

// Pattern 4: Pre-allocated pools for real-time
template<typename T, size_t N>
class ObjectPool {
    std::array<T, N> storage;
    std::array<bool, N> used{};
    
public:
    T* allocate() {
        for (size_t i = 0; i < N; ++i) {
            if (!used[i]) {
                used[i] = true;
                return &storage[i];
            }
        }
        return nullptr;
    }
    
    void deallocate(T* ptr) {
        if (ptr >= &storage[0] && ptr < &storage[N]) {
            size_t idx = ptr - &storage[0];
            used[idx] = false;
        }
    }
};

} // namespace robot
```

### 5.2 Smart Pointer Guidelines

```cpp
// When to use each smart pointer type

// unique_ptr: Default choice for heap objects
class System {
    std::unique_ptr<Subsystem> subsystem;
public:
    System() : subsystem(std::make_unique<Subsystem>()) {}
};

// shared_ptr: Only when truly needed (e.g., async callbacks)
class AsyncProcessor {
    void process_async(std::shared_ptr<Data> data) {
        // Launch async task that needs data to survive
        std::thread([data]() {
            // Process data in background
            process(*data);
        }).detach();
    }
};

// weak_ptr: Break circular dependencies (rare)
class Node {
    std::shared_ptr<Node> child;
    std::weak_ptr<Node> parent;  // Breaks cycle
};

// Raw pointers: Non-owning observers
class Display {
    const Robot* robot;  // Just observing, not owning
public:
    explicit Display(const Robot& r) : robot(&r) {}
    void render() {
        if (robot) {
            draw_robot(*robot);
        }
    }
};
```

---

## Part II: Core Libraries

## 6. Essential Libraries Overview

### 6.1 Library Selection Criteria

We choose libraries that are:
- **Well-maintained** with active communities
- **Header-only or easy to integrate**
- **Performance-oriented** for robotics use cases
- **Well-documented** with clear examples

### 6.2 Core Library Stack

| Library | Purpose | Version | Usage |
|---------|---------|---------|-------|
| fmt | String formatting | 10.x | All string operations |
| spdlog | Logging | 1.12.x | Async logging |
| Catch2 | Unit testing | 3.x | All tests |
| Eigen | Linear algebra | 3.4.x | Math operations |
| CycloneDDS | Communication | 0.10.x | Inter-process/network |
| ImGui | User interface | 1.89.x | Debug/control UI |
| moodycamel | Lock-free queues | Latest | RT communication |
| CLI11 | Command-line parsing | 2.x | Program arguments |

---

## 7. String Formatting with fmt

### 7.1 Basic Formatting

```cpp
// fmt_examples.cpp
#include <fmt/core.h>
#include <fmt/format.h>
#include <fmt/chrono.h>
#include <fmt/ranges.h>
#include <fmt/color.h>

void formatting_basics() {
    // Basic formatting
    std::string msg = fmt::format("Motor {} at position {:.2f}°", 1, 45.678);
    // Output: "Motor 1 at position 45.68°"
    
    // Print directly
    fmt::print("Status: {}\n", "OK");
    
    // Positional arguments
    fmt::print("{1} {0}\n", "world", "Hello");  // "Hello world"
    
    // Named arguments
    fmt::print("x={x}, y={y}\n", fmt::arg("x", 10), fmt::arg("y", 20));
    
    // Formatting specifications
    fmt::print("{:>10}\n", "right");     // Right align
    fmt::print("{:^10}\n", "center");    // Center align
    fmt::print("{:<10}\n", "left");      // Left align
    fmt::print("{:*^10}\n", "center");   // Center with fill
    
    // Number formatting
    fmt::print("{:04d}\n", 42);          // "0042"
    fmt::print("{:#x}\n", 255);          // "0xff"
    fmt::print("{:+.2f}\n", 3.14);       // "+3.14"
    fmt::print("{:.2e}\n", 1234.5);      // "1.23e+03"
}

void advanced_formatting() {
    // Containers
    std::vector<int> values = {1, 2, 3, 4, 5};
    fmt::print("Values: {}\n", values);  // "Values: [1, 2, 3, 4, 5]"
    
    // Time formatting
    auto now = std::chrono::system_clock::now();
    fmt::print("Time: {:%Y-%m-%d %H:%M:%S}\n", now);
    
    // Custom types
    struct Point {
        float x, y;
    };
    
    // Make Point formattable
    template <>
    struct fmt::formatter<Point> {
        constexpr auto parse(format_parse_context& ctx) {
            return ctx.begin();
        }
        
        template <typename FormatContext>
        auto format(const Point& p, FormatContext& ctx) const {
            return fmt::format_to(ctx.out(), "({:.2f}, {:.2f})", p.x, p.y);
        }
    };
    
    Point p{3.14f, 2.71f};
    fmt::print("Point: {}\n", p);  // "Point: (3.14, 2.71)"
    
    // Colored output
    fmt::print(fg(fmt::color::green), "Success: ");
    fmt::print("Operation completed\n");
    fmt::print(fg(fmt::color::red) | fmt::emphasis::bold, 
               "Error: Invalid parameter\n");
}

// For real-time: pre-allocated formatting
void realtime_formatting() {
    // Use memory_buffer for zero allocation
    fmt::memory_buffer buffer;
    
    for (int i = 0; i < 100; ++i) {
        buffer.clear();
        fmt::format_to(std::back_inserter(buffer), 
                      "Sensor {}: {:.3f}\n", i, read_sensor(i));
        
        // Use buffer.data() and buffer.size() to access formatted string
        write_to_log(buffer.data(), buffer.size());
    }
    
    // Fixed-size buffer for real-time
    std::array<char, 256> rt_buffer;
    auto result = fmt::format_to_n(rt_buffer.data(), rt_buffer.size(),
                                   "RT data: {}", get_rt_data());
    *result.out = '\0';  // Null terminate
}
```

---

## 8. Logging with spdlog

### 8.1 Logger Setup and Configuration

```cpp
// logging_setup.cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <spdlog/sinks/rotating_file_sink.h>
#include <spdlog/sinks/daily_file_sink.h>
#include <spdlog/async.h>
#include <memory>

namespace robot {

class LogManager {
public:
    static void initialize() {
        // Create thread pool for async logging
        spdlog::init_thread_pool(8192, 1);  // queue size, thread count
        
        // Console sink with colors
        auto console_sink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
        console_sink->set_level(spdlog::level::debug);
        console_sink->set_pattern("[%Y-%m-%d %H:%M:%S.%e] [%^%l%$] [%n] %v");
        
        // Rotating file sink (5MB x 3 files)
        auto file_sink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
            "logs/robot.log", 5 * 1024 * 1024, 3);
        file_sink->set_level(spdlog::level::trace);
        
        // Daily file sink
        auto daily_sink = std::make_shared<spdlog::sinks::daily_file_sink_mt>(
            "logs/daily.log", 2, 30);  // 2:30am rotation
        
        // Combine sinks
        std::vector<spdlog::sink_ptr> sinks{console_sink, file_sink, daily_sink};
        
        // Create async logger
        auto logger = std::make_shared<spdlog::async_logger>(
            "robot", 
            sinks.begin(), 
            sinks.end(), 
            spdlog::thread_pool(),
            spdlog::async_overflow_policy::block);
        
        // Set as default
        spdlog::set_default_logger(logger);
        spdlog::set_level(spdlog::level::debug);
        
        // Flush every 3 seconds
        spdlog::flush_every(std::chrono::seconds(3));
        
        spdlog::info("Logging system initialized");
    }
    
    static std::shared_ptr<spdlog::logger> get_logger(const std::string& name) {
        auto logger = spdlog::get(name);
        if (!logger) {
            logger = spdlog::default_logger()->clone(name);
            spdlog::register_logger(logger);
        }
        return logger;
    }
};

// Module-specific loggers
class MotorLogger {
    std::shared_ptr<spdlog::logger> logger;
    
public:
    MotorLogger() : logger(LogManager::get_logger("motor")) {}
    
    void log_position(int motor_id, float position) {
        logger->debug("Motor {} position: {:.3f}", motor_id, position);
    }
    
    void log_error(int motor_id, ErrorCode error) {
        logger->error("Motor {} error: {}", motor_id, to_string(error));
    }
    
    void log_command(int motor_id, float target) {
        logger->trace("Motor {} command: {:.3f}", motor_id, target);
    }
};

// Structured logging
struct LogEvent {
    std::string component;
    std::string action;
    std::unordered_map<std::string, std::string> fields;
    
    void log() const {
        fmt::memory_buffer buffer;
        fmt::format_to(std::back_inserter(buffer), 
                      "component={} action={}", component, action);
        
        for (const auto& [key, value] : fields) {
            fmt::format_to(std::back_inserter(buffer), 
                          " {}={}", key, value);
        }
        
        spdlog::info(std::string_view(buffer.data(), buffer.size()));
    }
};

// Usage examples
void logging_examples() {
    // Initialize once at startup
    LogManager::initialize();
    
    // Basic logging
    spdlog::trace("Detailed trace message");
    spdlog::debug("Debug information");
    spdlog::info("Information message");
    spdlog::warn("Warning message");
    spdlog::error("Error message");
    spdlog::critical("Critical error!");
    
    // With formatting
    int motor_id = 1;
    float position = 45.67f;
    spdlog::info("Motor {} at position {:.2f}", motor_id, position);
    
    // Structured logging
    LogEvent event{
        .component = "motor_controller",
        .action = "move",
        .fields = {
            {"motor_id", "1"},
            {"target", "45.67"},
            {"duration_ms", "250"}
        }
    };
    event.log();
    
    // Module logger
    MotorLogger motor_log;
    motor_log.log_position(1, 45.67f);
    
    // Log only in debug mode
    SPDLOG_DEBUG("This only logs in debug builds");
    
    // Conditional logging
    spdlog::log_if(position > 100, spdlog::level::warn, 
                   "Position {} exceeds limit", position);
}

} // namespace robot
```

### 8.2 Real-Time Safe Logging

```cpp
// rt_logging.cpp
#include <atomic>
#include <array>

namespace robot {

// Lock-free ring buffer for RT logging
template<size_t SIZE>
class RTLogBuffer {
    struct LogEntry {
        std::chrono::steady_clock::time_point timestamp;
        spdlog::level::level_enum level;
        std::array<char, 256> message;
        size_t message_len;
    };
    
    std::array<LogEntry, SIZE> buffer;
    std::atomic<size_t> write_index{0};
    std::atomic<size_t> read_index{0};
    
public:
    // Called from RT thread
    void log_rt(spdlog::level::level_enum level, 
                std::string_view message) {
        size_t write = write_index.load(std::memory_order_relaxed);
        size_t next = (write + 1) % SIZE;
        
        // Check if buffer is full
        if (next == read_index.load(std::memory_order_acquire)) {
            return;  // Drop message
        }
        
        // Copy message
        auto& entry = buffer[write];
        entry.timestamp = std::chrono::steady_clock::now();
        entry.level = level;
        entry.message_len = std::min(message.size(), entry.message.size() - 1);
        std::memcpy(entry.message.data(), message.data(), entry.message_len);
        entry.message[entry.message_len] = '\0';
        
        write_index.store(next, std::memory_order_release);
    }
    
    // Called from non-RT thread
    void flush_to_logger() {
        size_t read = read_index.load(std::memory_order_relaxed);
        
        while (read != write_index.load(std::memory_order_acquire)) {
            const auto& entry = buffer[read];
            
            // Log to spdlog
            spdlog::log(entry.level, "[RT] {}", 
                       std::string_view(entry.message.data(), entry.message_len));
            
            read = (read + 1) % SIZE;
            read_index.store(read, std::memory_order_release);
        }
    }
};

// Global RT logger
inline RTLogBuffer<1024> rt_log_buffer;

// Macros for RT-safe logging
#define RT_LOG_DEBUG(fmt, ...) \
    do { \
        char buffer[256]; \
        int len = snprintf(buffer, sizeof(buffer), fmt, ##__VA_ARGS__); \
        rt_log_buffer.log_rt(spdlog::level::debug, \
                            std::string_view(buffer, len)); \
    } while(0)

#define RT_LOG_ERROR(fmt, ...) \
    do { \
        char buffer[256]; \
        int len = snprintf(buffer, sizeof(buffer), fmt, ##__VA_ARGS__); \
        rt_log_buffer.log_rt(spdlog::level::err, \
                            std::string_view(buffer, len)); \
    } while(0)

// RT thread usage
void realtime_control_loop() {
    while (running) {
        float sensor_value = read_sensor();
        
        // RT-safe logging
        RT_LOG_DEBUG("Sensor: %.3f", sensor_value);
        
        if (sensor_value > 100.0f) {
            RT_LOG_ERROR("Sensor value %.3f exceeds limit", sensor_value);
        }
        
        // Control logic...
    }
}

// Non-RT thread flushes logs
void logging_thread() {
    while (running) {
        rt_log_buffer.flush_to_logger();
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

} // namespace robot
```

---

## 9. Testing with Catch2

### 9.1 Test Structure and Organization

```cpp
// test_motor_controller.cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp>
#include <catch2/benchmark/catch_benchmark.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>
#include "motor_controller.h"

using namespace robot;
using Catch::Approx;

// Test fixture for setup/teardown
class MotorControllerFixture {
protected:
    std::unique_ptr<MotorController> motor;
    static constexpr float MAX_POSITION = 6.28f;
    
    MotorControllerFixture() {
        motor = std::make_unique<MotorController>();
        motor->initialize();
    }
};

TEST_CASE("Motor Controller Initialization", "[motor][init]") {
    MotorController motor;
    
    SECTION("Initial state") {
        REQUIRE_FALSE(motor.is_initialized());
        auto pos_result = motor.get_position();
        REQUIRE_FALSE(pos_result.has_value());
        REQUIRE(pos_result.error() == ErrorCode::SYS_NOT_INITIALIZED);
    }
    
    SECTION("Successful initialization") {
        auto status = motor.initialize();
        REQUIRE(status.has_value());
        REQUIRE(motor.is_initialized());
        
        // Position should be readable after init
        auto pos_result = motor.get_position();
        REQUIRE(pos_result.has_value());
        REQUIRE(pos_result.value() == Approx(0.0f));
    }
    
    SECTION("Double initialization") {
        REQUIRE(motor.initialize().has_value());
        
        auto second_init = motor.initialize();
        REQUIRE_FALSE(second_init.has_value());
        REQUIRE(second_init.error() == ErrorCode::SYS_ALREADY_RUNNING);
    }
}

TEST_CASE_METHOD(MotorControllerFixture, 
                 "Motor Movement", "[motor][movement]") {
    
    SECTION("Valid movement") {
        auto status = motor->move_to(1.57f);
        REQUIRE(status.has_value());
        
        // Allow time for movement (simulation)
        for (int i = 0; i < 100; ++i) {
            motor->update(0.001f);
        }
        
        auto pos = motor->get_position();
        REQUIRE(pos.has_value());
        REQUIRE_THAT(pos.value(), 
                    Catch::Matchers::WithinRel(1.57f, 0.01f));
    }
    
    SECTION("Out of range movement") {
        auto status = motor->move_to(10.0f);
        REQUIRE_FALSE(status.has_value());
        REQUIRE(status.error() == ErrorCode::PARAM_OUT_OF_RANGE);
        
        // Position should not change
        auto pos = motor->get_position();
        REQUIRE(pos.value() == Approx(0.0f));
    }
    
    SECTION("Multiple movements") {
        std::vector<float> targets = {1.0f, 2.0f, 3.0f, 2.0f, 1.0f};
        
        for (float target : targets) {
            REQUIRE(motor->move_to(target).has_value());
            
            // Simulate movement
            for (int i = 0; i < 100; ++i) {
                motor->update(0.001f);
            }
            
            auto pos = motor->get_position();
            REQUIRE(pos.has_value());
            CHECK_THAT(pos.value(), 
                      Catch::Matchers::WithinAbs(target, 0.1f));
        }
    }
}

TEST_CASE("Motor Performance", "[motor][performance][!benchmark]") {
    MotorController motor;
    motor.initialize();
    
    BENCHMARK("Initialize") {
        MotorController m;
        return m.initialize();
    };
    
    BENCHMARK("Move command") {
        return motor.move_to(1.57f);
    };
    
    BENCHMARK("Update cycle") {
        motor.update(0.001f);
    };
    
    BENCHMARK("Get position") {
        return motor.get_position();
    };
}

// Property-based testing
TEST_CASE("Motor Properties", "[motor][properties]") {
    MotorController motor;
    motor.initialize();
    
    // Property: Position always within bounds after valid move
    SECTION("Position bounds invariant") {
        for (int i = 0; i < 100; ++i) {
            float random_pos = (rand() / float(RAND_MAX)) * 12.0f - 6.0f;
            
            auto status = motor.move_to(random_pos);
            
            if (status.has_value()) {
                // Simulate movement
                for (int j = 0; j < 100; ++j) {
                    motor.update(0.001f);
                }
                
                auto pos = motor.get_position();
                REQUIRE(pos.has_value());
                REQUIRE(std::abs(pos.value()) <= 6.28f);
            }
        }
    }
}

// Integration tests
TEST_CASE("Motor System Integration", "[integration]") {
    struct System {
        MotorController motors[6];
        
        Status initialize_all() {
            for (int i = 0; i < 6; ++i) {
                if (auto status = motors[i].initialize(); !status) {
                    return status;
                }
            }
            return OkStatus();
        }
        
        Status move_all(float position) {
            for (int i = 0; i < 6; ++i) {
                if (auto status = motors[i].move_to(position); !status) {
                    return status;
                }
            }
            return OkStatus();
        }
    };
    
    System system;
    
    REQUIRE(system.initialize_all().has_value());
    REQUIRE(system.move_all(1.0f).has_value());
    
    // Verify all motors moved
    for (int i = 0; i < 6; ++i) {
        // Simulate
        for (int j = 0; j < 100; ++j) {
            system.motors[i].update(0.001f);
        }
        
        auto pos = system.motors[i].get_position();
        REQUIRE(pos.has_value());
        CHECK_THAT(pos.value(), Catch::Matchers::WithinRel(1.0f, 0.1f));
    }
}
```

### 9.2 Test Execution and Coverage

```cmake
# CMakeLists.txt test configuration
Include(FetchContent)

FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v3.4.0
)

FetchContent_MakeAvailable(Catch2)

# Create test executable
add_executable(tests
    test_main.cpp
    test_motor_controller.cpp
    test_robot_system.cpp
)

target_link_libraries(tests PRIVATE 
    Catch2::Catch2WithMain
    robot_lib
)

# Enable testing
include(CTest)
include(Catch)
catch_discover_tests(tests)

# Coverage configuration
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(tests PRIVATE --coverage)
    target_link_options(tests PRIVATE --coverage)
endif()
```

```bash
#!/bin/bash
# run_tests.sh - Test execution with coverage

# Build tests
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# Run tests
cd build
./tests

# Run with detailed output
./tests --reporter console --success

# Run specific test
./tests "[motor]"

# Run benchmarks
./tests "[!benchmark]"

# Generate coverage report
gcov tests
lcov --capture --directory . --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
lcov --list coverage.info
```

---

## 10. Linear Algebra with Eigen

### 10.1 Eigen Basics for Robotics

```cpp
// eigen_basics.cpp
#include <Eigen/Core>
#include <Eigen/Geometry>
#include <fmt/core.h>

namespace robot {

// Type aliases for cleaner code
using Vector3f = Eigen::Vector3f;
using Vector6f = Eigen::Matrix<float, 6, 1>;
using Matrix3f = Eigen::Matrix3f;
using Matrix4f = Eigen::Matrix4f;
using Matrix6f = Eigen::Matrix<float, 6, 6>;
using Quaternionf = Eigen::Quaternionf;
using AngleAxisf = Eigen::AngleAxisf;
using Isometry3f = Eigen::Isometry3f;

class KinematicsCalculator {
public:
    // Forward kinematics for 6-DOF arm
    struct DHParameters {
        float a;      // Link length
        float alpha;  // Link twist
        float d;      // Link offset
        float theta;  // Joint angle
    };
    
    static Matrix4f dh_transform(const DHParameters& params) {
        float ct = std::cos(params.theta);
        float st = std::sin(params.theta);
        float ca = std::cos(params.alpha);
        float sa = std::sin(params.alpha);
        
        Matrix4f T;
        T << ct, -st * ca,  st * sa, params.a * ct,
             st,  ct * ca, -ct * sa, params.a * st,
              0,       sa,       ca, params.d,
              0,        0,        0, 1;
        
        return T;
    }
    
    // Calculate end-effector pose
    static Isometry3f forward_kinematics(const Vector6f& joint_angles) {
        // DH parameters for each joint (example values)
        std::array<DHParameters, 6> dh_params = {{
            {0,     M_PI/2,  0.089, joint_angles[0]},
            {-0.425, 0,      0,     joint_angles[1]},
            {-0.392, 0,      0,     joint_angles[2]},
            {0,      M_PI/2, 0.109, joint_angles[3]},
            {0,     -M_PI/2, 0.095, joint_angles[4]},
            {0,      0,      0.082, joint_angles[5]}
        }};
        
        Isometry3f T = Isometry3f::Identity();
        
        for (const auto& params : dh_params) {
            T = T * Isometry3f(dh_transform(params));
        }
        
        return T;
    }
    
    // Jacobian calculation
    static Matrix6f calculate_jacobian(const Vector6f& joint_angles) {
        Matrix6f J = Matrix6f::Zero();
        
        // Calculate transformation to each joint
        std::array<Isometry3f, 7> transforms;
        transforms[0] = Isometry3f::Identity();
        
        std::array<DHParameters, 6> dh_params = {{
            {0,     M_PI/2,  0.089, joint_angles[0]},
            {-0.425, 0,      0,     joint_angles[1]},
            {-0.392, 0,      0,     joint_angles[2]},
            {0,      M_PI/2, 0.109, joint_angles[3]},
            {0,     -M_PI/2, 0.095, joint_angles[4]},
            {0,      0,      0.082, joint_angles[5]}
        }};
        
        for (size_t i = 0; i < 6; ++i) {
            transforms[i + 1] = transforms[i] * 
                               Isometry3f(dh_transform(dh_params[i]));
        }
        
        Vector3f p_end = transforms[6].translation();
        
        for (size_t i = 0; i < 6; ++i) {
            Vector3f z_i = transforms[i].rotation().col(2);
            Vector3f p_i = transforms[i].translation();
            
            // Linear velocity contribution
            J.block<3, 1>(0, i) = z_i.cross(p_end - p_i);
            
            // Angular velocity contribution
            J.block<3, 1>(3, i) = z_i;
        }
        
        return J;
    }
    
    // Simple inverse kinematics using Jacobian
    static Result<Vector6f> inverse_kinematics(
        const Vector6f& current_joints,
        const Isometry3f& target_pose,
        float tolerance = 1e-3f,
        int max_iterations = 100) {
        
        Vector6f joints = current_joints;
        
        for (int iter = 0; iter < max_iterations; ++iter) {
            Isometry3f current_pose = forward_kinematics(joints);
            
            // Calculate position error
            Vector3f pos_error = target_pose.translation() - 
                                current_pose.translation();
            
            // Calculate orientation error
            Quaternionf q_current(current_pose.rotation());
            Quaternionf q_target(target_pose.rotation());
            Quaternionf q_error = q_target * q_current.inverse();
            AngleAxisf angle_axis(q_error);
            Vector3f rot_error = angle_axis.angle() * angle_axis.axis();
            
            // Combined error
            Vector6f error;
            error.head<3>() = pos_error;
            error.tail<3>() = rot_error;
            
            // Check convergence
            if (error.norm() < tolerance) {
                return Ok(joints);
            }
            
            // Calculate Jacobian
            Matrix6f J = calculate_jacobian(joints);
            
            // Damped least squares (avoid singularities)
            float lambda = 0.01f;
            Matrix6f JtJ = J.transpose() * J;
            JtJ.diagonal() += Matrix6f::Identity().diagonal() * lambda;
            
            // Calculate joint update
            Vector6f delta_q = JtJ.ldlt().solve(J.transpose() * error);
            
            // Update joints
            joints += delta_q * 0.1f;  // Scale factor for stability
            
            // Apply joint limits
            for (int i = 0; i < 6; ++i) {
                joints[i] = std::clamp(joints[i], -M_PI, M_PI);
            }
        }
        
        return Err<Vector6f>(ErrorCode::IK_CONVERGENCE_FAILED);
    }
};

// Trajectory generation
class TrajectoryGenerator {
public:
    struct Waypoint {
        Vector6f position;
        float time;
    };
    
    // Generate cubic spline trajectory
    static std::vector<Vector6f> generate_trajectory(
        const std::vector<Waypoint>& waypoints,
        float dt = 0.001f) {
        
        std::vector<Vector6f> trajectory;
        
        for (size_t i = 0; i < waypoints.size() - 1; ++i) {
            const auto& start = waypoints[i];
            const auto& end = waypoints[i + 1];
            
            float duration = end.time - start.time;
            int steps = static_cast<int>(duration / dt);
            
            for (int step = 0; step < steps; ++step) {
                float t = step * dt / duration;
                
                // Cubic interpolation (smooth acceleration)
                float s = 3 * t * t - 2 * t * t * t;
                
                Vector6f pos = start.position + 
                              s * (end.position - start.position);
                trajectory.push_back(pos);
            }
        }
        
        return trajectory;
    }
};

// Usage examples
void eigen_usage_examples() {
    // Basic vector operations
    Vector3f v1(1.0f, 2.0f, 3.0f);
    Vector3f v2(4.0f, 5.0f, 6.0f);
    
    Vector3f v3 = v1 + v2;
    float dot = v1.dot(v2);
    Vector3f cross = v1.cross(v2);
    float norm = v1.norm();
    
    fmt::print("Vector sum: [{}, {}, {}]\n", v3.x(), v3.y(), v3.z());
    fmt::print("Dot product: {}\n", dot);
    fmt::print("Cross product: [{}, {}, {}]\n", 
              cross.x(), cross.y(), cross.z());
    fmt::print("Norm: {}\n", norm);
    
    // Matrix operations
    Matrix3f R = AngleAxisf(0.5f, Vector3f::UnitZ()).toRotationMatrix();
    Vector3f v_rotated = R * v1;
    
    // Transformation
    Isometry3f T = Isometry3f::Identity();
    T.rotate(AngleAxisf(0.5f, Vector3f::UnitZ()));
    T.pretranslate(Vector3f(1.0f, 2.0f, 3.0f));
    
    // Forward kinematics
    Vector6f joint_angles;
    joint_angles << 0, -M_PI/4, M_PI/4, 0, M_PI/2, 0;
    
    Isometry3f end_effector = 
        KinematicsCalculator::forward_kinematics(joint_angles);
    
    fmt::print("End effector position: [{}, {}, {}]\n",
              end_effector.translation().x(),
              end_effector.translation().y(),
              end_effector.translation().z());
    
    // Inverse kinematics
    Isometry3f target = Isometry3f::Identity();
    target.pretranslate(Vector3f(0.5f, 0.3f, 0.4f));
    
    auto ik_result = KinematicsCalculator::inverse_kinematics(
        joint_angles, target);
    
    if (ik_result.has_value()) {
        Vector6f solution = ik_result.value();
        fmt::print("IK solution found\n");
    }
}

} // namespace robot
```

---

## Part III: Concurrent Programming

## 11. Threading Fundamentals

### 11.1 Thread Creation and Management

```cpp
// threading_basics.cpp
#include <thread>
#include <atomic>
#include <chrono>
#include <fmt/core.h>

namespace robot {

class ThreadManager {
private:
    std::vector<std::thread> threads;
    std::atomic<bool> running{false};
    
public:
    // Basic thread creation
    void example_basic_thread() {
        // Method 1: Function pointer
        std::thread t1(standalone_function);
        
        // Method 2: Lambda
        std::thread t2([]() {
            fmt::print("Lambda thread\n");
        });
        
        // Method 3: Member function
        std::thread t3(&ThreadManager::worker_function, this);
        
        // Always join or detach
        t1.join();
        t2.join();
        t3.join();
    }
    
    // Thread with parameters
    void example_parameterized_thread() {
        int id = 1;
        float value = 3.14f;
        
        std::thread t([](int thread_id, float data) {
            fmt::print("Thread {} processing {}\n", thread_id, data);
        }, id, value);
        
        t.join();
    }
    
    // Managing multiple threads
    void start_workers(size_t num_workers) {
        running = true;
        
        for (size_t i = 0; i < num_workers; ++i) {
            threads.emplace_back([this, i]() {
                worker_loop(i);
            });
        }
        
        fmt::print("Started {} worker threads\n", num_workers);
    }
    
    void stop_workers() {
        running = false;
        
        for (auto& t : threads) {
            if (t.joinable()) {
                t.join();
            }
        }
        
        threads.clear();
        fmt::print("All workers stopped\n");
    }
    
private:
    void worker_function() {
        fmt::print("Worker thread ID: {}\n", 
                  std::this_thread::get_id());
    }
    
    void worker_loop(size_t worker_id) {
        // Set thread name (Linux)
        std::string name = fmt::format("Worker{}", worker_id);
        pthread_setname_np(pthread_self(), name.c_str());
        
        while (running) {
            // Do work
            process_data(worker_id);
            
            // Sleep to prevent busy waiting
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }
    
    void process_data(size_t worker_id) {
        // Simulate work
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    
    static void standalone_function() {
        fmt::print("Standalone thread function\n");
    }
};

// Thread lifecycle management
class ScopedThread {
    std::thread thread;
    
public:
    template<typename Func, typename... Args>
    explicit ScopedThread(Func&& func, Args&&... args)
        : thread(std::forward<Func>(func), std::forward<Args>(args)...) {}
    
    ~ScopedThread() {
        if (thread.joinable()) {
            thread.join();
        }
    }
    
    ScopedThread(ScopedThread&& other) noexcept 
        : thread(std::move(other.thread)) {}
    
    ScopedThread& operator=(ScopedThread&& other) noexcept {
        if (thread.joinable()) {
            thread.join();
        }
        thread = std::move(other.thread);
        return *this;
    }
    
    // Delete copy
    ScopedThread(const ScopedThread&) = delete;
    ScopedThread& operator=(const ScopedThread&) = delete;
};

// Usage
void thread_examples() {
    // Scoped thread automatically joins
    {
        ScopedThread t([]() {
            fmt::print("Scoped thread\n");
        });
    }  // Thread joined here
    
    // Manager example
    ThreadManager manager;
    manager.start_workers(4);
    
    // Let workers run
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    manager.stop_workers();
}

} // namespace robot
```

### 11.2 Thread Synchronization Primitives

```cpp
// synchronization.cpp
#include <mutex>
#include <condition_variable>
#include <shared_mutex>
#include <atomic>
#include <latch>
#include <barrier>
#include <semaphore>

namespace robot {

// Mutex examples
class MutexExamples {
    mutable std::mutex mutex;
    std::shared_mutex shared_mutex;
    int shared_data = 0;
    
public:
    // Basic mutex
    void increment() {
        std::lock_guard<std::mutex> lock(mutex);
        shared_data++;
    }
    
    // Unique lock for conditional locking
    bool try_increment() {
        std::unique_lock<std::mutex> lock(mutex, std::defer_lock);
        
        if (lock.try_lock()) {
            shared_data++;
            return true;
        }
        return false;
    }
    
    // Reader/writer lock
    void write_data(int value) {
        std::unique_lock<std::shared_mutex> lock(shared_mutex);
        shared_data = value;
    }
    
    int read_data() const {
        std::shared_lock<std::shared_mutex> lock(shared_mutex);
        return shared_data;
    }
    
    // Scoped locking multiple mutexes
    void transfer(MutexExamples& other, int amount) {
        // Lock both mutexes without deadlock
        std::scoped_lock lock(mutex, other.mutex);
        
        if (shared_data >= amount) {
            shared_data -= amount;
            other.shared_data += amount;
        }
    }
};

// Condition variable examples
class ProducerConsumer {
    std::mutex mutex;
    std::condition_variable cv;
    std::queue<int> queue;
    const size_t max_size = 10;
    bool done = false;
    
public:
    void produce(int item) {
        {
            std::unique_lock<std::mutex> lock(mutex);
            
            // Wait until queue has space
            cv.wait(lock, [this]() {
                return queue.size() < max_size || done;
            });
            
            if (done) return;
            
            queue.push(item);
            fmt::print("Produced: {}\n", item);
        }
        
        cv.notify_one();  // Notify consumer
    }
    
    std::optional<int> consume() {
        std::unique_lock<std::mutex> lock(mutex);
        
        // Wait until queue has data or done
        cv.wait(lock, [this]() {
            return !queue.empty() || done;
        });
        
        if (queue.empty()) {
            return std::nullopt;
        }
        
        int item = queue.front();
        queue.pop();
        
        lock.unlock();
        cv.notify_one();  // Notify producer
        
        return item;
    }
    
    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex);
            done = true;
        }
        cv.notify_all();
    }
};

// Atomic examples
class AtomicCounter {
    std::atomic<int> count{0};
    std::atomic<bool> flag{false};
    
public:
    void increment() {
        count.fetch_add(1, std::memory_order_relaxed);
    }
    
    int get() const {
        return count.load(std::memory_order_relaxed);
    }
    
    void set_flag() {
        flag.store(true, std::memory_order_release);
    }
    
    bool check_flag() const {
        return flag.load(std::memory_order_acquire);
    }
    
    // Compare and swap
    bool try_update(int expected, int desired) {
        return count.compare_exchange_strong(expected, desired);
    }
};

// C++20 synchronization primitives
class ModernSync {
public:
    // Latch - single use barrier
    void latch_example() {
        std::latch start_latch(1);
        std::vector<std::thread> threads;
        
        for (int i = 0; i < 4; ++i) {
            threads.emplace_back([&start_latch, i]() {
                // Wait for start signal
                start_latch.wait();
                fmt::print("Thread {} started\n", i);
            });
        }
        
        // Start all threads
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        start_latch.count_down();
        
        for (auto& t : threads) {
            t.join();
        }
    }
    
    // Barrier - reusable synchronization point
    void barrier_example() {
        const int num_threads = 4;
        std::barrier sync_point(num_threads, []() {
            fmt::print("All threads reached barrier\n");
        });
        
        std::vector<std::thread> threads;
        
        for (int i = 0; i < num_threads; ++i) {
            threads.emplace_back([&sync_point, i]() {
                for (int phase = 0; phase < 3; ++phase) {
                    fmt::print("Thread {} phase {}\n", i, phase);
                    sync_point.arrive_and_wait();
                }
            });
        }
        
        for (auto& t : threads) {
            t.join();
        }
    }
    
    // Semaphore - resource counting
    void semaphore_example() {
        std::counting_semaphore<3> resources(3);  // 3 resources
        
        auto worker = [&resources](int id) {
            resources.acquire();  // Get resource
            fmt::print("Worker {} acquired resource\n", id);
            
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            
            resources.release();  // Release resource
            fmt::print("Worker {} released resource\n", id);
        };
        
        std::vector<std::thread> threads;
        for (int i = 0; i < 6; ++i) {
            threads.emplace_back(worker, i);
        }
        
        for (auto& t : threads) {
            t.join();
        }
    }
};

} // namespace robot
```

---

## 12. Thread Safety Patterns

### 12.1 Lock-Free Data Structures

```cpp
// lock_free_structures.cpp
#include <atomic>
#include <memory>
#include "concurrentqueue.h"  // moodycamel

namespace robot {

// Simple lock-free stack
template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        std::atomic<Node*> next;
        
        explicit Node(T value) : data(std::move(value)), next(nullptr) {}
    };
    
    std::atomic<Node*> head{nullptr};
    
public:
    void push(T value) {
        Node* new_node = new Node(std::move(value));
        new_node->next = head.load();
        
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // Retry
        }
    }
    
    std::optional<T> pop() {
        Node* old_head = head.load();
        
        while (old_head && 
               !head.compare_exchange_weak(old_head, old_head->next)) {
            // Retry
        }
        
        if (!old_head) {
            return std::nullopt;
        }
        
        T value = std::move(old_head->data);
        delete old_head;
        return value;
    }
    
    ~LockFreeStack() {
        while (pop().has_value()) {
            // Clear stack
        }
    }
};

// Single producer, single consumer queue
template<typename T, size_t SIZE>
class SPSCQueue {
    static_assert((SIZE & (SIZE - 1)) == 0, "Size must be power of 2");
    
    std::array<T, SIZE> buffer;
    alignas(64) std::atomic<size_t> write_index{0};
    alignas(64) std::atomic<size_t> read_index{0};
    
public:
    bool try_push(const T& value) {
        size_t write = write_index.load(std::memory_order_relaxed);
        size_t next_write = (write + 1) & (SIZE - 1);
        
        if (next_write == read_index.load(std::memory_order_acquire)) {
            return false;  // Full
        }
        
        buffer[write] = value;
        write_index.store(next_write, std::memory_order_release);
        return true;
    }
    
    std::optional<T> try_pop() {
        size_t read = read_index.load(std::memory_order_relaxed);
        
        if (read == write_index.load(std::memory_order_acquire)) {
            return std::nullopt;  // Empty
        }
        
        T value = buffer[read];
        read_index.store((read + 1) & (SIZE - 1), 
                        std::memory_order_release);
        return value;
    }
    
    size_t size() const {
        size_t write = write_index.load(std::memory_order_acquire);
        size_t read = read_index.load(std::memory_order_acquire);
        return (write - read) & (SIZE - 1);
    }
};

// Using moodycamel concurrent queue
class HighPerformanceQueue {
    moodycamel::ConcurrentQueue<int> queue;
    
public:
    void producer(int start, int end) {
        for (int i = start; i < end; ++i) {
            queue.enqueue(i);
        }
    }
    
    void consumer() {
        int item;
        while (queue.try_dequeue(item)) {
            fmt::print("Consumed: {}\n", item);
        }
    }
    
    // Bulk operations
    void bulk_produce(const std::vector<int>& items) {
        queue.enqueue_bulk(items.begin(), items.size());
    }
    
    std::vector<int> bulk_consume(size_t max_items) {
        std::vector<int> items(max_items);
        size_t actual = queue.try_dequeue_bulk(items.begin(), max_items);
        items.resize(actual);
        return items;
    }
};

} // namespace robot
```

### 12.2 Thread Pool Implementation

```cpp
// thread_pool.cpp
#include <thread>
#include <functional>
#include <queue>
#include <future>

namespace robot {

class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mutex;
    std::condition_variable condition;
    std::atomic<bool> stop{false};
    
public:
    explicit ThreadPool(size_t num_threads = std::thread::hardware_concurrency()) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this]() {
                worker_loop();
            });
        }
    }
    
    ~ThreadPool() {
        stop = true;
        condition.notify_all();
        
        for (auto& worker : workers) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }
    
    // Submit task returning future
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args) 
        -> std::future<decltype(f(args...))> {
        
        using return_type = decltype(f(args...));
        
        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        std::future<return_type> result = task->get_future();
        
        {
            std::unique_lock<std::mutex> lock(mutex);
            
            if (stop) {
                throw std::runtime_error("ThreadPool is stopped");
            }
            
            tasks.emplace([task]() { (*task)(); });
        }
        
        condition.notify_one();
        return result;
    }
    
    // Submit and forget
    void execute(std::function<void()> task) {
        {
            std::unique_lock<std::mutex> lock(mutex);
            
            if (stop) {
                return;
            }
            
            tasks.emplace(std::move(task));
        }
        
        condition.notify_one();
    }
    
    size_t pending_tasks() const {
        std::unique_lock<std::mutex> lock(mutex);
        return tasks.size();
    }
    
private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;
            
            {
                std::unique_lock<std::mutex> lock(mutex);
                
                condition.wait(lock, [this]() {
                    return stop || !tasks.empty();
                });
                
                if (stop && tasks.empty()) {
                    return;
                }
                
                task = std::move(tasks.front());
                tasks.pop();
            }
            
            task();
        }
    }
};

// Usage examples
void thread_pool_examples() {
    ThreadPool pool(4);
    
    // Submit with future
    auto future1 = pool.submit([]() {
        return 42;
    });
    
    auto future2 = pool.submit([](int x, int y) {
        return x + y;
    }, 10, 20);
    
    // Get results
    fmt::print("Result 1: {}\n", future1.get());
    fmt::print("Result 2: {}\n", future2.get());
    
    // Fire and forget
    pool.execute([]() {
        fmt::print("Background task\n");
    });
    
    // Parallel processing
    std::vector<std::future<int>> futures;
    
    for (int i = 0; i < 10; ++i) {
        futures.push_back(pool.submit([i]() {
            return i * i;
        }));
    }
    
    for (size_t i = 0; i < futures.size(); ++i) {
        fmt::print("Square of {} is {}\n", i, futures[i].get());
    }
}

} // namespace robot
```

---

## 13. Lock-Free Programming

### 13.1 Memory Ordering and Atomics

```cpp
// memory_ordering.cpp
#include <atomic>
#include <thread>

namespace robot {

class MemoryOrderingExamples {
public:
    // Relaxed ordering - no synchronization
    void relaxed_example() {
        std::atomic<int> counter{0};
        
        auto increment = [&counter]() {
            for (int i = 0; i < 1000; ++i) {
                counter.fetch_add(1, std::memory_order_relaxed);
            }
        };
        
        std::thread t1(increment);
        std::thread t2(increment);
        
        t1.join();
        t2.join();
        
        // Counter will be 2000, but no ordering guarantees
        fmt::print("Counter: {}\n", counter.load());
    }
    
    // Acquire-Release for synchronization
    void acquire_release_example() {
        std::atomic<bool> ready{false};
        int data = 0;
        
        std::thread producer([&]() {
            data = 42;  // Non-atomic write
            ready.store(true, std::memory_order_release);
        });
        
        std::thread consumer([&]() {
            while (!ready.load(std::memory_order_acquire)) {
                // Wait
            }
            fmt::print("Data: {}\n", data);  // Will see 42
        });
        
        producer.join();
        consumer.join();
    }
    
    // Sequential consistency (default, strongest)
    void sequential_consistency_example() {
        std::atomic<bool> x{false}, y{false};
        std::atomic<int> z{0};
        
        std::thread t1([&]() {
            x.store(true);  // memory_order_seq_cst by default
        });
        
        std::thread t2([&]() {
            y.store(true);
        });
        
        std::thread t3([&]() {
            while (!x.load()) {
                // Wait for x
            }
            if (y.load()) {
                z++;
            }
        });
        
        std::thread t4([&]() {
            while (!y.load()) {
                // Wait for y
            }
            if (x.load()) {
                z++;
            }
        });
        
        t1.join();
        t2.join();
        t3.join();
        t4.join();
        
        // z will be at least 1 (both threads see consistent order)
        fmt::print("Z: {}\n", z.load());
    }
};

// Seqlock pattern for multiple readers, single writer
template<typename T>
class Seqlock {
    mutable std::atomic<uint64_t> seq{0};
    T data;
    
public:
    void write(const T& new_data) {
        uint64_t s = seq.load(std::memory_order_relaxed);
        seq.store(s + 1, std::memory_order_release);  // Start write
        
        data = new_data;
        
        seq.store(s + 2, std::memory_order_release);  // End write
    }
    
    std::optional<T> read() const {
        uint64_t s1, s2;
        T copy;
        
        do {
            s1 = seq.load(std::memory_order_acquire);
            if (s1 & 1) {
                continue;  // Write in progress
            }
            
            copy = data;
            
            s2 = seq.load(std::memory_order_acquire);
        } while (s1 != s2);
        
        return copy;
    }
};

} // namespace robot
```

---

## 14. Real-Time vs Non-Real-Time Code

### 14.1 Real-Time Constraints and Patterns

```cpp
// realtime_patterns.cpp
#include <chrono>
#include <sched.h>
#include <sys/mman.h>

namespace robot {

class RealtimeController {
    // Pre-allocated memory
    static constexpr size_t BUFFER_SIZE = 1024;
    std::array<float, BUFFER_SIZE> sensor_buffer;
    std::array<float, 6> joint_commands;
    
    // Lock-free communication
    SPSCQueue<SensorData, 64> sensor_queue;
    SPSCQueue<MotorCommand, 64> command_queue;
    
    // Timing
    std::chrono::steady_clock::time_point next_cycle;
    std::chrono::microseconds period{1000};  // 1kHz
    
public:
    void configure_realtime() {
        // Set CPU affinity
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(2, &cpuset);  // Use CPU core 2
        
        if (pthread_setaffinity_np(pthread_self(), 
                                   sizeof(cpuset), &cpuset) != 0) {
            fmt::print(stderr, "Failed to set CPU affinity\n");
        }
        
        // Set real-time priority
        struct sched_param param;
        param.sched_priority = 99;  // Highest priority
        
        if (pthread_setschedparam(pthread_self(), 
                                  SCHED_FIFO, &param) != 0) {
            fmt::print(stderr, "Failed to set RT priority\n");
        }
        
        // Lock memory to prevent page faults
        if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
            fmt::print(stderr, "Failed to lock memory\n");
        }
        
        // Pre-fault stack
        volatile char stack[65536];
        memset(const_cast<char*>(stack), 0, sizeof(stack));
    }
    
    void control_loop() {
        configure_realtime();
        
        next_cycle = std::chrono::steady_clock::now();
        
        while (running) {
            // Wait for next period
            next_cycle += period;
            std::this_thread::sleep_until(next_cycle);
            
            // Real-time work (no allocations!)
            read_sensors();
            compute_control();
            write_actuators();
            
            // Check deadline
            auto now = std::chrono::steady_clock::now();
            if (now > next_cycle) {
                deadline_miss_count++;
            }
        }
    }
    
private:
    void read_sensors() {
        // Fixed-time sensor reading
        SensorData data;
        if (sensor_queue.try_pop()) {
            // Process sensor data
            for (size_t i = 0; i < 6; ++i) {
                sensor_buffer[i] = data.values[i];
            }
        }
    }
    
    void compute_control() {
        // Fixed-time control computation
        for (size_t i = 0; i < 6; ++i) {
            // Simple P controller
            float error = setpoint[i] - sensor_buffer[i];
            joint_commands[i] = Kp * error;
        }
    }
    
    void write_actuators() {
        // Fixed-time actuator commands
        MotorCommand cmd;
        for (size_t i = 0; i < 6; ++i) {
            cmd.motor_id = i;
            cmd.torque = joint_commands[i];
            command_queue.try_push(cmd);  // Non-blocking
        }
    }
};

// Non-real-time support thread
class NonRealtimeSupport {
    ThreadPool pool{4};
    std::shared_ptr<spdlog::logger> logger;
    
public:
    void support_loop() {
        while (running) {
            // Can allocate memory
            auto data = std::make_unique<ProcessedData>();
            
            // Can use dynamic containers
            std::vector<float> analysis_results;
            
            // Can do file I/O
            save_telemetry(*data);
            
            // Can log
            logger->info("Support thread cycle");
            
            // Can sleep for variable time
            std::this_thread::sleep_for(
                std::chrono::milliseconds(100));
        }
    }
    
    void process_async(std::function<void()> task) {
        pool.execute(std::move(task));
    }
};

} // namespace robot
```

---

## Part IV: Communication

## 15. Inter-Thread Communication

### 15.1 Channel Implementation (Go-style)

```cpp
// channels.cpp
#include <queue>
#include <optional>

namespace robot {

template<typename T>
class Channel {
    std::queue<T> queue;
    std::mutex mutex;
    std::condition_variable cv_producers;
    std::condition_variable cv_consumers;
    size_t capacity;
    bool closed = false;
    
public:
    explicit Channel(size_t cap = 0) : capacity(cap) {}
    
    // Send value (blocks if full)
    bool send(T value) {
        std::unique_lock<std::mutex> lock(mutex);
        
        // Unbuffered channel or full buffer
        if (capacity == 0 || queue.size() >= capacity) {
            cv_producers.wait(lock, [this]() {
                return queue.size() < capacity || closed;
            });
        }
        
        if (closed) {
            return false;
        }
        
        queue.push(std::move(value));
        cv_consumers.notify_one();
        return true;
    }
    
    // Receive value (blocks if empty)
    std::optional<T> receive() {
        std::unique_lock<std::mutex> lock(mutex);
        
        cv_consumers.wait(lock, [this]() {
            return !queue.empty() || closed;
        });
        
        if (queue.empty()) {
            return std::nullopt;
        }
        
        T value = std::move(queue.front());
        queue.pop();
        
        cv_producers.notify_one();
        return value;
    }
    
    // Try to send (non-blocking)
    bool try_send(T value) {
        std::unique_lock<std::mutex> lock(mutex);
        
        if (closed || queue.size() >= capacity) {
            return false;
        }
        
        queue.push(std::move(value));
        cv_consumers.notify_one();
        return true;
    }
    
    // Try to receive (non-blocking)
    std::optional<T> try_receive() {
        std::unique_lock<std::mutex> lock(mutex);
        
        if (queue.empty()) {
            return std::nullopt;
        }
        
        T value = std::move(queue.front());
        queue.pop();
        
        cv_producers.notify_one();
        return value;
    }
    
    void close() {
        std::unique_lock<std::mutex> lock(mutex);
        closed = true;
        cv_producers.notify_all();
        cv_consumers.notify_all();
    }
    
    bool is_closed() const {
        std::unique_lock<std::mutex> lock(mutex);
        return closed;
    }
};

// Select-like functionality
template<typename T>
class Selector {
    std::vector<Channel<T>*> channels;
    
public:
    void add_channel(Channel<T>& ch) {
        channels.push_back(&ch);
    }
    
    std::pair<std::optional<T>, size_t> select() {
        while (true) {
            // Try all channels
            for (size_t i = 0; i < channels.size(); ++i) {
                if (auto value = channels[i]->try_receive()) {
                    return {value, i};
                }
            }
            
            // Brief sleep to avoid busy waiting
            std::this_thread::sleep_for(std::chrono::microseconds(10));
        }
    }
};

// Usage examples
void channel_examples() {
    // Unbuffered channel (synchronous)
    Channel<int> ch1(0);
    
    std::thread producer([&ch1]() {
        for (int i = 0; i < 10; ++i) {
            ch1.send(i);
            fmt::print("Sent: {}\n", i);
        }
        ch1.close();
    });
    
    std::thread consumer([&ch1]() {
        while (auto value = ch1.receive()) {
            fmt::print("Received: {}\n", *value);
        }
    });
    
    producer.join();
    consumer.join();
    
    // Buffered channel
    Channel<std::string> ch2(5);
    
    // Can send up to 5 without blocking
    ch2.send("Hello");
    ch2.send("World");
    
    // Fan-out pattern
    Channel<int> work_queue(100);
    
    auto worker = [&work_queue](int id) {
        while (auto job = work_queue.receive()) {
            fmt::print("Worker {} processing {}\n", id, *job);
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    };
    
    // Start workers
    std::vector<std::thread> workers;
    for (int i = 0; i < 4; ++i) {
        workers.emplace_back(worker, i);
    }
    
    // Send work
    for (int i = 0; i < 20; ++i) {
        work_queue.send(i);
    }
    
    work_queue.close();
    
    for (auto& w : workers) {
        w.join();
    }
}

} // namespace robot
```

---

## 16. CycloneDDS for ROS2

### 16.1 Basic DDS Setup and Configuration

```cpp
// dds_setup.cpp
#include <dds/dds.hpp>
#include <dds/pub/PublisherListener.hpp>
#include <dds/sub/SubscriberListener.hpp>

namespace robot {

// IDL definition (in separate .idl file)
/*
module robot_msgs {
    struct JointState {
        sequence<double> position;
        sequence<double> velocity;
        sequence<double> effort;
        string name;
        int64 timestamp;
    };
    
    struct MotorCommand {
        int32 motor_id;
        double target_position;
        double max_velocity;
        double max_torque;
    };
};
*/

class DDSManager {
    dds::domain::DomainParticipant participant;
    dds::topic::Topic<robot_msgs::JointState> joint_state_topic;
    dds::topic::Topic<robot_msgs::MotorCommand> command_topic;
    
    dds::pub::Publisher publisher;
    dds::sub::Subscriber subscriber;
    
public:
    DDSManager(int domain_id = 0) 
        : participant(domain_id) {
        
        // Configure QoS
        auto reliable_qos = dds::core::QosProvider::Default()
            .topic_qos("ReliableQos");
        
        // Create topics
        joint_state_topic = dds::topic::Topic<robot_msgs::JointState>(
            participant, "joint_states", reliable_qos);
            
        command_topic = dds::topic::Topic<robot_msgs::MotorCommand>(
            participant, "motor_commands", reliable_qos);
        
        // Create publisher/subscriber
        publisher = dds::pub::Publisher(participant);
        subscriber = dds::sub::Subscriber(participant);
        
        fmt::print("DDS initialized on domain {}\n", domain_id);
    }
    
    // Create a writer for publishing
    dds::pub::DataWriter<robot_msgs::JointState> 
    create_joint_state_writer() {
        return dds::pub::DataWriter<robot_msgs::JointState>(
            publisher, joint_state_topic);
    }
    
    // Create a reader for subscribing
    dds::sub::DataReader<robot_msgs::MotorCommand> 
    create_command_reader() {
        return dds::sub::DataReader<robot_msgs::MotorCommand>(
            subscriber, command_topic);
    }
};

// Publisher example
class JointStatePublisher {
    dds::pub::DataWriter<robot_msgs::JointState> writer;
    
public:
    explicit JointStatePublisher(DDSManager& dds) 
        : writer(dds.create_joint_state_writer()) {}
    
    void publish(const std::vector<double>& positions,
                 const std::vector<double>& velocities) {
        
        robot_msgs::JointState msg;
        msg.position(positions);
        msg.velocity(velocities);
        msg.effort(std::vector<double>(positions.size(), 0.0));
        msg.name("robot_arm");
        msg.timestamp(std::chrono::steady_clock::now()
                     .time_since_epoch().count());
        
        writer.write(msg);
        fmt::print("Published joint state\n");
    }
};

// Subscriber with callback
class MotorCommandSubscriber {
    dds::sub::DataReader<robot_msgs::MotorCommand> reader;
    std::function<void(const robot_msgs::MotorCommand&)> callback;
    
public:
    MotorCommandSubscriber(DDSManager& dds, 
                          std::function<void(const robot_msgs::MotorCommand&)> cb)
        : reader(dds.create_command_reader()), callback(cb) {
        
        // Set up listener
        reader.listener(new CommandListener(callback), 
                       dds::core::status::StatusMask::data_available());
    }
    
private:
    class CommandListener : public dds::sub::NoOpDataReaderListener<robot_msgs::MotorCommand> {
        std::function<void(const robot_msgs::MotorCommand&)> callback;
        
    public:
        explicit CommandListener(std::function<void(const robot_msgs::MotorCommand&)> cb)
            : callback(cb) {}
        
        void on_data_available(dds::sub::DataReader<robot_msgs::MotorCommand>& reader) {
            auto samples = reader.take();
            
            for (const auto& sample : samples) {
                if (sample.info().valid()) {
                    callback(sample.data());
                }
            }
        }
    };
};

// Complete DDS example
void dds_example() {
    DDSManager dds(42);  // Domain ID 42
    
    // Publisher thread
    std::thread pub_thread([&dds]() {
        JointStatePublisher publisher(dds);
        
        for (int i = 0; i < 100; ++i) {
            std::vector<double> positions = {
                sin(i * 0.1), cos(i * 0.1), sin(i * 0.2),
                cos(i * 0.2), sin(i * 0.3), cos(i * 0.3)
            };
            
            std::vector<double> velocities(6, 0.0);
            
            publisher.publish(positions, velocities);
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    });
    
    // Subscriber with callback
    MotorCommandSubscriber subscriber(dds, 
        [](const robot_msgs::MotorCommand& cmd) {
            fmt::print("Received command: motor={}, target={:.3f}\n",
                      cmd.motor_id(), cmd.target_position());
        });
    
    pub_thread.join();
}

} // namespace robot
```

### 16.2 Advanced DDS Patterns (Complete)

```cpp
// dds_advanced.cpp
#include <dds/dds.hpp>
#include <future>
#include <chrono>

namespace robot {

// Complete Request-Reply Service Implementation
class DDSService {
    using Request = robot_msgs::ServiceRequest;
    using Reply = robot_msgs::ServiceReply;
    
    dds::domain::DomainParticipant participant;
    dds::topic::Topic<Request> request_topic;
    dds::topic::Topic<Reply> reply_topic;
    std::atomic<bool> running{true};
    
public:
    DDSService(int domain_id) : participant(domain_id) {
        // QoS for request-reply pattern
        auto req_qos = dds::core::QosProvider::Default()
            .topic_qos()
            .reliability().reliable()
            .history().keep_last(10)
            .durability().transient_local();
        
        request_topic = dds::topic::Topic<Request>(
            participant, "service_request", req_qos);
        reply_topic = dds::topic::Topic<Reply>(
            participant, "service_reply", req_qos);
    }
    
    // Service provider implementation
    void provide_service(std::function<Reply(const Request&)> handler) {
        auto reader = dds::sub::DataReader<Request>(
            dds::sub::Subscriber(participant), request_topic);
        auto writer = dds::pub::DataWriter<Reply>(
            dds::pub::Publisher(participant), reply_topic);
        
        while (running) {
            // Use waitset for efficient waiting
            dds::core::cond::WaitSet waitset;
            dds::sub::cond::ReadCondition read_condition(
                reader,
                dds::sub::status::DataState::any());
            
            waitset.attach_condition(read_condition);
            
            // Wait for data with timeout
            auto conditions = waitset.wait(dds::core::Duration::from_millisecs(100));
            
            if (!conditions.empty()) {
                auto samples = reader.take();
                
                for (const auto& sample : samples) {
                    if (sample.info().valid()) {
                        try {
                            Reply reply = handler(sample.data());
                            reply.request_id(sample.data().request_id());
                            writer.write(reply);
                        } catch (const std::exception& e) {
                            fmt::print(stderr, "Service handler error: {}\n", e.what());
                        }
                    }
                }
            }
        }
    }
    
    // Async service client
    std::future<Reply> call_service_async(const Request& request) {
        auto promise = std::make_shared<std::promise<Reply>>();
        auto future = promise->get_future();
        
        // Generate unique request ID
        static std::atomic<int64_t> request_counter{0};
        int64_t request_id = request_counter.fetch_add(1);
        
        Request req = request;
        req.request_id(request_id);
        
        // Set up reply listener
        auto reader = dds::sub::DataReader<Reply>(
            dds::sub::Subscriber(participant), reply_topic);
        
        std::thread([reader, request_id, promise]() mutable {
            auto timeout = std::chrono::steady_clock::now() + 
                          std::chrono::seconds(5);
            
            while (std::chrono::steady_clock::now() < timeout) {
                auto samples = reader.take();
                
                for (const auto& sample : samples) {
                    if (sample.info().valid() && 
                        sample.data().request_id() == request_id) {
                        promise->set_value(sample.data());
                        return;
                    }
                }
                
                std::this_thread::sleep_for(std::chrono::milliseconds(10));
            }
            
            promise->set_exception(std::make_exception_ptr(
                std::runtime_error("Service call timeout")));
        }).detach();
        
        // Send request
        auto writer = dds::pub::DataWriter<Request>(
            dds::pub::Publisher(participant), request_topic);
        writer.write(req);
        
        return future;
    }
    
    void shutdown() {
        running = false;
    }
};

// QoS Profiles for different use cases
class QoSProfiles {
public:
    // Best effort for high-frequency sensor data
    static dds::topic::qos::TopicQos sensor_data_qos() {
        return dds::core::QosProvider::Default()
            .topic_qos()
            .reliability().best_effort()
            .history().keep_last(1)
            .durability().volatile_durability()
            .deadline().deadline_period(dds::core::Duration::from_millisecs(10))
            .liveliness().automatic().lease_duration(dds::core::Duration::from_secs(1));
    }
    
    // Reliable for commands
    static dds::topic::qos::TopicQos command_qos() {
        return dds::core::QosProvider::Default()
            .topic_qos()
            .reliability().reliable()
            .history().keep_all()
            .durability().transient_local()
            .lifespan().lifespan_duration(dds::core::Duration::from_secs(10));
    }
    
    // Large data transfer
    static dds::topic::qos::TopicQos large_data_qos() {
        return dds::core::QosProvider::Default()
            .topic_qos()
            .reliability().reliable()
            .history().keep_last(5)
            .resource_limits()
                .max_samples(5)
                .max_instances(1)
                .max_samples_per_instance(5);
    }
};

// Discovery and monitoring
class DDSMonitor {
    dds::domain::DomainParticipant participant;
    
public:
    explicit DDSMonitor(int domain_id) : participant(domain_id) {
        // Set up built-in topic readers
        auto subscriber = dds::sub::Subscriber(participant);
        
        // Monitor participant discovery
        auto participant_reader = dds::sub::DataReader<dds::topic::ParticipantBuiltinTopicData>(
            subscriber,
            dds::topic::participant_topic(participant));
        
        participant_reader.listener(
            new ParticipantListener(),
            dds::core::status::StatusMask::data_available());
    }
    
private:
    class ParticipantListener : public dds::sub::NoOpDataReaderListener<dds::topic::ParticipantBuiltinTopicData> {
    public:
        void on_data_available(dds::sub::DataReader<dds::topic::ParticipantBuiltinTopicData>& reader) {
            auto samples = reader.take();
            
            for (const auto& sample : samples) {
                if (sample.info().valid()) {
                    const auto& data = sample.data();
                    
                    if (sample.info().state().instance_state() == dds::sub::status::InstanceState::alive()) {
                        fmt::print("Participant discovered: {}\n", 
                                  data.key().value()[0]);
                    } else {
                        fmt::print("Participant lost: {}\n", 
                                  data.key().value()[0]);
                    }
                }
            }
        }
    };
};

} // namespace robot
```

---

## 17. Message Design and Serialization

### 17.1 Efficient Message Design

```cpp
// message_design.cpp
#include <cstring>
#include <vector>
#include <msgpack.hpp>

namespace robot {

// Efficient binary message format
#pragma pack(push, 1)  // Disable padding
struct CompactJointState {
    uint32_t timestamp_ms;
    float positions[6];
    float velocities[6];
    uint8_t status_flags;
    
    // Serialization helpers
    static constexpr size_t SIZE = sizeof(CompactJointState);
    
    void to_bytes(uint8_t* buffer) const {
        std::memcpy(buffer, this, SIZE);
    }
    
    static CompactJointState from_bytes(const uint8_t* buffer) {
        CompactJointState state;
        std::memcpy(&state, buffer, SIZE);
        return state;
    }
};
#pragma pack(pop)

// MessagePack serialization for flexible messages
struct FlexibleCommand {
    int motor_id;
    std::string command_type;
    std::vector<float> parameters;
    std::map<std::string, float> metadata;
    
    MSGPACK_DEFINE(motor_id, command_type, parameters, metadata);
};

class MessageSerializer {
public:
    // Serialize to MessagePack
    static std::vector<uint8_t> serialize(const FlexibleCommand& cmd) {
        msgpack::sbuffer buffer;
        msgpack::pack(buffer, cmd);
        
        return std::vector<uint8_t>(buffer.data(), 
                                    buffer.data() + buffer.size());
    }
    
    // Deserialize from MessagePack
    static Result<FlexibleCommand> deserialize(const std::vector<uint8_t>& data) {
        try {
            msgpack::object_handle oh = msgpack::unpack(
                reinterpret_cast<const char*>(data.data()), data.size());
            
            FlexibleCommand cmd;
            oh.get().convert(cmd);
            return Ok(cmd);
        } catch (const msgpack::exception& e) {
            return Err<FlexibleCommand>(ErrorCode::DESERIALIZATION_FAILED);
        }
    }
};

// Ring buffer for message history
template<typename T, size_t N>
class MessageHistory {
    std::array<T, N> buffer;
    std::array<uint64_t, N> timestamps;
    size_t write_index = 0;
    bool full = false;
    
public:
    void add(const T& message) {
        buffer[write_index] = message;
        timestamps[write_index] = get_timestamp_us();
        
        write_index = (write_index + 1) % N;
        if (write_index == 0) {
            full = true;
        }
    }
    
    std::vector<T> get_recent(size_t count) const {
        std::vector<T> recent;
        size_t available = full ? N : write_index;
        count = std::min(count, available);
        
        for (size_t i = 0; i < count; ++i) {
            size_t idx = (write_index - 1 - i + N) % N;
            recent.push_back(buffer[idx]);
        }
        
        return recent;
    }
    
    std::optional<T> get_at_time(uint64_t timestamp) const {
        size_t available = full ? N : write_index;
        
        for (size_t i = 0; i < available; ++i) {
            if (timestamps[i] == timestamp) {
                return buffer[i];
            }
        }
        
        return std::nullopt;
    }
};

} // namespace robot
```

---

## Part V: User Interface

## 18. ImGui Fundamentals

### 18.1 Basic ImGui Setup

```cpp
// imgui_setup.cpp
#include <imgui.h>
#include <imgui_impl_glfw.h>
#include <imgui_impl_opengl3.h>
#include <GLFW/glfw3.h>

namespace robot {

class ImGuiApplication {
    GLFWwindow* window = nullptr;
    ImGuiContext* imgui_context = nullptr;
    
public:
    Result<void> initialize() {
        // Initialize GLFW
        if (!glfwInit()) {
            return Err<void>(ErrorCode::GUI_INIT_FAILED);
        }
        
        // GL version
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
        glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        
        // Create window
        window = glfwCreateWindow(1280, 720, "Robot Control Panel", nullptr, nullptr);
        if (!window) {
            glfwTerminate();
            return Err<void>(ErrorCode::GUI_WINDOW_FAILED);
        }
        
        glfwMakeContextCurrent(window);
        glfwSwapInterval(1); // Enable vsync
        
        // Setup ImGui
        IMGUI_CHECKVERSION();
        imgui_context = ImGui::CreateContext();
        ImGuiIO& io = ImGui::GetIO();
        io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;
        io.ConfigFlags |= ImGuiConfigFlags_DockingEnable;
        
        // Setup style
        ImGui::StyleColorsDark();
        
        // Setup platform/renderer bindings
        ImGui_ImplGlfw_InitForOpenGL(window, true);
        ImGui_ImplOpenGL3_Init("#version 330");
        
        return Ok();
    }
    
    void run() {
        while (!glfwWindowShouldClose(window)) {
            glfwPollEvents();
            
            // Start ImGui frame
            ImGui_ImplOpenGL3_NewFrame();
            ImGui_ImplGlfw_NewFrame();
            ImGui::NewFrame();
            
            // Render UI
            render_ui();
            
            // Rendering
            ImGui::Render();
            int display_w, display_h;
            glfwGetFramebufferSize(window, &display_w, &display_h);
            glViewport(0, 0, display_w, display_h);
            glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
            glClear(GL_COLOR_BUFFER_BIT);
            ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());
            
            glfwSwapBuffers(window);
        }
    }
    
    void shutdown() {
        ImGui_ImplOpenGL3_Shutdown();
        ImGui_ImplGlfw_Shutdown();
        ImGui::DestroyContext(imgui_context);
        
        glfwDestroyWindow(window);
        glfwTerminate();
    }
    
private:
    void render_ui() {
        // Main menu bar
        if (ImGui::BeginMainMenuBar()) {
            if (ImGui::BeginMenu("File")) {
                if (ImGui::MenuItem("Open Config")) {
                    // Handle open
                }
                if (ImGui::MenuItem("Save Config")) {
                    // Handle save
                }
                ImGui::Separator();
                if (ImGui::MenuItem("Exit")) {
                    glfwSetWindowShouldClose(window, true);
                }
                ImGui::EndMenu();
            }
            
            if (ImGui::BeginMenu("View")) {
                ImGui::MenuItem("Show Metrics", nullptr, &show_metrics);
                ImGui::MenuItem("Show Logs", nullptr, &show_logs);
                ImGui::EndMenu();
            }
            
            ImGui::EndMainMenuBar();
        }
        
        // Docking space
        ImGui::DockSpaceOverViewport(ImGui::GetMainViewport());
        
        // Various windows
        render_control_panel();
        render_telemetry();
        render_diagnostics();
        
        if (show_metrics) {
            ImGui::ShowMetricsWindow(&show_metrics);
        }
    }
    
    bool show_metrics = false;
    bool show_logs = false;
    
    void render_control_panel();
    void render_telemetry();
    void render_diagnostics();
};

} // namespace robot
```

### 18.2 Robot Control Interface

```cpp
// imgui_robot_control.cpp
namespace robot {

class RobotControlPanel {
    // Robot state
    std::array<float, 6> joint_positions = {0};
    std::array<float, 6> joint_targets = {0};
    std::array<float, 6> joint_velocities = {0};
    std::array<float, 6> joint_torques = {0};
    
    // Control modes
    enum ControlMode { POSITION, VELOCITY, TORQUE };
    ControlMode control_mode = POSITION;
    
    // Safety
    bool emergency_stop = false;
    bool motors_enabled = false;
    
    // Plotting data
    static constexpr size_t PLOT_HISTORY = 1000;
    std::array<float, PLOT_HISTORY> position_history = {0};
    size_t history_offset = 0;
    
public:
    void render() {
        // Control Window
        ImGui::Begin("Robot Control", nullptr, ImGuiWindowFlags_AlwaysAutoResize);
        
        // Emergency Stop Button
        ImGui::PushStyleColor(ImGuiCol_Button, 
            emergency_stop ? IM_COL32(255, 0, 0, 255) : IM_COL32(128, 0, 0, 255));
        
        if (ImGui::Button("EMERGENCY STOP", ImVec2(200, 50))) {
            emergency_stop = !emergency_stop;
            if (emergency_stop) {
                motors_enabled = false;
                // Trigger actual emergency stop
            }
        }
        ImGui::PopStyleColor();
        
        ImGui::Separator();
        
        // Motor Enable/Disable
        if (ImGui::Checkbox("Motors Enabled", &motors_enabled)) {
            if (motors_enabled && !emergency_stop) {
                // Enable motors
            } else {
                // Disable motors
            }
        }
        
        ImGui::Separator();
        
        // Control Mode Selection
        ImGui::Text("Control Mode:");
        ImGui::RadioButton("Position", (int*)&control_mode, POSITION);
        ImGui::SameLine();
        ImGui::RadioButton("Velocity", (int*)&control_mode, VELOCITY);
        ImGui::SameLine();
        ImGui::RadioButton("Torque", (int*)&control_mode, TORQUE);
        
        ImGui::Separator();
        
        // Joint Controls
        ImGui::Text("Joint Controls:");
        
        for (int i = 0; i < 6; ++i) {
            ImGui::PushID(i);
            
            ImGui::Text("Joint %d", i + 1);
            ImGui::SameLine(100);
            
            // Current position display
            ImGui::Text("Pos: %.2f°", joint_positions[i] * 180.0f / M_PI);
            ImGui::SameLine(200);
            
            // Target control
            switch (control_mode) {
                case POSITION: {
                    float target_deg = joint_targets[i] * 180.0f / M_PI;
                    if (ImGui::SliderFloat("##target", &target_deg, -180, 180, "%.1f°")) {
                        joint_targets[i] = target_deg * M_PI / 180.0f;
                    }
                    break;
                }
                case VELOCITY: {
                    ImGui::SliderFloat("##vel", &joint_velocities[i], -3.14f, 3.14f, "%.2f rad/s");
                    break;
                }
                case TORQUE: {
                    ImGui::SliderFloat("##torque", &joint_torques[i], -10.0f, 10.0f, "%.1f Nm");
                    break;
                }
            }
            
            ImGui::PopID();
        }
        
        ImGui::Separator();
        
        // Send Command Button
        if (ImGui::Button("Send Command", ImVec2(200, 30))) {
            if (motors_enabled && !emergency_stop) {
                send_command();
            }
        }
        
        // Preset Positions
        ImGui::Separator();
        ImGui::Text("Preset Positions:");
        
        if (ImGui::Button("Home")) {
            for (int i = 0; i < 6; ++i) {
                joint_targets[i] = 0;
            }
        }
        ImGui::SameLine();
        
        if (ImGui::Button("Ready")) {
            joint_targets = {0, -M_PI/4, M_PI/4, 0, M_PI/2, 0};
        }
        ImGui::SameLine();
        
        if (ImGui::Button("Stow")) {
            joint_targets = {0, -M_PI/2, M_PI/2, 0, 0, 0};
        }
        
        ImGui::End();
        
        // Telemetry Window
        render_telemetry();
        
        // Diagnostics Window
        render_diagnostics();
    }
    
private:
    void render_telemetry() {
        ImGui::Begin("Telemetry");
        
        // Joint position plot
        if (ImPlot::BeginPlot("Joint Positions", ImVec2(-1, 300))) {
            ImPlot::SetupAxes("Time", "Position (rad)");
            ImPlot::SetupAxisLimits(ImAxis_X1, 0, PLOT_HISTORY, ImGuiCond_Always);
            ImPlot::SetupAxisLimits(ImAxis_Y1, -M_PI, M_PI);
            
            for (int i = 0; i < 6; ++i) {
                std::string label = fmt::format("Joint {}", i + 1);
                ImPlot::PlotLine(label.c_str(), 
                                position_history.data(), 
                                PLOT_HISTORY, 
                                1.0, 
                                0.0, 
                                ImPlotLineFlags_None, 
                                history_offset);
            }
            
            ImPlot::EndPlot();
        }
        
        // Statistics table
        if (ImGui::BeginTable("Stats", 7, ImGuiTableFlags_Borders)) {
            ImGui::TableSetupColumn("Joint");
            ImGui::TableSetupColumn("Position");
            ImGui::TableSetupColumn("Velocity");
            ImGui::TableSetupColumn("Torque");
            ImGui::TableSetupColumn("Temperature");
            ImGui::TableSetupColumn("Current");
            ImGui::TableSetupColumn("Status");
            ImGui::TableHeadersRow();
            
            for (int i = 0; i < 6; ++i) {
                ImGui::TableNextRow();
                ImGui::TableSetColumnIndex(0);
                ImGui::Text("J%d", i + 1);
                
                ImGui::TableSetColumnIndex(1);
                ImGui::Text("%.2f°", joint_positions[i] * 180.0f / M_PI);
                
                ImGui::TableSetColumnIndex(2);
                ImGui::Text("%.2f", joint_velocities[i]);
                
                ImGui::TableSetColumnIndex(3);
                ImGui::Text("%.1f", joint_torques[i]);
                
                ImGui::TableSetColumnIndex(4);
                ImGui::Text("%.1f°C", get_temperature(i));
                
                ImGui::TableSetColumnIndex(5);
                ImGui::Text("%.2fA", get_current(i));
                
                ImGui::TableSetColumnIndex(6);
                ImGui::TextColored(is_fault(i) ? ImVec4(1, 0, 0, 1) : ImVec4(0, 1, 0, 1),
                                  is_fault(i) ? "FAULT" : "OK");
            }
            
            ImGui::EndTable();
        }
        
        ImGui::End();
    }
    
    void render_diagnostics() {
        ImGui::Begin("Diagnostics");
        
        // System Health
        ImGui::Text("System Health:");
        
        float cpu_usage = get_cpu_usage();
        ImGui::Text("CPU Usage:");
        ImGui::SameLine();
        ImGui::ProgressBar(cpu_usage / 100.0f, ImVec2(200, 0), 
                          fmt::format("{:.1f}%%", cpu_usage).c_str());
        
        float mem_usage = get_memory_usage();
        ImGui::Text("Memory Usage:");
        ImGui::SameLine();
        ImGui::ProgressBar(mem_usage / 100.0f, ImVec2(200, 0),
                          fmt::format("{:.1f}%%", mem_usage).c_str());
        
        ImGui::Separator();
        
        // Communication Status
        ImGui::Text("Communication:");
        
        render_status_light("CAN Bus", is_can_connected());
        render_status_light("DDS", is_dds_connected());
        render_status_light("Emergency Stop", !emergency_stop);
        
        ImGui::Separator();
        
        // Error Log
        ImGui::Text("Recent Errors:");
        
        ImGui::BeginChild("ErrorLog", ImVec2(0, 200), true,
                         ImGuiWindowFlags_HorizontalScrollbar);
        
        for (const auto& error : get_recent_errors()) {
            ImGui::TextColored(ImVec4(1, 0.5f, 0.5f, 1),
                              "[%s] %s", 
                              error.timestamp.c_str(),
                              error.message.c_str());
        }
        
        ImGui::EndChild();
        
        ImGui::End();
    }
    
    void render_status_light(const char* label, bool status) {
        ImGui::Text("%s:", label);
        ImGui::SameLine();
        
        ImDrawList* draw_list = ImGui::GetWindowDrawList();
        ImVec2 pos = ImGui::GetCursorScreenPos();
        
        ImU32 color = status ? IM_COL32(0, 255, 0, 255) : IM_COL32(255, 0, 0, 255);
        draw_list->AddCircleFilled(ImVec2(pos.x + 10, pos.y + 10), 8, color);
        
        ImGui::Dummy(ImVec2(20, 20));
    }
    
    void send_command() {
        // Send command based on control mode
        switch (control_mode) {
            case POSITION:
                // Send position command
                break;
            case VELOCITY:
                // Send velocity command
                break;
            case TORQUE:
                // Send torque command
                break;
        }
    }
    
    float get_temperature(int joint) { return 25.0f + joint * 2.0f; }
    float get_current(int joint) { return 0.5f + joint * 0.1f; }
    bool is_fault(int joint) { return false; }
    float get_cpu_usage() { return 45.0f; }
    float get_memory_usage() { return 62.0f; }
    bool is_can_connected() { return true; }
    bool is_dds_connected() { return true; }
    
    struct ErrorEntry {
        std::string timestamp;
        std::string message;
    };
    
    std::vector<ErrorEntry> get_recent_errors() {
        return {
            {"12:34:56", "Motor 3 overcurrent warning"},
            {"12:35:12", "Communication timeout on CAN bus"},
            {"12:36:45", "Temperature warning on motor 5"}
        };
    }
};

} // namespace robot
```

---

## 19. Real-Time Visualization

### 19.1 High-Performance Plotting

```cpp
// realtime_visualization.cpp
#include <implot.h>
#include <deque>

namespace robot {

class RealtimePlotter {
    struct PlotData {
        std::deque<float> x_data;
        std::deque<float> y_data;
        size_t max_points;
        float x_counter = 0;
        
        explicit PlotData(size_t max = 1000) : max_points(max) {}
        
        void add_point(float y) {
            x_data.push_back(x_counter++);
            y_data.push_back(y);
            
            if (x_data.size() > max_points) {
                x_data.pop_front();
                y_data.pop_front();
            }
        }
        
        void clear() {
            x_data.clear();
            y_data.clear();
            x_counter = 0;
        }
    };
    
    std::map<std::string, PlotData> plots;
    bool paused = false;
    float time_window = 10.0f;  // seconds
    
public:
    void add_data(const std::string& series, float value) {
        if (!paused) {
            plots[series].add_point(value);
        }
    }
    
    void render() {
        ImGui::Begin("Real-Time Plots");
        
        // Controls
        if (ImGui::Button(paused ? "Resume" : "Pause")) {
            paused = !paused;
        }
        
        ImGui::SameLine();
        if (ImGui::Button("Clear")) {
            for (auto& [name, data] : plots) {
                data.clear();
            }
        }
        
        ImGui::SameLine();
        ImGui::SliderFloat("Time Window", &time_window, 1.0f, 60.0f, "%.1f s");
        
        // Plot
        if (ImPlot::BeginPlot("##Realtime", ImVec2(-1, -1))) {
            ImPlot::SetupAxes("Time", "Value");
            
            // Dynamic axis limits
            if (!plots.empty()) {
                float x_max = plots.begin()->second.x_counter;
                float x_min = std::max(0.0f, x_max - time_window * 100);  // Assuming 100Hz
                ImPlot::SetupAxisLimits(ImAxis_X1, x_min, x_max, ImGuiCond_Always);
            }
            
            // Plot each series
            for (const auto& [name, data] : plots) {
                if (!data.x_data.empty()) {
                    ImPlot::PlotLine(name.c_str(),
                                    data.x_data.data(),
                                    data.y_data.data(),
                                    data.x_data.size());
                }
            }
            
            ImPlot::EndPlot();
        }
        
        ImGui::End();
    }
};

// 3D Visualization
class Robot3DViewer {
    struct JointTransform {
        Eigen::Vector3f position;
        Eigen::Quaternionf rotation;
    };
    
    std::array<JointTransform, 6> joint_transforms;
    float camera_distance = 5.0f;
    float camera_angle_x = 45.0f;
    float camera_angle_y = 30.0f;
    
public:
    void update_joint_angles(const std::array<float, 6>& angles) {
        // Update forward kinematics
        // This would use your actual robot kinematics
        for (size_t i = 0; i < 6; ++i) {
            joint_transforms[i].rotation = 
                Eigen::AngleAxisf(angles[i], Eigen::Vector3f::UnitZ());
        }
    }
    
    void render() {
        ImGui::Begin("3D Robot View");
        
        // Camera controls
        ImGui::SliderFloat("Distance", &camera_distance, 1.0f, 10.0f);
        ImGui::SliderFloat("Angle X", &camera_angle_x, -180.0f, 180.0f);
        ImGui::SliderFloat("Angle Y", &camera_angle_y, -90.0f, 90.0f);
        
        // Get available region
        ImVec2 avail = ImGui::GetContentRegionAvail();
        
        // Render to texture (simplified - actual implementation would use OpenGL)
        ImGui::Image(render_robot_to_texture(), avail);
        
        ImGui::End();
    }
    
private:
    ImTextureID render_robot_to_texture() {
        // This would render the robot using OpenGL
        // and return the texture ID
        return nullptr;
    }
};

} // namespace robot
```

---

## Part VI: Production

## 20. Build System and Tooling

### 20.1 Complete CMakeLists.txt

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(RobotSystem VERSION 1.0.0 LANGUAGES CXX)

# C++ Standard
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Build type
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# Export compile commands for IDEs
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Options
option(BUILD_TESTS "Build test programs" ON)
option(BUILD_GUI "Build GUI components" ON)
option(USE_SANITIZERS "Enable sanitizers in debug build" ON)
option(USE_STATIC_ANALYSIS "Run static analysis" ON)

# Compiler flags
set(WARNING_FLAGS
    -Wall -Wextra -Wpedantic -Werror
    -Wcast-align -Wcast-qual
    -Wconversion -Wsign-conversion
    -Wdouble-promotion -Wformat=2
    -Wnull-dereference -Wold-style-cast
    -Woverloaded-virtual -Wshadow
    -Wunused -Wno-unused-parameter
)

# Platform-specific flags
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(${WARNING_FLAGS})
    
    # Debug flags
    set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")
    
    # Release flags
    set(CMAKE_CXX_FLAGS_RELEASE "-O3 -DNDEBUG -march=native -flto")
    
    # Sanitizers for debug
    if(USE_SANITIZERS AND CMAKE_BUILD_TYPE STREQUAL "Debug")
        add_compile_options(
            -fsanitize=address
            -fsanitize=undefined
            -fsanitize=leak
            -fno-omit-frame-pointer
        )
        add_link_options(
            -fsanitize=address
            -fsanitize=undefined
            -fsanitize=leak
        )
    endif()
endif()

# Find packages
find_package(fmt REQUIRED)
find_package(spdlog REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(Threads REQUIRED)

# Find optional packages
find_package(CycloneDDS QUIET)
if(CycloneDDS_FOUND)
    message(STATUS "CycloneDDS found")
    add_definitions(-DHAS_CYCLONEDDS)
endif()

if(BUILD_GUI)
    find_package(glfw3 REQUIRED)
    find_package(OpenGL REQUIRED)
    find_package(imgui REQUIRED)
endif()

# Include directories
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# Library sources
set(LIB_SOURCES
    src/core/error_codes.cpp
    src/core/robot_system.cpp
    src/control/motor_controller.cpp
    src/control/trajectory_generator.cpp
    src/drivers/can_driver.cpp
    src/comm/dds_manager.cpp
    src/utils/logger.cpp
)

# Create main library
add_library(robot_lib STATIC ${LIB_SOURCES})

target_link_libraries(robot_lib PUBLIC
    fmt::fmt
    spdlog::spdlog
    Eigen3::Eigen
    Threads::Threads
)

if(CycloneDDS_FOUND)
    target_link_libraries(robot_lib PUBLIC
        CycloneDDS::ddsc
    )
endif()

# Main executable
add_executable(robot_control src/main.cpp)
target_link_libraries(robot_control PRIVATE robot_lib)

# GUI executable
if(BUILD_GUI)
    add_executable(robot_gui 
        src/ui/main_gui.cpp
        src/ui/control_panel.cpp
        src/ui/visualization.cpp
    )
    
    target_link_libraries(robot_gui PRIVATE
        robot_lib
        glfw
        OpenGL::GL
        imgui::imgui
    )
endif()

# Tests
if(BUILD_TESTS)
    Include(FetchContent)
    
    FetchContent_Declare(
        Catch2
        GIT_REPOSITORY https://github.com/catchorg/Catch2.git
        GIT_TAG v3.4.0
    )
    
    FetchContent_MakeAvailable(Catch2)
    
    enable_testing()
    
    add_executable(tests
        tests/test_main.cpp
        tests/test_motor_controller.cpp
        tests/test_robot_system.cpp
        tests/test_trajectory.cpp
    )
    
    target_link_libraries(tests PRIVATE
        robot_lib
        Catch2::Catch2WithMain
    )
    
    include(CTest)
    include(Catch)
    catch_discover_tests(tests)
endif()

# Installation
install(TARGETS robot_control robot_lib
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

install(DIRECTORY include/
    DESTINATION include
)

# Package configuration
include(CPack)
set(CPACK_PACKAGE_NAME "RobotSystem")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_GENERATOR "DEB;TGZ")
```

### 20.2 Static Analysis Integration

```python
#!/usr/bin/env python3
# tools/static_analysis.py

import subprocess
import sys
import os
from pathlib import Path

def run_clang_tidy(source_files):
    """Run clang-tidy on source files."""
    print("Running clang-tidy...")
    cmd = [
        "clang-tidy",
        "-p", "build",
        "--checks=-*,bugprone-*,performance-*,readability-*,modernize-*",
        "--warnings-as-errors=*"
    ] + source_files
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print("clang-tidy found issues:")
        print(result.stdout)
        return False
    return True

def run_cppcheck(source_dir):
    """Run cppcheck on source directory."""
    print("Running cppcheck...")
    cmd = [
        "cppcheck",
        "--enable=all",
        "--error-exitcode=1",
        "--suppress=missingIncludeSystem",
        "--quiet",
        source_dir
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print("cppcheck found issues:")
        print(result.stdout)
        return False
    return True

def run_cpplint(source_files):
    """Run cpplint on source files."""
    print("Running cpplint...")
    cmd = [
        "cpplint",
        "--filter=-build/include_subdir,-legal/copyright",
        "--linelength=100"
    ] + source_files
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        print("cpplint found style issues:")
        print(result.stderr)
        return False
    return True

def check_includes(source_files):
    """Check for forbidden includes."""
    forbidden = [
        "<iostream>",  # Use fmt instead
        "<boost/",     # No boost
        "<experimental/"  # No experimental features
    ]
    
    issues = []
    for filepath in source_files:
        with open(filepath, 'r') as f:
            for line_no, line in enumerate(f, 1):
                for forbidden_include in forbidden:
                    if forbidden_include in line:
                        issues.append(f"{filepath}:{line_no}: Forbidden include: {forbidden_include}")
    
    if issues:
        print("Forbidden includes found:")
        for issue in issues:
            print(f"  {issue}")
        return False
    return True

def main():
    # Get all C++ source files
    src_dir = Path("src")
    source_files = list(src_dir.glob("**/*.cpp")) + list(src_dir.glob("**/*.h"))
    source_files = [str(f) for f in source_files]
    
    if not source_files:
        print("No source files found")
        return 1
    
    # Run all checks
    all_passed = True
    
    if not run_clang_tidy(source_files):
        all_passed = False
    
    if not run_cppcheck("src"):
        all_passed = False
    
    if not run_cpplint(source_files):
        all_passed = False
    
    if not check_includes(source_files):
        all_passed = False
    
    if all_passed:
        print("All static analysis checks passed!")
        return 0
    else:
        print("Some checks failed. Please fix the issues.")
        return 1

if __name__ == "__main__":
    sys.exit(main())
```

---

## 21. Static Analysis and Sanitizers

### 21.1 Sanitizer Configuration

```cpp
// sanitizer_config.h
#pragma once

// Address Sanitizer options
#ifdef __has_feature
    #if __has_feature(address_sanitizer)
        #define ASAN_ENABLED
    #endif
#elif defined(__SANITIZE_ADDRESS__)
    #define ASAN_ENABLED
#endif

#ifdef ASAN_ENABLED
extern "C" {
    const char* __asan_default_options() {
        return "strict_string_checks=1:"
               "detect_stack_use_after_return=1:"
               "check_initialization_order=1:"
               "strict_init_order=1:"
               "print_stats=1:"
               "halt_on_error=0";
    }
}
#endif

// UBSan options
#ifdef __has_feature
    #if __has_feature(undefined_behavior_sanitizer)
        #define UBSAN_ENABLED
    #endif
#elif defined(__SANITIZE_UNDEFINED__)
    #define UBSAN_ENABLED
#endif

#ifdef UBSAN_ENABLED
extern "C" {
    const char* __ubsan_default_options() {
        return "print_stacktrace=1:"
               "halt_on_error=0:"
               "suppressions=ubsan.supp";
    }
}
#endif

// Thread Sanitizer options
#ifdef __has_feature
    #if __has_feature(thread_sanitizer)
        #define TSAN_ENABLED
    #endif
#elif defined(__SANITIZE_THREAD__)
    #define TSAN_ENABLED
#endif

#ifdef TSAN_ENABLED
extern "C" {
    const char* __tsan_default_options() {
        return "halt_on_error=0:"
               "history_size=7:"
               "suppressions=tsan.supp";
    }
}
#endif
```

### 21.2 Valgrind Suppression File

```
# valgrind.supp
{
   OpenGL/Mesa
   Memcheck:Leak
   ...
   obj:*/libGL.so*
}

{
   DDS/CycloneDDS
   Memcheck:Leak
   ...
   obj:*/libddsc.so*
}
```

---

## 22. Debugging and Profiling

### 22.1 Performance Profiling

```cpp
// profiler.cpp
#include <chrono>
#include <map>
#include <fmt/core.h>

namespace robot {

class Profiler {
    struct ProfileData {
        uint64_t total_time_us = 0;
        uint64_t call_count = 0;
        uint64_t min_time_us = UINT64_MAX;
        uint64_t max_time_us = 0;
    };
    
    static inline std::map<std::string, ProfileData> profiles;
    
public:
    class ScopedTimer {
        std::string name;
        std::chrono::high_resolution_clock::time_point start;
        
    public:
        explicit ScopedTimer(std::string n) 
            : name(std::move(n))
            , start(std::chrono::high_resolution_clock::now()) {}
        
        ~ScopedTimer() {
            auto end = std::chrono::high_resolution_clock::now();
            auto duration = std::chrono::duration_cast<std::chrono::microseconds>
                           (end - start).count();
            
            auto& data = profiles[name];
            data.total_time_us += duration;
            data.call_count++;
            data.min_time_us = std::min(data.min_time_us, 
                                        static_cast<uint64_t>(duration));
            data.max_time_us = std::max(data.max_time_us, 
                                        static_cast<uint64_t>(duration));
        }
    };
    
    static void print_report() {
        fmt::print("\n=== Performance Profile ===\n");
        fmt::print("{:<30} {:>10} {:>10} {:>10} {:>10} {:>10}\n",
                  "Function", "Calls", "Total(ms)", "Avg(us)", "Min(us)", "Max(us)");
        fmt::print("{:-<90}\n", "");
        
        for (const auto& [name, data] : profiles) {
            double total_ms = data.total_time_us / 1000.0;
            double avg_us = data.total_time_us / 
                           static_cast<double>(data.call_count);
            
            fmt::print("{:<30} {:>10} {:>10.2f} {:>10.2f} {:>10} {:>10}\n",
                      name, data.call_count, total_ms, avg_us,
                      data.min_time_us, data.max_time_us);
        }
    }
    
    static void reset() {
        profiles.clear();
    }
};

#ifdef ENABLE_PROFILING
    #define PROFILE(name) robot::Profiler::ScopedTimer _timer(name)
#else
    #define PROFILE(name) ((void)0)
#endif

// Usage
void example_function() {
    PROFILE("example_function");
    
    // Function body
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
}

} // namespace robot
```

---

## 23. Complete Robot Example

### 23.1 Full Robot System Implementation

```cpp
// complete_robot_system.cpp
#include <thread>
#include <atomic>
#include <memory>

namespace robot {

class CompleteRobotSystem {
public:
    // Configuration
    struct Config {
        int can_bus_id = 0;
        int dds_domain = 42;
        bool enable_gui = true;
        bool enable_logging = true;
        std::string config_file = "robot.yaml";
    };
    
private:
    Config config;
    
    // Core components
    std::unique_ptr<MotorController> motors[6];
    std::unique_ptr<TrajectoryGenerator> trajectory_gen;
    std::unique_ptr<DDSManager> dds;
    std::unique_ptr<Logger> logger;
    
    // Thread management
    std::thread control_thread;
    std::thread sensor_thread;
    std::thread comm_thread;
    std::thread gui_thread;
    
    // Communication
    SPSCQueue<SensorData, 128> sensor_queue;
    SPSCQueue<MotorCommand, 128> command_queue;
    Channel<SystemCommand> system_channel{10};
    
    // State
    std::atomic<bool> running{false};
    std::atomic<bool> emergency_stop{false};
    std::atomic<SystemState> state{SystemState::UNINITIALIZED};
    
    // GUI
    std::unique_ptr<ImGuiApplication> gui;
    std::unique_ptr<RobotControlPanel> control_panel;
    
public:
    explicit CompleteRobotSystem(const Config& cfg) : config(cfg) {}
    
    Result<void> initialize() {
        // Initialize logging
        if (config.enable_logging) {
            LogManager::initialize();
            logger = std::make_unique<Logger>("RobotSystem");
            logger->info("Initializing robot system");
        }
        
        // Initialize motors
        for (int i = 0; i < 6; ++i) {
            motors[i] = std::make_unique<MotorController>(i);
            if (auto result = motors[i]->initialize(); !result) {
                logger->error("Failed to initialize motor {}: {}",
                            i, to_string(result.error()));
                return result;
            }
        }
        
        // Initialize trajectory generator
        trajectory_gen = std::make_unique<TrajectoryGenerator>();
        
        // Initialize DDS
        if (config.dds_domain >= 0) {
            dds = std::make_unique<DDSManager>(config.dds_domain);
        }
        
        // Initialize GUI
        if (config.enable_gui) {
            gui = std::make_unique<ImGuiApplication>();
            if (auto result = gui->initialize(); !result) {
                logger->error("Failed to initialize GUI: {}",
                            to_string(result.error()));
                return result;
            }
            control_panel = std::make_unique<RobotControlPanel>();
        }
        
        state = SystemState::INITIALIZED;
        logger->info("Robot system initialized successfully");
        return Ok();
    }
    
    Result<void> start() {
        if (state != SystemState::INITIALIZED) {
            return Err<void>(ErrorCode::SYS_INVALID_STATE);
        }
        
        running = true;
        
        // Start threads
        control_thread = std::thread([this]() { control_loop(); });
        sensor_thread = std::thread([this]() { sensor_loop(); });
        comm_thread = std::thread([this]() { communication_loop(); });
        
        if (config.enable_gui) {
            gui_thread = std::thread([this]() { gui_loop(); });
        }
        
        state = SystemState::RUNNING;
        logger->info("Robot system started");
        return Ok();
    }
    
    Result<void> stop() {
        logger->info("Stopping robot system");
        
        // Signal threads to stop
        running = false;
        system_channel.close();
        
        // Join threads
        if (control_thread.joinable()) control_thread.join();
        if (sensor_thread.joinable()) sensor_thread.join();
        if (comm_thread.joinable()) comm_thread.join();
        if (gui_thread.joinable()) gui_thread.join();
        
        // Shutdown components
        for (auto& motor : motors) {
            motor->shutdown();
        }
        
        state = SystemState::STOPPED;
        logger->info("Robot system stopped");
        return Ok();
    }
    
private:
    void control_loop() {
        set_thread_name("control");
        configure_realtime_thread(99, 2);  // Priority 99, CPU 2
        
        const auto period = std::chrono::microseconds(1000);  // 1kHz
        auto next_wake = std::chrono::steady_clock::now();
        
        while (running) {
            PROFILE("control_loop");
            
            next_wake += period;
            std::this_thread::sleep_until(next_wake);
            
            // Check emergency stop
            if (emergency_stop) {
                for (auto& motor : motors) {
                    motor->emergency_stop();
                }
                continue;
            }
            
            // Read sensors
            SensorData sensor_data;
            if (sensor_queue.try_pop()) {
                process_sensor_data(sensor_data);
            }
            
            // Compute control
            auto commands = compute_control();
            
            // Send commands
            for (const auto& cmd : commands) {
                command_queue.try_push(cmd);
            }
            
            // Check deadline
            auto now = std::chrono::steady_clock::now();
            if (now > next_wake) {
                logger->warn("Control loop deadline miss");
            }
        }
    }
    
    void sensor_loop() {
        set_thread_name("sensors");
        
        while (running) {
            PROFILE("sensor_loop");
            
            SensorData data;
            
            // Read from all motors
            for (int i = 0; i < 6; ++i) {
                auto pos_result = motors[i]->get_position();
                auto vel_result = motors[i]->get_velocity();
                
                if (pos_result && vel_result) {
                    data.positions[i] = *pos_result;
                    data.velocities[i] = *vel_result;
                }
            }
            
            data.timestamp = get_timestamp_us();
            
            // Send to control thread
            sensor_queue.try_push(data);
            
            // Publish via DDS
            if (dds) {
                publish_sensor_data(data);
            }
            
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }
    
    void communication_loop() {
        set_thread_name("comm");
        
        if (!dds) return;
        
        // Set up DDS subscribers
        auto command_subscriber = dds->create_command_subscriber(
            [this](const robot_msgs::MotorCommand& cmd) {
                handle_dds_command(cmd);
            });
        
        while (running) {
            PROFILE("comm_loop");
            
            // Process system commands
            if (auto cmd = system_channel.try_receive()) {
                handle_system_command(*cmd);
            }
            
            // Process outgoing commands
            MotorCommand motor_cmd;
            while (command_queue.try_pop(motor_cmd)) {
                send_motor_command(motor_cmd);
            }
            
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }
    
    void gui_loop() {
        set_thread_name("gui");
        
        while (running && !gui->should_close()) {
            PROFILE("gui_loop");
            
            gui->begin_frame();
            
            // Render control panel
            control_panel->render();
            
            // Update visualization
            update_visualization();
            
            gui->end_frame();
        }
    }
    
    std::vector<MotorCommand> compute_control() {
        // Your control algorithm here
        std::vector<MotorCommand> commands;
        
        for (int i = 0; i < 6; ++i) {
            MotorCommand cmd;
            cmd.motor_id = i;
            cmd.mode = MotorCommand::POSITION;
            cmd.target = 0.0f;  // Computed target
            commands.push_back(cmd);
        }
        
        return commands;
    }
    
    void process_sensor_data(const SensorData& data) {
        // Process sensor data
    }
    
    void publish_sensor_data(const SensorData& data) {
        // Publish via DDS
    }
    
    void handle_dds_command(const robot_msgs::MotorCommand& cmd) {
        // Handle incoming DDS command
    }
    
    void handle_system_command(const SystemCommand& cmd) {
        // Handle system command
    }
    
    void send_motor_command(const MotorCommand& cmd) {
        if (cmd.motor_id >= 0 && cmd.motor_id < 6) {
            motors[cmd.motor_id]->execute_command(cmd);
        }
    }
    
    void update_visualization() {
        // Update 3D visualization
    }
    
    void set_thread_name(const std::string& name) {
        pthread_setname_np(pthread_self(), name.c_str());
    }
    
    void configure_realtime_thread(int priority, int cpu_core) {
        // Set CPU affinity
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(cpu_core, &cpuset);
        pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
        
        // Set real-time priority
        struct sched_param param;
        param.sched_priority = priority;
        pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
        
        // Lock memory
        mlockall(MCL_CURRENT | MCL_FUTURE);
    }
    
    uint64_t get_timestamp_us() {
        return std::chrono::steady_clock::now()
               .time_since_epoch()
               .count();
    }
};

} // namespace robot

// Main entry point
int main(int argc, char* argv[]) {
    using namespace robot;
    
    // Parse command line arguments
    CompleteRobotSystem::Config config;
    
    CLI::App app{"Robot Control System"};
    app.add_option("-c,--config", config.config_file, "Configuration file")
       ->check(CLI::ExistingFile);
    app.add_option("-d,--domain", config.dds_domain, "DDS domain ID");
    app.add_flag("--no-gui", config.enable_gui, "Disable GUI");
    app.add_flag("--no-log", config.enable_logging, "Disable logging");
    
    CLI11_PARSE(app, argc, argv);
    
    // Create and run robot system
    CompleteRobotSystem robot(config);
    
    if (auto result = robot.initialize(); !result) {
        fmt::print(stderr, "Failed to initialize: {}\n", 
                  to_string(result.error()));
        return 1;
    }
    
    if (auto result = robot.start(); !result) {
        fmt::print(stderr, "Failed to start: {}\n",
                  to_string(result.error()));
        return 1;
    }
    
    // Set up signal handlers
    std::signal(SIGINT, [](int) {
        fmt::print("\nShutdown requested\n");
        // Signal shutdown
    });
    
    // Wait for shutdown
    fmt::print("Robot system running. Press Ctrl+C to stop.\n");
    
    // Main thread can do other work or just wait
    while (true) {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        
        // Check for shutdown signal
        // ...
    }
    
    // Stop system
    robot.stop();
    
    // Print profiling report
    #ifdef ENABLE_PROFILING
    Profiler::print_report();
    #endif
    
    return 0;
}
```

---

## Appendix A: Quick Reference

### Thread Safety Cheat Sheet

| Pattern | Use Case | Example |
|---------|----------|---------|
| `std::mutex` | Protect shared data | Simple critical sections |
| `std::shared_mutex` | Many readers, few writers | Configuration data |
| `std::atomic` | Lock-free single values | Flags, counters |
| `SPSCQueue` | RT producer/consumer | Sensor data flow |
| `Channel` | Go-style communication | Commands |
| `std::condition_variable` | Thread synchronization | Producer-consumer |

### Memory Ordering

| Order | Use Case | Guarantees |
|-------|----------|------------|
| `memory_order_relaxed` | Counters | No synchronization |
| `memory_order_acquire` | Read synchronization | See all releases before |
| `memory_order_release` | Write synchronization | Visible to acquires |
| `memory_order_seq_cst` | Default, safest | Total order |

### Real-Time Checklist

- [ ] No dynamic allocation
- [ ] No system calls
- [ ] No mutex locks
- [ ] Fixed execution time
- [ ] Memory locked (mlockall)
- [ ] CPU affinity set
- [ ] RT priority configured
- [ ] Deadline monitoring

---

## Appendix B: Common Pitfalls

### 1. Data Races

```cpp
// ❌ BAD: Data race
int shared_data = 0;

void thread1() {
    shared_data++;  // Not atomic!
}

void thread2() {
    shared_data++;  // Race condition
}

// ✅ GOOD: Use atomic
std::atomic<int> shared_data{0};

void thread1() {
    shared_data.fetch_add(1);
}
```

### 2. Deadlocks

```cpp
// ❌ BAD: Potential deadlock
std::mutex m1, m2;

void thread1() {
    std::lock_guard<std::mutex> lock1(m1);
    std::lock_guard<std::mutex> lock2(m2);
}

void thread2() {
    std::lock_guard<std::mutex> lock2(m2);  // Different order!
    std::lock_guard<std::mutex> lock1(m1);
}

// ✅ GOOD: Lock together
void thread1() {
    std::scoped_lock lock(m1, m2);
}

void thread2() {
    std::scoped_lock lock(m1, m2);  // Same order
}
```

### 3. Real-Time Violations

```cpp
// ❌ BAD: Allocation in RT
void control_loop() {
    std::vector<float> data;  // Allocation!
    data.push_back(sensor_read());  // More allocation!
}

// ✅ GOOD: Pre-allocated
std::array<float, 100> data;
size_t index = 0;

void control_loop() {
    if (index < data.size()) {
        data[index++] = sensor_read();
    }
}
```

---

## Appendix C: Migration from Python

### Python to C++ Patterns

```python
# Python: List comprehension
squares = [x**2 for x in range(10) if x % 2 == 0]
```

```cpp
// C++: Simple loop
std::vector<int> squares;
for (int x = 0; x < 10; ++x) {
    if (x % 2 == 0) {
        squares.push_back(x * x);
    }
}
```

```python
# Python: Dictionary
config = {
    "motor_count": 6,
    "control_rate": 1000,
    "max_torque": 10.0
}
```

```cpp
// C++: Struct
struct Config {
    int motor_count = 6;
    int control_rate = 1000;
    float max_torque = 10.0;
};
```

```python
# Python: Context manager
with open("file.txt") as f:
    data = f.read()
```

```cpp
// C++: RAII
{
    std::ifstream file("file.txt");
    std::string data((std::istreambuf_iterator<char>(file)),
                     std::istreambuf_iterator<char>());
}  // File closed automatically
```

---

## Final Notes

This comprehensive guide provides:

1. **Safety through tooling** - Sanitizers, static analysis, and strict compiler flags
2. **Modern C++ patterns** - std::expected, smart pointers, structured bindings
3. **Practical examples** - Complete implementations for robotics systems
4. **Library integration** - fmt, spdlog, Catch2, Eigen, CycloneDDS, ImGui
5. **Threading patterns** - Lock-free queues, channels, thread pools
6. **Real-time guarantees** - Proper RT thread configuration and patterns
7. **Production readiness** - Build systems, testing, profiling, debugging

The guide balances safety with practicality, using established libraries rather than reinventing wheels, while maintaining clear patterns that Python developers can understand and follow.
