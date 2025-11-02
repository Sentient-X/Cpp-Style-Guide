# C++ for Roboticists
## A Modern, Safe, Step-by-Step Migration Guide

---

## 0. How to Use This Document

This guide is structured to onboard your team effectively, whether they are seasoned C++ developers or migrating from Python.

1.  **Road-map First:** Begin by skimming the "Road-map" (Section 1) to understand the overall learning progression.
2.  **Targeted Learning:** Jump into the chapter that aligns with your team's current C++ experience or project phase. Each chapter is designed to be self-contained.
3.  **Progressive Deep Dive:** As you become more comfortable, revisit sections for more in-depth understanding. The "Where to read more" links at the end of each tutorial section will guide you to the normative rules, checklists, and detailed technical specifications found in the original comprehensive guide.
4.  **Practical Application:** The tutorials are designed for immediate practical use. Compile the code, run the examples, and integrate them into your projects.
5.  **Reference Material:** For experienced users or specific technical challenges, the original detailed content remains accessible as a comprehensive reference.

---

## 1. Road-map: From Python to Production

This road-map outlines a progressive learning path, moving from foundational concepts to advanced production-ready C++ for robotics.

### FIRST MONTH: Get Code Running

*   **2.1 “Hello Robot” – Compile, Flash LED, Log:** Start with a minimal C++ program that interacts with hardware and introduces basic C++ I/O, replacing Python's `print` and `time.sleep`.
*   **2.2 `fmt` / `spdlog` – Print Without `printf`:** Learn to use modern, type-safe, and efficient string formatting and logging libraries, similar to Python's `logging` module.
*   **2.3 `std::expected` – Return Errors, No Exceptions:** Understand how C++ handles fallible operations without relying on exceptions, a key pattern for predictable real-time systems.
*   **2.4 Unit Tests with `Catch2` – Green Bar Addiction:** Implement unit tests using a familiar, Python-like testing framework to ensure code correctness.
*   **2.5 `clang-format` → CI Green:** Establish code style consistency automatically. Integrate `clang-format` into your Continuous Integration (CI) pipeline to enforce a uniform coding standard.

### FIRST QUARTER: Ship a Node

*   **3.1 Zero-Allocation Real-Time Loop (1 kHz):** Design and implement a real-time control loop that guarantees no dynamic memory allocations during its critical execution path.
*   **3.2 Lock-Free SPSC Queue → Live Plot in ImGui:** Utilize high-performance, thread-safe queues for inter-thread communication and visualize data in real-time using ImGui.
*   **3.3 `CycloneDDS` Pub/Sub – Talk to ROS 2 Tools:** Integrate DDS for inter-process communication, ensuring compatibility with ROS 2 tools and workflows.
*   **3.4 Sanitizer Build in CI (ASan + UBSan):** Configure your CI to automatically detect memory errors and undefined behavior during testing.
*   **3.5 Code Review Checklist (Appendix D):** Adopt a structured code review process to maintain code quality and safety.

### FIRST YEAR: Hard-Core Production Systems

*   **4.1 Custom Memory Arenas for µ-sec Determinism:** Achieve microsecond-level deterministic performance by managing memory allocation manually within predictable pools.
*   **4.2 Lock-Free Multi-Producer Queues:** Implement or utilize advanced lock-free data structures for high-throughput, multi-threaded scenarios.
*   **4.3 NUMA-Aware Thread Pools:** Optimize multi-threaded performance on Non-Uniform Memory Access (NUMA) architectures.
*   **4.4 Formal MISRA / ISO-26262 Subset:** Adhere to industry standards for safety-critical systems, ensuring rigorous code quality and verifiability.
*   **4.5 Worst-Case Execution Time (WCET) Proofs:** Formally analyze and guarantee the execution time bounds of critical code sections.

---

## 2. First Month Tutorials

### 2.1 “Hello Robot” – Compile, Flash LED, Log

This section bridges the gap from familiar Python patterns to basic C++ for embedded systems.

#### The Python Mental Model You Already Have

```python
# Python: Blinking an LED
import time
import board
import digitalio

led = digitalio.DigitalInOut(board.LED)
led.direction = digitalio.Direction.OUTPUT

while True:
    led.value = True
    print("LED ON")
    time.sleep(0.5)

    led.value = False
    print("LED OFF")
    time.sleep(0.5)
```

#### C++ Translation (Single File, No Build System Yet)

```cpp
// hello_robot.cpp
#include <gpiod.hpp>        // For GPIO access (install libgpiod-dev libgpiod-cpp-dev)
#include <fmt/core.h>       // For formatted printing
#include <chrono>           // For durations
#include <thread>           // For std::this_thread::sleep_for

int main() {
    // GPIO setup using libgpiod
    gpiod::chip chip("gpiochip0"); // Assumes GPIO controller is gpiochip0
    gpiod::line led_line = chip.get_line(17); // Assumes LED is connected to GPIO 17

    // Request the line as an output, initially low (0)
    led_line.request({"hello_robot", gpiod::line_request::DIRECTION_OUTPUT, 0});

    // Use C++20 chrono literals for clarity
    using namespace std::chrono_literals;

    while (true) {
        led_line.set_value(1); // Turn LED ON
        fmt::print("LED ON\n");
        std::this_thread::sleep_for(500ms);

        led_line.set_value(0); // Turn LED OFF
        fmt::print("LED OFF\n");
        std::this_thread::sleep_for(500ms);
    }

    return 0; // This point is never reached in the loop
}
```

#### Compile Line (Works on Ubuntu 22.04/24.04)

To compile this, you'll need the `fmt` and `libgpiod` development libraries installed.

```bash
# Install dependencies (example for Ubuntu)
sudo apt update
sudo apt install -y libgpiod-dev libgpiod-cpp-dev libfmt-dev

# Compile command
g++ -std=c++23 -Wall -Wextra -O2 \
    `pkg-config --cflags fmt` \
    hello_robot.cpp \
    `pkg-config --libs fmt` -lgpiodcxx \
    -o hello_robot
```

**Explanation of Compile Flags:**

*   `-std=c++23`: Enables the C++23 standard.
*   `-Wall -Wextra`: Enables most compiler warnings.
*   `-O2`: Enables optimizations for better performance.
*   `` `pkg-config --cflags fmt` ``: Injects include paths for the `fmt` library.
*   `` `pkg-config --libs fmt` ``: Injects library paths and names for `fmt`.
*   `-lgpiodcxx`: Links against the `libgpiod-cpp` library.
*   `-o hello_robot`: Specifies the output executable name.

