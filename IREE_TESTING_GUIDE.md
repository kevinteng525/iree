# IREE 测试框架研究总结

> 本文档总结了 IREE 项目的测试框架架构、测试用例组织方式，以及如何添加新测试用例的完整指南。

---

## 目录

- [一、测试框架整体架构](#一测试框架整体架构)
- [二、测试用例类型和组织](#二测试用例类型和组织)
- [三、如何添加新测试用例](#三如何添加新测试用例)
- [四、完整示例](#四完整示例)
- [五、测试编写最佳实践](#五测试编写最佳实践)
- [六、参考资源](#六参考资源)

---

## 一、测试框架整体架构

IREE 项目采用 **LLVM lit (LLVM Integrated Tester)** 作为主要测试框架，具有以下特点：

### 1.1 核心配置文件

- **`tests/lit.cfg.py`** - 根测试目录配置
- **`compiler/lit.cfg.py`** - 编译器测试配置
- **`runtime/lit.cfg.py`** - 运行时测试配置
- **`samples/lit.cfg.py`** - 示例代码测试配置

这些配置文件定义了：
- 测试名称（统一为 "IREE"）
- 可测试的文件后缀（`.mlir`, `.txt`）
- 测试格式（`lit.formats.ShTest(execute_external=True)`）
- 环境变量和执行路径配置

### 1.2 构建系统支持

#### Bazel 构建系统
- 使用 `iree_lit_test_suite` 宏定义 lit 测试
- 使用 `iree_check_single_backend_test_suite` 宏定义端到端测试
- 配置文件：`build_tools/bazel/lit_test.bzl`, `build_tools/bazel/iree_check_test.bzl`

#### CMake 构建系统
- 使用 `iree_lit_test` 和 `iree_lit_test_suite` 函数
- 通过 `IREE_BUILD_TESTS` 选项控制是否构建测试
- 配置文件：`build_tools/cmake/iree_lit_test.cmake`

### 1.3 测试工具链

| 工具 | 用途 |
|------|------|
| `iree-opt` | 编译器优化工具，用于测试编译器 passes |
| `iree-compile` | MLIR 编译工具，将 MLIR 编译为可执行模块 |
| `iree-run-module` | 模块运行工具，执行编译后的模块 |
| `iree-check-module` | 编译并验证模块的正确性 |
| `FileCheck` | LLVM 输出验证工具，用于检查输出是否符合预期 |

### 1.4 测试框架工作原理

1. **测试发现**：构建系统自动发现符合条件的测试文件
2. **环境设置**：根据 lit 配置文件设置测试环境
3. **命令执行**：执行测试文件中定义的 `RUN` 命令
4. **结果验证**：使用 FileCheck 验证输出是否符合预期
5. **报告生成**：生成测试结果报告

---

## 二、测试用例类型和组织

### 2.1 测试类型分类

#### 编译器测试 (Compiler Tests)
- **位置**：`compiler/plugins/*/test/`
- **格式**：使用 `.mlir` 文件，包含 `RUN` 和 `CHECK` 指令
- **用途**：验证编译器 passes 和 IR 转换的正确性
- **示例**：

```mlir
// RUN: iree-opt --split-input-file --pass-pipeline="builtin.module(func.func(iree-tosa-to-linalg-ext))" %s | FileCheck %s

// CHECK-LABEL: @test_function
func.func @test_function(%arg0 : tensor<4xf32>) -> tensor<4xf32> {
  // CHECK: linalg.generic
  %result = "some.operation"(%arg0) : (tensor<4xf32>) -> tensor<4xf32>
  return %result : tensor<4xf32>
}
```

#### 端到端测试 (End-to-End Tests)
- **位置**：`tests/e2e/`
- **格式**：使用 `.mlir` 文件，包含完整的功能测试
- **用途**：验证从编译到执行的完整流程
- **特点**：
  - 使用 `util.unfoldable_constant` 定义输入（防止编译时优化）
  - 使用 `check.expect_almost_eq_const` 验证结果
- **示例**：

```mlir
func.func @abs_test() {
  %input = util.unfoldable_constant dense<[-2.0, -1.0, 0.0, 1.0, 2.0]> : tensor<5xf32>
  %result = math.absf %input : tensor<5xf32>
  check.expect_almost_eq_const(%result, dense<[2.0, 1.0, 0.0, 1.0, 2.0]> : tensor<5xf32>) : tensor<5xf32>
  return
}
```

#### 运行时测试 (Runtime Tests)
- **位置**：`runtime/src/iree/*/test/`
- **用途**：验证运行时组件和 VM 指令
- **示例**：VM 指令测试

```mlir
vm.module @arithmetic_ops_f64 {
  vm.export @test_add_f64
  vm.func @test_add_f64() {
    %c1 = vm.const.f64 1.5
    %c1dno = util.optimization_barrier %c1 : f64
    %v = vm.add.f64 %c1dno, %c1dno : f64
    %c2 = vm.const.f64 3.0
    vm.check.eq %v, %c2, "1.5+1.5=3" : f64
    vm.return
  }
}
```

#### 工具测试 (Tools Tests)
- **位置**：`tools/test/`
- **用途**：验证命令行工具的功能
- **特点**：通常包含多个 `RUN` 指令，测试不同的工具参数组合

### 2.2 测试目录结构

```
iree/
├── tests/                    # 核心测试目录
│   ├── e2e/                 # 端到端测试
│   │   ├── stablehlo_ops/   # StableHLO 操作测试
│   │   ├── matmul/          # 矩阵乘法测试
│   │   └── ...
│   ├── compiler_driver/     # 编译器驱动测试
│   └── transform_dialect/   # 变换方言测试
│
├── compiler/                 # 编译器源码及测试
│   └── plugins/*/test/      # 各插件的测试
│
├── runtime/                  # 运行时源码及测试
│   └── src/iree/*/test/     # 运行时组件测试
│
├── tools/test/              # 工具测试
└── samples/                 # 示例代码及测试
```

### 2.3 测试文件命名规范

- **描述性命名**：如 `tosa_to_linalg_ext.mlir`
- **按操作命名**：如 `abs.mlir`, `sin.mlir`, `cos.mlir`
- **统一扩展名**：使用 `.mlir` 扩展名

---

## 三、如何添加新测试用例

### 3.1 添加测试的完整流程

```
1. 选择测试类型和位置
   ↓
2. 创建测试文件
   ↓
3. 编写测试内容
   ↓
4. 更新构建配置
   ↓
5. 运行测试验证
```

### 3.2 步骤详解

#### 步骤 1: 选择测试类型和位置

根据测试目的选择：
- **测试编译器转换** → 编译器测试（`compiler/plugins/*/test/`）
- **测试完整功能** → 端到端测试（`tests/e2e/`）
- **测试运行时行为** → 运行时测试（`runtime/src/iree/*/test/`）
- **测试命令行工具** → 工具测试（`tools/test/`）

#### 步骤 2: 创建测试文件

在合适的目录下创建 `.mlir` 文件，遵循现有的命名约定。

#### 步骤 3: 编写测试内容

**编译器测试模板**：
```mlir
// RUN: <工具命令> %s | FileCheck %s

// CHECK-LABEL: @函数名
func.func @函数名(%arg0 : 类型) -> 返回类型 {
  // CHECK: 预期的输出模式
  %result = "操作"(%arg0) : (输入类型) -> (输出类型)
  return %result : 返回类型
}
```

**端到端测试模板**：
```mlir
func.func @测试名称() {
  %input = util.unfoldable_constant dense<[输入数据]> : tensor<形状x类型>
  %result = 操作(%input) : tensor<输入类型> -> tensor<输出类型>
  check.expect_almost_eq_const(%result, dense<[预期结果]> : tensor<形状x类型>) : tensor<形状x类型>
  return
}
```

#### 步骤 4: 更新构建配置

**在 BUILD.bazel 中注册测试**

对于编译器测试：
```python
load("//build_tools/bazel:iree_lit_test.bzl", "iree_lit_test_suite")

iree_lit_test_suite(
    name = "lit",
    srcs = glob(["*.mlir"]),  # 会自动包含新添加的测试文件
    tools = [
        "@llvm-project//llvm:FileCheck",
        "//tools:iree-opt",
    ],
)
```

对于端到端测试：
```python
load("//build_tools/bazel:iree_check_test.bzl", "iree_check_single_backend_test_suite")

ALL_SRCS = [
    "abs.mlir",
    "sin.mlir",
    "my_new_test.mlir",  # 添加新测试文件
]

iree_check_single_backend_test_suite(
    name = "check_llvm-cpu_local-task",
    srcs = ALL_SRCS,
    compiler_flags = [
        "--iree-input-demote-f64-to-f32",
        "--iree-llvmcpu-target-cpu=generic",
    ],
    driver = "local-task",
    target_backend = "llvm-cpu",
)
```

#### 步骤 5: 运行测试

**使用 Bazel**：
```bash
# 运行单个测试
bazel test //tests/e2e/stablehlo_ops:check_llvm-cpu_local-task_my_new_test.mlir

# 运行整个测试套件
bazel test //tests/e2e/stablehlo_ops:check_llvm-cpu_local-task

# 运行所有测试
bazel test //...
```

**使用 CMake/CTest**：
```bash
# 运行单个测试
ctest -R my_new_test

# 运行所有测试
ctest

# 详细输出
ctest -V
```

---

## 四、完整示例

### 4.1 示例场景：添加余弦函数测试

假设我们要为 `stablehlo.cosine` 操作添加端到端测试。

#### 步骤 1: 创建测试文件

文件路径：`tests/e2e/stablehlo_ops/cos.mlir`

#### 步骤 2: 编写测试内容

```mlir
// 张量测试
func.func @tensor() {
  // 输入：0, π/2, π, 3π/2
  %input = util.unfoldable_constant dense<[0.0, 1.57, 3.14, 4.71]> : tensor<4xf32>
  %result = "stablehlo.cosine"(%input) : (tensor<4xf32>) -> tensor<4xf32>
  // 预期输出：1, 0, -1, 0
  check.expect_almost_eq_const(%result, dense<[1.0, 0.0, -1.0, 0.0]> : tensor<4xf32>) : tensor<4xf32>
  return
}

// 标量测试
func.func @scalar() {
  %input = util.unfoldable_constant dense<3.14> : tensor<f32>
  %result = "stablehlo.cosine"(%input) : (tensor<f32>) -> tensor<f32>
  check.expect_almost_eq_const(%result, dense<-1.0> : tensor<f32>) : tensor<f32>
  return
}

// 负数测试
func.func @negative() {
  %input = util.unfoldable_constant dense<[-3.14, -1.57]> : tensor<2xf32>
  %result = "stablehlo.cosine"(%input) : (tensor<2xf32>) -> tensor<2xf32>
  check.expect_almost_eq_const(%result, dense<[-1.0, 0.0]> : tensor<2xf32>) : tensor<2xf32>
  return
}
```

#### 步骤 3: 更新 BUILD.bazel

在 `tests/e2e/stablehlo_ops/BUILD.bazel` 文件中：

```python
ALL_SRCS = [
    "abs.mlir",
    "add.mlir",
    # ... 其他测试文件 ...
    "cos.mlir",  # 添加新测试
    # ... 更多测试文件 ...
]
```

#### 步骤 4: 运行测试

```bash
# 测试 LLVM CPU 后端
bazel test //tests/e2e/stablehlo_ops:check_llvm-cpu_local-task_cos.mlir

# 测试 VMVX 后端
bazel test //tests/e2e/stablehlo_ops:check_vmvx_local-task_cos.mlir

# 测试所有后端
bazel test //tests/e2e/stablehlo_ops:all
```

#### 步骤 5: 验证结果

成功的测试输出示例：
```
//tests/e2e/stablehlo_ops:check_llvm-cpu_local-task_cos.mlir    PASSED in 2.3s
```

---

## 五、测试编写最佳实践

### 5.1 通用最佳实践

1. **使用 `util.unfoldable_constant`**
   - 防止编译器在编译时优化掉测试
   - 确保测试在运行时执行

2. **提供多个测试场景**
   - 标量测试（`@scalar()`）
   - 向量/张量测试（`@tensor()`）
   - 边界情况测试（`@edge_cases()`）
   - 负数测试（`@negative()`）
   - 零值测试（`@zero()`）

3. **使用描述性函数名**
   ```mlir
   func.func @tensor() { ... }           // 张量测试
   func.func @scalar() { ... }           // 标量测试
   func.func @negative_values() { ... }  // 负数测试
   func.func @boundary_cases() { ... }   // 边界测试
   ```

4. **验证精度**
   - 使用 `check.expect_almost_eq_const` 而非严格相等
   - 适用于浮点运算可能有微小误差的情况

5. **参考现有测试**
   - 查看同类型操作的测试作为模板
   - 保持一致的测试风格和结构

### 5.2 编译器测试最佳实践

1. **使用 `--split-input-file`**
   - 允许在单个文件中包含多个测试用例
   - 使用 `// -----` 分隔不同的测试

2. **精确的 CHECK 模式**
   ```mlir
   // CHECK-LABEL: @function_name
   // CHECK: %[[VAR:.+]] = operation
   // CHECK-SAME: attribute
   // CHECK-NEXT: another_operation
   // CHECK-DAG: unordered_check
   ```

3. **测试不同的优化级别**
   - 验证在不同优化级别下的行为
   - 测试 passes 的组合效果

### 5.3 端到端测试最佳实践

1. **测试多种数据类型**
   ```mlir
   // f32 测试
   func.func @f32_test() { ... }
   
   // f64 测试
   func.func @f64_test() { ... }
   
   // i32 测试
   func.func @i32_test() { ... }
   ```

2. **测试不同形状和维度**
   ```mlir
   // 1D 张量
   dense<[1.0, 2.0, 3.0]> : tensor<3xf32>
   
   // 2D 张量
   dense<[[1.0, 2.0], [3.0, 4.0]]> : tensor<2x2xf32>
   
   // 标量
   dense<1.0> : tensor<f32>
   ```

3. **测试多个后端**
   - 在 BUILD.bazel 中为不同后端创建测试套件
   - 常见后端：`llvm-cpu`, `vmvx`, `vulkan-spirv`

### 5.4 避免的常见错误

❌ **不要**：
```mlir
// 错误：使用常量直接计算，会被优化掉
%input = arith.constant dense<[1.0, 2.0]> : tensor<2xf32>
%result = math.absf %input : tensor<2xf32>
```

✅ **要**：
```mlir
// 正确：使用 unfoldable_constant 防止优化
%input = util.unfoldable_constant dense<[1.0, 2.0]> : tensor<2xf32>
%result = math.absf %input : tensor<2xf32>
```

❌ **不要**：
```mlir
// 错误：只测试一个场景
func.func @test() {
  %input = util.unfoldable_constant dense<1.0> : tensor<f32>
  %result = math.absf %input : tensor<f32>
  check.expect_almost_eq_const(%result, dense<1.0> : tensor<f32>) : tensor<f32>
  return
}
```

✅ **要**：
```mlir
// 正确：测试多个场景
func.func @positive() { ... }
func.func @negative() { ... }
func.func @zero() { ... }
func.func @tensor() { ... }
```

---

## 六、参考资源

### 6.1 官方文档

- **测试指南**：`docs/website/docs/developers/general/testing-guide.md`
- **贡献指南**：`CONTRIBUTING.md`
- **README**：`README.md`

### 6.2 示例测试位置

| 测试类型 | 目录路径 | 说明 |
|---------|---------|------|
| 编译器测试 | `compiler/plugins/input/TOSA/InputConversion/test/` | TOSA 输入转换测试 |
| 端到端测试 | `tests/e2e/stablehlo_ops/` | StableHLO 操作测试 |
| 运行时测试 | `runtime/src/iree/vm/test/` | VM 指令测试 |
| 工具测试 | `tools/test/` | 命令行工具测试 |
| 示例测试 | `samples/` | 各种使用示例 |

### 6.3 相关工具文档

- **LLVM lit**：https://llvm.org/docs/CommandGuide/lit.html
- **FileCheck**：https://llvm.org/docs/CommandGuide/FileCheck.html
- **MLIR**：https://mlir.llvm.org/

### 6.4 常用命令速查

```bash
# Bazel 测试命令
bazel test //...                                    # 运行所有测试
bazel test //tests/e2e/...                         # 运行 e2e 测试
bazel test --test_output=all <target>              # 显示详细输出
bazel test --test_filter=<pattern> <target>        # 过滤测试

# CMake 测试命令
ctest                                               # 运行所有测试
ctest -R <pattern>                                  # 运行匹配模式的测试
ctest -V                                            # 详细输出
ctest -j<N>                                         # 并行运行 N 个测试

# 手动运行 lit 测试
lit -v <test_file>                                  # 运行单个 lit 测试文件
lit <directory>                                     # 运行目录下所有测试
```

---

## 附录：快速参考

### A.1 测试文件模板

#### 编译器测试模板
```mlir
// RUN: iree-opt --pass-pipeline="builtin.module(func.func(<pass-name>))" %s | FileCheck %s

// CHECK-LABEL: @test_name
func.func @test_name(%arg0 : tensor<4xf32>) -> tensor<4xf32> {
  // CHECK: expected_pattern
  %result = "operation"(%arg0) : (tensor<4xf32>) -> tensor<4xf32>
  return %result : tensor<4xf32>
}
```

#### 端到端测试模板
```mlir
func.func @test_name() {
  %input = util.unfoldable_constant dense<[1.0, 2.0, 3.0]> : tensor<3xf32>
  %result = operation(%input) : tensor<3xf32> -> tensor<3xf32>
  check.expect_almost_eq_const(%result, dense<[1.0, 2.0, 3.0]> : tensor<3xf32>) : tensor<3xf32>
  return
}
```

### A.2 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 测试被优化掉 | 使用了普通常量 | 使用 `util.unfoldable_constant` |
| FileCheck 失败 | CHECK 模式不匹配 | 检查输出格式，调整 CHECK 模式 |
| 测试未运行 | 未在 BUILD.bazel 中注册 | 将测试文件添加到 `srcs` 列表 |
| 精度误差 | 使用了严格相等检查 | 改用 `check.expect_almost_eq_const` |
| 找不到工具 | tools 列表不完整 | 在 BUILD.bazel 中添加所需工具 |

### A.3 测试开发工作流

```
1. 确定测试需求
   ↓
2. 找到类似的现有测试作为参考
   ↓
3. 创建测试文件并编写基本测试
   ↓
4. 更新 BUILD.bazel 配置
   ↓
5. 本地运行测试
   ↓
6. 调试并修复问题
   ↓
7. 添加更多测试场景
   ↓
8. 运行完整测试套件验证
   ↓
9. 提交代码审查
```

---

**文档版本**：v1.0  
**最后更新**：2025-12-10  
**维护者**：IREE 测试团队

如有问题或建议，请参考 `CONTRIBUTING.md` 或在项目仓库提交 issue。
