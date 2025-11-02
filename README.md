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

### 16.2 Advanced DDS Patterns

```cpp
// dds_advanced.cpp
namespace robot {

// Request-Reply pattern
class DDSService {
    using Request = robot_msgs::ServiceRequest;
    using Reply = robot_msgs::ServiceReply;
    
    dds::domain::DomainParticipant participant;
    dds::topic::Topic<Request> request_topic;
    dds::topic::Topic<Reply> reply_topic;
    
public:
    DDSService(int domain_id) : participant(domain_id) {
        request_topic = dds::topic::Topic<Request>(
            participant, "service_request");
        reply_topic = dds::topic::Topic<Reply>(
            participant, "service_reply");
    }
    
    // Service provider
    void provide_service(std::function<Reply(const Request&)> handler) {
        auto reader = dds::sub::DataReader<Request>(
            dds::sub::Subscriber(participant), request_topic);
        auto writer = dds::pub::DataWriter<Reply>(
            dds::pub::Publisher(participant), reply_topic);
        
        while (running) {
            auto samples = reader.read();
            
            for (const auto& sample : samples) {
                if (sample.info().valid()) {
                    Reply reply = handler(sample.data());
                    reply.request_id(sample.data().request_id());
                    writer.write(reply);
                }
            }
            
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    }
    
    // Service client
    std::future<Reply> call_service(const Request& request) {
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