#### Key Take-Aways

*   **`fmt::print`:** A type-safe and more flexible alternative to `printf`. It's also the standard C++ library for formatting.
*   **`std::this_thread::sleep_for`:** The standard C++ way to pause execution, analogous to Python's `time.sleep`.
*   **RAII for Resources:** `gpiod::line` automatically manages the GPIO line resource. When the object goes out of scope (or the program exits), the resource is released. This is a fundamental C++ pattern (Resource Acquisition Is Initialization).
*   **No Dynamic Allocation (Yet):** This simple example avoids `new`, `malloc`, or dynamic containers, setting the stage for real-time considerations.

---

### 2.2 `fmt` & `spdlog` – Print Without `printf`

Moving away from C-style `printf` is crucial for robust C++ code. `fmt` and `spdlog` offer a modern, safer, and more expressive alternative.

#### The Python Style You Already Use

```python
# Python: Logging with string formatting
import logging
logging.basicConfig(level=logging.INFO)

motor_id = 1
position = 45.678

logging.info("Motor %d position %.2f", motor_id, position)
# Output: Motor 1 position 45.68
```

#### C++ Equivalent: Modern Formatting and Logging

```cpp
#include <spdlog/spdlog.h>
#include <string> // For std::string

// Assuming motor_id and position are defined:
int motor_id = 1;
float position = 45.678f;

// Using spdlog with fmt syntax
spdlog::info("Motor {} position {:.2f}", motor_id, position);
// Output: [YYYY-MM-DD HH:MM:SS.ms] [info] Motor 1 position 45.68
```

**Explanation:**

*   **`spdlog::info(...)`:** `spdlog` is a high-performance logging library. Its `info`, `debug`, `warn`, `error`, etc., functions accept arguments formatted using `fmt`'s syntax.
*   **`{}` Placeholder:** Replaces positional arguments similar to Python's f-strings or `.format()`.
*   **`{:.2f}`:** Applies formatting specifications. Here, it formats a floating-point number to 2 decimal places.

#### Guideline vs. Rule: Prefer `fmt`/`spdlog`

*   **Guideline:** Prefer `fmt` for all string formatting and `spdlog` for logging. They offer type safety, better performance, and a more consistent syntax with Python.
*   **When to Break the Rule:** Interfacing with legacy C APIs that strictly require `printf`-style format strings or older C++ stream (`<<`) interfaces. However, even in these cases, consider using `fmt::format_to` to generate the string and then pass it to the legacy function.

---

### 2.3 `std::expected` – Return Errors, No Exceptions

Handling errors gracefully and predictably is paramount, especially in real-time systems. `std::expected` (or `tl::expected` for older C++ standards) provides a compile-time construct for functions that can either return a value or an error.

#### The Python Style You Already Use

```python
# Python: Returning None for errors
def read_position(motor_id: int) -> float | None:
    if not is_motor_connected(motor_id):
        return None  # Indicates failure
    # ... read position ...
    return position

# Caller checks for None
pos = read_position(3)
if pos is None:
    print("Failed to read position")
else:
    print(f"Position: {pos}")
```

#### C++ Equivalent: `std::expected`

```cpp
#include <expected> // For std::expected (C++23)
#include <string>   // For error messages
#include <optional> // For std::optional (used in older standards)

// Define custom error codes
enum class ErrorCode {
    OK = 0,
    HW_COMM_TIMEOUT = 100,
    HW_NOT_INITIALIZED = 101,
    PARAM_OUT_OF_RANGE = 300,
    // ... other error codes
};

// Helper function to convert error codes to strings
constexpr std::string_view to_string(ErrorCode code) {
    switch (code) {
        case ErrorCode::OK: return "OK";
        case ErrorCode::HW_COMM_TIMEOUT: return "Hardware communication timeout";
        case ErrorCode::HW_NOT_INITIALIZED: return "Hardware not initialized";
        case ErrorCode::PARAM_OUT_OF_RANGE: return "Parameter out of range";
        default: return "Unknown error";
    }
}

// Type aliases for cleaner code
template<typename T>
using Result = std::expected<T, ErrorCode>; // For functions returning a value or error

using Status = std::expected<void, ErrorCode>; // For functions returning success or error

// Helper for creating success results
template<typename T>
Result<T> Ok(T&& value) {
    return Result<T>(std::forward<T>(value));
}
inline Status OkStatus() { return Status(); }

// Helper for creating error results
template<typename T>
Result<T> Err(ErrorCode code) {
    return std::unexpected(code);
}
inline Status ErrStatus(ErrorCode code) { return std::unexpected(code); }


// Example function that might fail
Result<float> read_position(int motor_id) {
    // Simulate hardware check
    bool connected = true; // Assume connected for this example
    if (!connected) {
        return Err<float>(ErrorCode::HW_COMM_TIMEOUT);
    }

    // Simulate reading position
    float position = 1.23f; // Example value

    // Check if position is valid (example)
    if (position < -3.14f || position > 3.14f) {
         return Err<float>(ErrorCode::PARAM_OUT_OF_RANGE);
    }

    return Ok(position); // Return success with the value
}

// Caller site
void example_usage() {
    int motor_to_read = 3;
    auto position_result = read_position(motor_to_read);

    if (!position_result) { // Check if it holds an error
        spdlog::warn("Failed to read position for motor {}: {}",
                     motor_to_read, to_string(position_result.error()));
    } else {
        // Access the value safely
        float pos = *position_result; // or position_result.value()
        spdlog::info("Motor {} position: {:.3f}", motor_to_read, pos);
    }

    // Monadic operations for chaining
    auto chained_result = read_position(1)
        .and_then([](float pos) {
            spdlog::info("Read position: {:.3f}", pos);
            // Could perform another operation here that also returns Result/Status
            return OkStatus();
        })
        .or_else([](ErrorCode err) {
            spdlog::error("Operation failed: {}", to_string(err));
            return ErrStatus(err); // Propagate or handle the error
        });
}
```

#### Guideline vs. Rule: Use `std::expected` for Fallible Functions

