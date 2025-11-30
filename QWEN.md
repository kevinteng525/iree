# IREE Project Context

## Project Overview

**IREE** (Intermediate Representation Execution Environment) is an MLIR-based end-to-end compiler and runtime that lowers Machine Learning (ML) models to a unified IR that scales up to meet the needs of the datacenter and down to satisfy the constraints and special considerations of mobile and edge deployments.

IREE provides both compiler infrastructure for transforming ML models and a runtime for executing them across diverse hardware platforms. The project uses a dual build system approach with support for both CMake and Bazel, though CMake is recommended for external users.

## Key Architectural Components

### Core Directories
- `/compiler/` - MLIR dialects, LLVM compiler passes, module translation code
  - `/bindings/` - Python and other language bindings
- `/runtime/` - Standalone runtime code including the VM and HAL drivers
  - `/bindings/` - Python and other language bindings
- `/integrations/` - Integrations with other frameworks like TensorFlow
- `/tests/` - Tests for full compiler->runtime workflows
- `/tools/` - Developer tools (`iree-compile`, `iree-run-module`, etc.)
- `/samples/` - Example code and usage demonstrations

### Compiler Architecture
- `API/` - Public C API
- `Codegen/` - Code generation for compute kernels
- `Dialect/` - MLIR dialects (Flow, HAL, Stream, VM, etc.)
- `InputConversion/` - Conversions from input dialects and preprocessing

### Runtime Architecture
- `base/` - Common types and utilities
- `hal/` - Hardware Abstraction Layer with hardware implementations
- `schemas/` - Data storage format definitions using FlatBuffers
- `task/` - Task system for multi-threaded CPU execution
- `vm/` - Virtual Machine for IREE modules
- `tooling/` - Developer utilities (not for production use)

## Building and Running

### CMake Build (Recommended)

**Prerequisites:**
- CMake 3.21-3.24
- Ninja (recommended generator)
- Clang compiler preferred (GCC supported but less tested)

**Quick Build:**
```bash
git clone https://github.com/iree-org/iree.git
cd iree
git submodule update --init

# Recommended development build on Linux/macOS
cmake -G Ninja -B ../iree-build/ -S . \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo \
    -DIREE_ENABLE_ASSERTIONS=ON \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_CXX_COMPILER=clang++

cmake --build ../iree-build/
```

**Key CMake Options:**
- `-DIREE_BUILD_COMPILER=ON/OFF` - Build compiler tools (default ON)
- `-DIREE_BUILD_PYTHON_BINDINGS=ON/OFF` - Python bindings (default OFF) 
- `-DIREE_HAL_DRIVER_*` - Enable specific runtime drivers (CUDA, Vulkan, Metal, etc.)
- `-DIREE_TARGET_BACKEND_*` - Enable specific compiler backends (LLVM-CPU, CUDA, etc.)

**Running Tests:**
```bash
# Build test dependencies and run all tests
cmake --build ../iree-build/ --target iree-run-tests
# Or run with ctest directly
ctest --test-dir ../iree-build/
```

### Bazel Build (Internal Use)

**Note:** Bazel build is primarily for internal infrastructure.

**Quick Build:**
```bash
python3 configure_bazel.py
bazel test -k //...
```

## Key Developer Tools

- `iree-compile` - Main compiler driver
- `iree-run-module` - Execute compiled IREE modules
- `iree-check-module` - Test runner for check framework
- `iree-opt` - Compiler pass testing tool
- `iree-run-mlir` - Run MLIR files directly through compilation pipeline
- `iree-dump-module` - Inspect compiled module contents

## Development Conventions

- Written primarily in C++ with MLIR/LLVM infrastructure
- Uses both CMake and Bazel build systems
- Python bindings available for both compiler and runtime
- Follows LLVM coding standards and conventions
- Uses Clang format for code style
- Extensive testing with both unit tests and end-to-end tests

## License

IREE is licensed under the Apache 2.0 License with LLVM Exceptions.