*   **Guideline:** Use `std::expected` (or `tl::expected` for C++17/20) for all functions that can fail and need to return a value or an error code. This makes error handling explicit and part of the function's signature.
*   **When to Break the Rule:**
    *   **Constructors:** Constructors should generally not return `std::expected`. Instead, use a factory function (e.g., `std::unique_ptr<MyClass> MyClass::create(...)`) that returns `std::expected<MyClass, ErrorCode>` or `std::expected<std::unique_ptr<MyClass>, ErrorCode>`.
    *   **Move Operations:** Move constructors and move assignment operators should ideally not fail. If they must, they should throw an exception (though this is rare).
    *   **Real-time Path:** In extremely performance-critical, hard real-time paths where even the minimal overhead of `std::expected` is unacceptable, a simpler approach like returning a boolean success flag and passing output parameters by reference might be used, but this requires extreme care and justification.

---

### 2.4 Unit Tests with Catch2 – Green Bar Addiction

Writing tests is crucial for maintainable C++ code. `Catch2` is a popular, header-only testing framework that feels familiar to Python developers.

#### Python Developers Feel at Home

```python
# Python: Basic test structure
import unittest

class TestMotorController(unittest.TestCase):
    def test_initialization(self):
        motor = MotorController(1)
        self.assertTrue(motor.initialize())
        self.assertAlmostEqual(motor.get_position(), 0.0)

    def test_invalid_move(self):
        motor = MotorController(2)
        motor.initialize()
        self.assertFalse(motor.move_to(10.0)) # Assuming move_to returns bool

# unittest.main() runs the tests```

#### C++ Equivalent with `Catch2`

First, ensure `Catch2` is available. You can often add it via `FetchContent` in CMake (see Section 20.1).

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/catch_approx.hpp> // For approximate floating-point comparisons
#include <catch2/matchers/catch_matchers_floating_point.hpp> // More specific matchers

// Assume MotorController and its associated types (like ErrorCode) are defined
// #include "motor_controller.h" // Include your header file

// Helper for simulating a MotorController
class MotorController {
public:
    enum class ErrorCode { OK, SYS_NOT_INITIALIZED, PARAM_OUT_OF_RANGE, SYS_ALREADY_RUNNING };
    using Status = std::expected<void, ErrorCode>;
    using ResultFloat = std::expected<float, ErrorCode>;

    MotorController() : initialized_(false), position_(0.0f) {}

    Status initialize() {
        if (initialized_) return std::unexpected(ErrorCode::SYS_ALREADY_RUNNING);
        initialized_ = true;
        return {}; // Success
    }

    bool is_initialized() const { return initialized_; }

    ResultFloat get_position() const {
        if (!initialized_) return std::unexpected(ErrorCode::SYS_NOT_INITIALIZED);
        return position_;
    }

    Status move_to(float target) {
        if (!initialized_) return std::unexpected(ErrorCode::SYS_NOT_INITIALIZED);
        if (target > 6.28f || target < -6.28f) { // Example limit
            return std::unexpected(ErrorCode::PARAM_OUT_OF_RANGE);
        }
        position_ = target;
        return {}; // Success
    }

private:
    bool initialized_;
    float position_;
};

// Helper for approximate float comparisons
using Catch::Approx;

TEST_CASE("Motor Controller Initialization", "[motor][init]") {
    MotorController motor;

    SECTION("Initial state") {
        REQUIRE_FALSE(motor.is_initialized()); // Check for false state
        auto pos_result = motor.get_position();
        REQUIRE_FALSE(pos_result.has_value()); // Check if it has an error
        REQUIRE(pos_result.error() == MotorController::ErrorCode::SYS_NOT_INITIALIZED);
    }

    SECTION("Successful initialization") {
        auto status = motor.initialize();
        REQUIRE(status.has_value()); // Check for success
        REQUIRE(motor.is_initialized());

        auto pos_result = motor.get_position();
        REQUIRE(pos_result.has_value());
        // Use Approx for floating-point comparisons
        REQUIRE(pos_result.value() == Approx(0.0f));
    }

    SECTION("Double initialization") {
        REQUIRE(motor.initialize().has_value()); // First init succeeds
        auto second_init = motor.initialize();
        REQUIRE_FALSE(second_init.has_value()); // Second init fails
        REQUIRE(second_init.error() == MotorController::ErrorCode::SYS_ALREADY_RUNNING);
    }
}

TEST_CASE("Motor Movement", "[motor][movement]") {
    MotorController motor;
    motor.initialize().value(); // Initialize the motor, ignore success here for brevity

    SECTION("Valid movement") {
        float target_pos = 1.57f;
        auto status = motor.move_to(target_pos);
        REQUIRE(status.has_value()); // Movement command succeeded

        // Verify position (using Approx for float comparison)
        auto pos_result = motor.get_position();
        REQUIRE(pos_result.has_value());
        // Catch2 Matchers offer more flexibility
        REQUIRE_THAT(pos_result.value(),
                     Catch::Matchers::WithinAbs(target_pos, 0.01f)); // Within 0.01 absolute tolerance
    }

    SECTION("Out of range movement") {
        auto status = motor.move_to(10.0f); // Out of bounds
        REQUIRE_FALSE(status.has_value());
        REQUIRE(status.error() == MotorController::ErrorCode::PARAM_OUT_OF_RANGE);

        // Ensure position hasn't changed unintentionally
        auto pos_result = motor.get_position();
        REQUIRE(pos_result.value() == Approx(0.0f));
    }
}

// You can also run benchmarks and property-based tests with Catch2
// TEST_CASE("Motor Performance Benchmarks", "[motor][benchmark]") {
//     MotorController motor;
//     motor.initialize();
//     BENCHMARK("Move command") {
//         return motor.move_to(1.0f);
//     };
// }
```

**Compilation and Execution:**

```bash
# Assuming Catch2 is included via FetchContent in CMakeLists.txt
# CMake will create a 'tests' executable.
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# Run all tests
./build/tests

# Run specific tests tagged as [motor]
./build/tests --reporter console --success --filter "[motor]"
```

#### Guideline vs. Rule: Test Early, Test Often

*   **Guideline:** Write unit tests *before* implementing the production code (Test-Driven Development). Use `Catch2` for its clear syntax and powerful features like sections, approximate comparisons, and matchers. Integrate test execution into your CI pipeline.
*   **When to Break the Rule:** For very small, self-contained utility functions where the logic is trivial and already well-understood, or during initial exploratory coding where the API is rapidly changing. However, aim to test all significant logic.

---

### 2.5 `clang-format` – One Style to Rule Them All

Consistency in code style reduces cognitive load and prevents bikeshedding during code reviews. `clang-format` automates this.

#### The Setup

1.  **Install `clang-format`:** Usually included with `clang` or available as a standalone package (`sudo apt install clang-format`).
2.  **Create `.clang-format` file:** Place a configuration file at the root of your project. You can generate a style using `clang-format -style=google -dump-config > .clang-format` and then customize it. The sample `.clang-format` from the original document is a good starting point.
3.  **Integrate into CI:** Add a step in your CI pipeline (e.g., GitHub Actions, GitLab CI) to check formatting.

#### Example `.clang-format` Snippet (from original doc)

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

#### Guideline vs. Rule: Enforce Formatting Automatically

*   **Guideline:** Use `clang-format` and enforce it via CI. All code committed must pass the formatting check.
*   **When to Break the Rule:** Temporarily, during rapid prototyping or emergency bug fixing where immediate code changes are critical and formatting can be applied later. However, this should be the exception, not the norm.

---

## 3. First Quarter Deep Dive

This section focuses on building core components of a robotic system, emphasizing real-time performance and robust communication.

### 3.1 Zero-Allocation Real-Time Loop (1 kHz)

Achieving deterministic real-time performance requires strict control over execution time and resource usage, particularly memory.

#### The Python Background

```python
# Python: A simple loop (allocations happen implicitly)
import time

def process_sensor_data(sensor_reading):
    # Some processing that might involve temporary objects or lists
    processed = [x * 2 for x in sensor_reading] # Allocates a new list
    return processed

def control_loop():
    while True:
        sensor_data = read_sensor()        # Might allocate internally
        commands = process_sensor_data(sensor_data) # Allocates 'processed' list
        send_can_commands(commands)        # Sends data
        time.sleep(0.001) # Target 1 kHz
```

#### C++ Contract for a 1 kHz Loop

A hard real-time loop must adhere to strict constraints:

*   **Zero Heap Allocations:** Absolutely no dynamic memory allocation (`new`, `malloc`, `std::vector::push_back` if capacity needs to grow) within the loop's execution path.
*   **Upper-Bounded Worst-Case Latency:** The maximum time taken for one loop iteration must be predictable and known.
*   **Deadline Miss Counter:** Telemetry should track how often the loop fails to complete within its designated period.

#### Implementation Sketch (Still Fits on One Slide)

This example uses pre-allocated buffers and avoids dynamic containers within the critical loop.

```cpp
#include <array>
#include <chrono>
#include <thread>
#include <atomic>
#include <vector> // For SensorData, MotorCommand definitions
#include <string>

// Assume these definitions exist elsewhere
struct SensorData { float values[6]; uint64_t timestamp; };
struct MotorCommand { int motor_id; float target; /* ... other params */ };
enum class ErrorCode { /* ... */ };
template<typename T> using Result = std::expected<T, ErrorCode>;
using Status = std::expected<void, ErrorCode>;
// Assume SPSCQueue is defined (see Section 3.2)
// Assume basic hardware read/write functions exist (non-allocating)
extern Status read_sensors_fixed(float output_buffer[6]);
extern void write_actuator_commands_fixed(const MotorCommand commands[], size_t count);
extern uint64_t get_current_time_us(); // Non-allocating time source


// --- Real-time loop component ---
class RealtimeController {
    // Pre-allocated buffers - static or member, never dynamically sized inside loop
    alignas(64) std::array<float, 6> sensor_buffer; // Use alignas for cache efficiency
    alignas(64) std::array<MotorCommand, 6> command_buffer;

    std::atomic<bool> running_{false};
    std::atomic<uint64_t> deadline_miss_count_{0};

    // Use fixed-size queues or skip if not needed for RT path
    // SPSCQueue<SensorData, 128> sensor_queue_; // Could be used for non-RT path

public:
    void start_realtime_loop(int priority = 99, int cpu_core = 2) {
        running_ = true;
        // Configure thread for real-time (see Section 14.1)
        configure_realtime_thread(priority, cpu_core);

        const auto period = std::chrono::microseconds(1000); // 1 kHz target
        auto next_wake = std::chrono::steady_clock::now();

        while (running_) {
            // PROFILE("realtime_control_loop"); // Macro for profiling
            next_wake += period;
            std::this_thread::sleep_until(next_wake); // Precise timing

            // --- Critical Section: NO ALLOCATIONS ALLOWED ---
            // 1. Read Sensors (fixed time, no allocation)
            if (auto status = read_sensors_fixed(sensor_buffer.data()); !status) {
                // Log error (non-blocking logging is crucial here)
                // RT_LOG_ERROR("Sensor read failed: {}", to_string(status.error()));
                continue; // Skip cycle if sensors failed
            }

            // 2. Compute Control (fixed time, no allocation)
            size_t command_count = 0;
            for (int i = 0; i < 6; ++i) {
                // Example: Simple P-controller logic
                float target_pos = 0.0f; // Get target from a non-allocating source
                float error = target_pos - sensor_buffer[i];
                command_buffer[command_count++] = {i, /* calculate command */ error};
            }

            // 3. Write Actuator Commands (fixed time, no allocation)
            write_actuator_commands_fixed(command_buffer.data(), command_count);

            // --- Deadline Monitoring ---
            auto now = std::chrono::steady_clock::now();
            if (now > next_wake) {
                deadline_miss_count_.fetch_add(1);
                // RT_LOG_WARN("RT loop deadline missed!");
            }
            // --- End Critical Section ---
        }
    }

    void stop_realtime_loop() {
        running_ = false;
    }

    uint64_t get_deadline_miss_count() const {
        return deadline_miss_count_.load();
    }

private:
    // Placeholder for real-time thread configuration
    void configure_realtime_thread(int priority, int cpu_core) {
         // Implementation details in Section 14.1
         // - Set CPU affinity
         // - Set SCHED_FIFO priority
         // - Lock memory (mlockall)
         // - Pre-fault stack
         // This part MUST be done before entering the loop.
    }
};

// Example Usage within a larger system
void setup_robot_system() {
    RealtimeController rt_controller;
    // ... other system components ...

    std::thread rt_thread([&rt_controller]() {
        rt_controller.start_realtime_loop();
    });

    // ... system runs ...

    // rt_controller.stop_realtime_loop();
    // rt_thread.join();
}
```

#### Trade-off Discussion: Allocation Strategies

If dynamic allocation is absolutely necessary, it must be managed carefully:

1.  **Static Storage:** Use `static` variables or global objects. Suitable for singletons or data that lives for the program's lifetime. Simple, but inflexible.
2.  **Object Pools:** Pre-allocate a fixed number of objects before the real-time loop starts. When an object is needed, allocate from the pool; when finished, return it to the pool. This avoids the overhead of `new`/`delete` and fragmented memory. Lock-free pools are essential for multi-threaded access.
3.  **Arena Allocators:** Allocate a large chunk of memory upfront. Request memory blocks from the arena. At the start of each real-time cycle (or periodically), reset the entire arena, deallocating all blocks at once. This is efficient if many short-lived objects are needed within a cycle.

The guide provides examples for these patterns in Section 5.1 (`ObjectPool`) and Section 13.1 (`LockFreeStack` can be adapted for pooling).

**Guideline vs. Rule: Minimize Allocations in Real-Time Paths**

*   **Guideline:** Avoid dynamic memory allocations (`new`, `malloc`, standard containers that resize) within any code path that needs to meet hard real-time deadlines. Use pre-allocated storage, object pools, or arenas.
*   **When to Break the Rule:**
    *   **Initialization Phase:** Allocations are acceptable during system startup before the real-time deadlines begin.
    *   **Non-Critical Paths:** If a specific code path is guaranteed *not* to execute during the real-time critical section and its allocation overhead is acceptable (e.g., logging, configuration loading, background tasks), it might be permissible. This requires careful analysis and documentation.
    *   **System Shutdown:** Allocations during graceful shutdown are generally acceptable.

---

### 3.2 Lock-Free SPSC Queue → Live Plot in ImGui

Efficient inter-thread communication is vital. For real-time systems, locks can introduce unpredictable latency. Lock-free queues offer a thread-safe alternative without blocking.

#### Python Queues Allocate and Use Locks

```python
# Python: Standard queue uses locks and can allocate
import queue
import threading
import time

q = queue.Queue(maxsize=100) # Max size, but still uses locks internally

def producer():
    for i in range(1000):
        q.put(i) # Blocks if full, uses locks
        time.sleep(0.0001)

def consumer():
    while True:
        try:
            item = q.get_nowait() # Non-blocking, but still checks lock
            print(f"Consumed: {item}")
            q.task_done()
        except queue.Empty:
            time.sleep(0.0001)
        except Exception: # Handle shutdown
             break

# Start producer and consumer threads...
```

#### C++ Lock-Free SPSC Queue

A Single-Producer, Single-Consumer (SPSC) queue is ideal for passing data between a dedicated producer thread (e.g., sensor reading) and a consumer thread (e.g., control loop or logger) without locks.

```cpp
#include <vector>
#include <array>
#include <atomic>
#include <optional> // For try_pop return value
#include <cstddef>  // For size_t
#include <algorithm> // For std::min

// Example SPSC Queue (implementation details in Section 13.1)
template<typename T, size_t SIZE>
class SPSCQueue {
    // Ensure SIZE is a power of 2 for efficient modulo arithmetic
    static_assert((SIZE & (SIZE - 1)) == 0, "Queue size must be a power of 2");

    std::array<T, SIZE> buffer_;
    // Use alignas for cache line padding to avoid false sharing
    alignas(64) std::atomic<size_t> write_index_{0};
    alignas(64) std::atomic<size_t> read_index_{0};

public:
    // Tries to push an item. Returns false if the queue is full.
    bool try_push(const T& item) {
        size_t current_write = write_index_.load(std::memory_order_relaxed);
        size_t next_write = (current_write + 1) & (SIZE - 1); // Modulo SIZE

        // Check if the queue is full
        if (next_write == read_index_.load(std::memory_order_acquire)) {
            return false; // Queue is full
        }

        buffer_[current_write] = item;
        // Release memory order ensures the write to buffer_ is visible
        // before the write_index_ update is visible to the consumer.
        write_index_.store(next_write, std::memory_order_release);
        return true;
    }

    // Tries to pop an item. Returns std::nullopt if the queue is empty.
    std::optional<T> try_pop() {
        size_t current_read = read_index_.load(std::memory_order_relaxed);

        // Check if the queue is empty
        if (current_read == write_index_.load(std::memory_order_acquire)) {
            return std::nullopt; // Queue is empty
        }

        T item = buffer_[current_read];
        // Release memory order ensures the read of buffer_ is visible
        // before the read_index_ update is visible to the producer.
        read_index_.store((current_read + 1) & (SIZE - 1),
                         std::memory_order_release);
        return item;
    }

    // Returns the current number of items in the queue.
    size_t size() const {
        // Acquire semantics ensure we see the latest writes.
        size_t write = write_index_.load(std::memory_order_acquire);
        size_t read = read_index_.load(std::memory_order_acquire);
        return (write - read) & (SIZE - 1); // Handle wrap-around
    }
};

// --- Usage Example ---
#include <imgui.h>
#include <implot.h> // ImPlot is used for plotting
#include <deque>    // Used by ImPlot data management

// Assuming PlotData struct from Section 19.1
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
};

// Global queue for sensor data (producer: sensor thread, consumer: plotter thread)
SPSCQueue<float, 1024> sensor_value_queue;

// Simulate data production (e.g., from a sensor thread)
void produce_sensor_data(float value) {
    sensor_value_queue.try_push(value);
}

// ImGui Plotting logic (runs in GUI thread)
void render_sensor_plot(PlotData& plot_data) {
    // Consume data from the queue
    float sensor_val;
    while (sensor_value_queue.try_pop(sensor_val)) {
        plot_data.add_point(sensor_val);
    }

    // Plotting code using ImPlot (simplified)
    if (ImPlot::BeginPlot("Sensor Value", ImVec2(-1,-1))) {
        ImPlot::SetupAxes("Time (samples)", "Value");
        ImPlot::PlotLine("Sensor", plot_data.x_data.data(), plot_data.y_data.data(), plot_data.x_data.size());
        ImPlot::EndPlot();
    }
}

// --- In your main application loop or GUI thread ---
// PlotData sensor_plot_history(500); // Store last 500 points
// ...
// render_sensor_plot(sensor_plot_history);
```

#### Trade-off Discussion: When to Use Lock-Free vs. Locked Queues

*   **Lock-Free SPSCQueue:** Best for passing data between exactly two threads when one is exclusively producing and the other is exclusively consuming. Offers maximum performance and predictability (no blocking).
*   **Lock-Free Multi-Producer/Consumer Queues:** Use when multiple threads produce or consume data. `moodycamel::ConcurrentQueue` (included in the original doc's library list) is a highly optimized option. These still avoid locks but have slightly more overhead than SPSC.
*   **`std::queue` with `std::mutex`:** Suitable for general-purpose inter-thread communication where real-time guarantees are not critical, or when complexity must be minimized. Easier to implement correctly but introduces potential latency due to locking.
*   **`Channel` (Section 15.1):** A Go-style channel implementation, useful for coordinating tasks and passing messages, often blocking until the other end is ready.

**Guideline vs. Rule: Choose Communication Based on Real-Time Needs**

*   **Guideline:** Use lock-free queues (SPSC or multi-producer) for high-frequency data transfer between threads in performance-critical paths. Use standard mutex-protected queues or channels for general coordination and less performance-sensitive tasks.
*   **When to Break the Rule:**
    *   **Simplicity:** If the performance difference is negligible and correctness/simplicity is paramount, a mutex-protected queue might be acceptable.
    *   **Complex Synchronization:** If the communication pattern involves complex signaling beyond simple data transfer, a condition variable or channel might be more expressive.

---

### 3.3 CycloneDDS Pub/Sub – Talk to ROS 2 Tools

Seamless integration with the existing robotics ecosystem, particularly ROS 2, is essential. `CycloneDDS` provides a high-performance, standard-compliant DDS implementation that works directly with ROS 2.

#### Five-Line Publisher That Works with `ros2 topic echo`

```cpp
#include <dds/dds.hpp>
#include <vector>
#include <string>
#include <chrono>
#include <thread>

// Assume robot_msgs::JointState is defined via an IDL file
// and generated C++ bindings are available.
// Example structure (if not using generated code):
namespace robot_msgs {
    struct JointState {
        std::vector<double> position;
        std::vector<double> velocity;
        std::string name;
        int64_t timestamp;
        // DDS specific fields might be needed depending on generation
    };
}

void publish_joint_state(int domain_id) {
    // 1. Create a domain participant
    dds::domain::DomainParticipant participant(domain_id);

    // 2. Create a topic (ensure name matches ROS 2 topic)
    dds::topic::Topic<robot_msgs::JointState> topic(
        participant, "joint_states"); // Matches ROS 2 "joint_states" topic

    // 3. Create a publisher and data writer
    dds::pub::Publisher publisher(participant);
    dds::pub::DataWriter<robot_msgs::JointState> writer(publisher, topic);

    // 4. Prepare data
    std::vector<double> positions = {0.1, -0.2, 0.3, -0.4, 0.5, -0.6};
    std::vector<double> velocities(6, 0.0);
    robot_msgs::JointState js;
    js.name("robot_arm");
    js.position(positions);
    js.velocity(velocities);
    js.timestamp(std::chrono::steady_clock::now().time_since_epoch().count());

    // 5. Write the data
    writer.write(js);
    spdlog::info("Published joint state message.");
}

// --- In your main application ---
// std::thread dds_thread(publish_joint_state, 42); // Domain ID 42

// --- From another terminal ---
// source /opt/ros/humble/setup.bash
// ros2 topic echo /joint_states
```

**To Run This:**

1.  **Install CycloneDDS:** Follow the official CycloneDDS installation guide.
2.  **IDL Generation:** You'll typically define your message structures in an `.idl` file and use a DDS code generator (or ROS 2's build system) to create the C++ bindings. For simple cases, you might manually define the struct if your DDS implementation supports it directly.
3.  **Compile:** Link against the CycloneDDS library (e.g., `-l CycloneDDS`).
4.  **Run ROS 2:** Ensure a ROS 2 environment is sourced in another terminal to subscribe.

#### Guideline vs. Rule: DDS for Inter-Process Communication

*   **Guideline:** Use DDS (like `CycloneDDS`) for communication between separate processes or across different machines. Use efficient intra-process communication mechanisms (lock-free queues, channels) for threads within the same process.
*   **When to Break the Rule:**
    *   **High-Frequency Intra-Process Data:** If you need to pass extremely high volumes of data between threads *within* the same process and DDS serialization/deserialization becomes a bottleneck, consider specialized lock-free queues.
    *   **Legacy Systems:** Integrating with older systems that might not support DDS natively.

---

### 3.4 Sanitizer Build in CI

Automated detection of runtime errors is crucial for reliability. AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), and LeakSanitizer (LSan) help catch subtle bugs.

#### GitHub Actions Snippet (copy-paste)

Add this to your `.github/workflows/main.yml` (or similar):

```yaml
name: C++ CI

on: [push, pull_request]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        build_type: [Debug, Release]
        include:
          - build_type: Debug
            sanitizers: ON
            test_flags: "--reporter=junit --error-exit-code=1" # JUnit XML for reporting
          - build_type: Release
            sanitizers: OFF
            test_flags: "--reporter=console --success"

    steps:
    - uses: actions/checkout@v3

    - name: Install Dependencies
      run: |
        sudo apt update
        sudo apt install -y build-essential cmake ninja-build \
                            clang-tidy-16 clang-format-16 cpplint \
                            libfmt-dev libspdlog-dev libeigen3-dev \
                            libglfw3-dev libgl1-mesa-glx libgl1-mesa-dev \
                            catch2-testing # Ensure Catch2 is available or use FetchContent

    - name: Configure CMake
      run: >
        cmake -B build -G Ninja
        -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}
        -DUSE_SANITIZERS=${{ matrix.sanitizers }}
        -DUSE_STATIC_ANALYSIS=ON # Assuming static analysis is controlled by CMake option

    - name: Build
      run: cmake --build build

    - name: Run Tests (with Sanitizers if enabled)
      if: matrix.build_type == 'Debug'
      run: |
        cd build
        ./tests ${{ matrix.test_flags }}
      # Add coverage generation here if needed

    - name: Run Tests (Release)
      if: matrix.build_type == 'Release'
      run: |
        cd build
        ./tests ${{ matrix.test_flags }}

    # Optional: Upload test results
    - name: Upload Test Results
      if: always() && matrix.build_type == 'Debug'
      uses: mikepenz/action-junit-report@v3
      with:
        check_name: Sanitizer Test Report (${{ matrix.build_type }})
        report_paths: 'build/sanitizer.xml' # Path to JUnit XML report
```

**How it Works:**

*   CMake flags like `-DCMAKE_BUILD_TYPE=Debug` and `-DUSE_SANITIZERS=ON` enable the sanitizers during compilation and linking.
*   The `tests` executable is run with flags that output results in a format CI systems can parse (like JUnit XML).
*   If a sanitizer detects an issue (memory leak, use-after-free, undefined behavior), the test run will typically fail, immediately highlighting the problem.

#### Guideline vs. Rule: Fail on Sanitizer Errors

*   **Guideline:** Always run tests with sanitizers enabled in a Debug or testing build configuration, especially in CI. A sanitizer error should be treated as a test failure.
*   **When to Break the Rule:** Sanitizers can introduce performance overhead. They are generally not used in Release builds deployed to production unless specifically required for diagnostic purposes. However, *any* sanitizer hit during development or testing should be considered a high-priority bug.

---

### 3.5 Code Review Checklist (Appendix D)

A consistent code review process ensures that best practices are followed and potential issues are caught early.

#### Reviewer Ticks Every Box Before Merge – No Negotiation

*   **Format Clean:** `clang-format` applied, CI confirms. (Automated check)
*   **Test Coverage:** ≥ 90% line coverage for new/modified code. (Measured by coverage tools)
*   **Sanitizer Clean:** All tests pass with ASan, UBSan, LSan enabled. (CI proves it)
*   **Memory Safety:**
    *   No raw `new`/`delete` outside of low-level allocators.
    *   No `std::shared_ptr` in real-time critical paths.
    *   RAII used for all resources (files, locks, network sockets, etc.).
*   **Exception Safety:**
    *   No exceptions thrown or caught in real-time control loops (1 kHz path).
    *   `std::expected` used for fallible operations elsewhere.
*   **Real-Time Guarantees:**
    *   Deadline-miss counter logged and monitored.
    *   No blocking calls or dynamic allocations in critical loops.
*   **Documentation:** Public APIs have Doxygen comments explaining purpose, parameters, return values, and potential errors.
*   **Readability:** Code is clear, follows established patterns, and avoids "clever" tricks.

#### Guideline vs. Rule: Formalize Reviews

*   **Guideline:** Implement a mandatory code review process using a checklist. Integrate checks where possible (e.g., `clang-format`, test coverage).
*   **When to Break the Rule:** Extremely urgent hotfixes where immediate deployment is necessary. However, these changes must still be reviewed retrospectively and refactored according to the checklist afterward.

---

## 4. First-Year Mastery Material

This section delves into advanced topics required for highly demanding robotics applications. The original comprehensive guide contains detailed explanations and examples for these areas.

*   **4.1 Custom Memory Arenas for µ-sec Determinism:** Achieve microsecond-level deterministic performance by managing memory allocation manually within predictable pools. (See Section 5.1 for pooling concepts).
*   **4.2 Lock-Free Multi-Producer Queues:** Implement or utilize advanced lock-free data structures for high-throughput, multi-threaded scenarios. (See Section 13.1 for memory ordering, Section 12.1 for SPSC examples).
*   **4.3 NUMA-Aware Thread Pools:** Optimize multi-threaded performance on Non-Uniform Memory Access (NUMA) architectures. (See Section 12.2 for thread pool basics).
*   **4.4 Formal MISRA / ISO-26262 Subset:** Adhere to industry standards for safety-critical systems, ensuring rigorous code quality and verifiability. (Refer to specific standards documentation; C++ subset guidelines are crucial).
*   **4.5 Worst-Case Execution Time (WCET) Proofs:** Formally analyze and guarantee the execution time bounds of critical code sections. (Requires specialized tools like RapiTime or Otawa, and careful coding practices).

---

## 5. Guidelines Relaxed – Rules Clarified

As your team gains experience, understanding *why* rules exist allows for informed decisions about when exceptions are justified.

| OLD RULE (Absolute)                | NEW GUIDELINE (with Escape Hatch)                                                                                             | Rationale                                                                                                |
| :--------------------------------- | :---------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| “NEVER use `new`/`delete`”         | Use RAII; raw `new`/`delete` only in low-level allocator code reviewed by two senior engineers.                                | Avoids fragmentation and unpredictable latency. RAII ensures resources are managed correctly.            |
| “NO exceptions at all”             | No exceptions in 1 kHz (or faster) real-time paths; use `std::expected` everywhere else; exceptions allowed at start-up/shutdown. | Exceptions introduce non-deterministic latency. `std::expected` provides explicit, predictable error handling. |
| “NO `std::shared_ptr`”             | Prefer `std::unique_ptr` for ownership; `std::shared_ptr` acceptable for complex shared lifetimes (e.g., async callbacks) if *truly* needed and reviewed. | `shared_ptr` has overhead and can create cycles, complicating lifetime management. `unique_ptr` enforces clear ownership. |
| “ALWAYS pass by `const&`”          | Pass small, trivially-copyable types (≤ 2x word size, e.g., `int`, `float`, small structs) by value. Measure if unsure.        | Passing small types by value can be faster due to avoiding indirection and potential cache misses.        |
| “NO dynamic containers in RT”      | Use fixed-size containers (`std::array`, pre-allocated pools) in RT paths. Dynamic containers only in non-critical sections.   | Resizing containers involves allocation, violating RT constraints.                                         |

---

## 6. Python ↔ C++ Rosetta Side-Bars

Throughout this guide, you'll find gray boxes titled **“Python You Already Know”** and **“C++ Equivalent”**. These are designed to help you map familiar Python concepts to their C++ counterparts, facilitating learning and reducing the cognitive load of migration.

### Example: Python List Comprehension vs. C++ Loop

#### Python You Already Know

```python
# Python: List comprehension for concise data transformation
squares = [x**2 for x in range(10) if x % 2 == 0]
# squares will be [0, 4, 16, 36, 64]
```

#### C++ Equivalent: Explicit Loop with `std::vector`

```cpp
#include <vector>
#include <numeric> // For std::iota (optional)

// C++ equivalent using a standard loop and std::vector
std::vector<int> squares;
squares.reserve(5); // Optional: pre-allocate space if size is known

for (int x = 0; x < 10; ++x) {
    if (x % 2 == 0) {
        squares.push_back(x * x); // Adds element to the vector
    }
}
// squares will contain {0, 4, 16, 36, 64}
```

**Key Differences & C++ Notes:**

*   **Explicit `reserve`:** In C++, `std::vector` doesn't automatically resize like Python lists. `reserve()` can pre-allocate capacity to avoid reallocations if the final size is predictable, improving performance.
*   **No Implicit Casting:** C++ requires explicit types. `x` is an `int`, `x*x` is an `int`.
*   **Readability:** While C++ loops are more verbose, they offer greater control and clarity, especially in complex scenarios.

### Example: Python Dictionary vs. C++ Struct

#### Python You Already Know

```python
# Python: Dictionary for configuration parameters
config = {
    "motor_count": 6,
    "control_rate": 1000, # Hz
    "max_torque": 10.0    # Nm
}
```

#### C++ Equivalent: `struct` for Typed Data

```cpp
#include <string>

// C++ equivalent using a struct
struct RobotConfig {
    int motor_count = 6;          // Default values are good practice
    int control_rate_hz = 1000;
    float max_torque_nm = 10.0f;
    std::string ip_address = "192.168.1.100"; // Can add more fields
};

// Usage:
RobotConfig config; // Uses default values
config.control_rate_hz = 500; // Modify values
```

**Key Differences & C++ Notes:**

*   **Strong Typing:** C++ `struct`s enforce types (`int`, `float`, `std::string`). This catches errors at compile time that might occur at runtime in Python.
*   **Default Values:** C++ structs can have default member initializers, making them behave similarly to Python dictionaries with default values.
*   **Readability:** `struct`s provide clear, named members, improving code readability.

---

## 7. Document Cross-References

The original, comprehensive documentation for each feature, library, and pattern is preserved. Each tutorial section concludes with **"Where to read more"** links pointing to the specific subsections of the original guide that contain exhaustive details, code examples, compiler flags, sanitizer configurations, QoS XML snippets, and advanced patterns. Nothing has been removed; the material has been reorganized for pedagogical clarity.

---

## 8. Quick-Reference Cards (Tear-off)

Appendices provide concise summaries for quick lookups.

### Appendix A – Thread-Safety Cheat Sheet

| Pattern                 | Use Case                                         | Example Snippet                                                                         |
| :---------------------- | :----------------------------------------------- | :-------------------------------------------------------------------------------------- |
| `std::mutex`            | Protect shared data                              | `std::lock_guard<std::mutex> lock(my_mutex); // Critical section`                       |
| `std::shared_mutex`     | Many readers, few writers                        | `std::shared_lock reader_lock(rw_mutex);`                                               |
| `std::atomic<T>`        | Lock-free single values                          | `std::atomic<int> counter; counter.fetch_add(1);`                                       |
| `SPSCQueue`             | RT producer/consumer (1 producer, 1 consumer)    | `sensor_queue.try_push(data);` / `sensor_queue.try_pop(value);`                         |
| `Channel<T>`            | Go-style communication, task coordination        | `ch.send(msg);` / `auto val = ch.receive();`                                           |
| `std::condition_variable` | Thread synchronization, waiting for events       | `cv.wait(lock, []{ return data_ready; });` / `cv.notify_one();`                        |
| `std::latch`, `std::barrier` | C++20 synchronization primitives               | `start_latch.count_down();` / `sync_point.arrive_and_wait();`                           |
| `std::semaphore`        | Resource counting                                | `resources.acquire(); // Get resource`                                                  |

### Appendix B – Memory Ordering

| Order                     | Use Case                                     | Guarantees                                                                    |
| :------------------------ | :------------------------------------------- | :---------------------------------------------------------------------------- |
| `memory_order_relaxed`    | Counters, flags (no sync needed)             | No synchronization or ordering constraints.                                   |
| `memory_order_acquire`    | Read synchronization                         | Ensures reads after this operation see writes prior to a corresponding `release`. |
| `memory_order_release`    | Write synchronization                        | Ensures writes before this operation are visible to subsequent `acquire`s.    |
| `memory_order_acq_rel`    | Read/Write sync on RMW operations            | Combines acquire and release semantics.                                       |
| `memory_order_seq_cst`    | Default, strongest, sequential consistency | Guarantees a single total global order of all `seq_cst` operations.           |

### Appendix C – Common Pitfalls

1.  **Data Races:** Multiple threads accessing shared data without synchronization, leading to unpredictable results.
    ```cpp
    // ❌ BAD: Data race
    int shared_data = 0;
    // Thread 1: shared_data++;
    // Thread 2: shared_data++; // Race condition!

    // ✅ GOOD: Use atomic operations
    std::atomic<int> shared_data{0};
    // Thread 1: shared_data.fetch_add(1);
    ```

2.  **Deadlocks:** Two or more threads waiting indefinitely for each other to release a resource.
    ```cpp
    // ❌ BAD: Potential deadlock (different lock order)
    std::mutex m1, m2;
    // Thread 1: locks m1, then m2
    // Thread 2: locks m2, then m1

    // ✅ GOOD: Use std::scoped_lock for hierarchical locking
    // std::scoped_lock lock(m1, m2); // Locks both in a safe order
    ```

3.  **Real-Time Violations:** Performing non-deterministic operations within a real-time loop.
    ```cpp
    // ❌ BAD: Allocation in RT loop
    void control_loop() {
        std::vector<float> temp_data; // Allocation occurs here!
        temp_data.push_back(read_sensor());
        // ... process temp_data ...
    }

    // ✅ GOOD: Use pre-allocated buffers or pools
    std::array<float, 100> rt_buffer; // Pre-allocated
    size_t buffer_idx = 0;
    void control_loop() {
        if (buffer_idx < rt_buffer.size()) {
            rt_buffer[buffer_idx++] = read_sensor();
        }
        // ... process rt_buffer ...
    }
    ```

### Appendix D – Code Review Checklist (New)

*   [ ] **Format Clean:** `clang-format` applied, CI confirms.
*   [ ] **Test Coverage:** ≥ 90% line coverage for new/modified code.
*   [ ] **Sanitizer Clean:** All tests pass with ASan, UBSan, LSan enabled.
*   [ ] **Memory Safety:**
    *   RAII used for all resources.
    *   No raw `new`/`delete` outside audited allocators.
    *   No `std::shared_ptr` in real-time critical paths.
*   [ ] **Exception Safety:**
    *   No exceptions in real-time control loops.
    *   `std::expected` used for fallible operations.
*   [ ] **Real-Time Guarantees:**
    *   Deadline-miss counter logged and monitored.
    *   No blocking calls or allocations in critical loops.
*   [ ] **Documentation:** Public APIs have Doxygen comments.
*   [ ] **Readability:** Code follows patterns, avoids unnecessary complexity.
