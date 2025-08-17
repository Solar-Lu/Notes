### **研究方法与工具详解**

本文提出的 **TensorScope** 是一种针对**跨深度学习框架 API** 的**差异测试（Differential Testing）**方法，旨在检测不同框架间 API 实现的不一致性及其潜在安全漏洞。其核心方法可分为四个主要步骤：

---

## **1. 提取对应 API（Counterparts Extraction）**
### **目标**  
识别不同框架中功能等效的API，以便后续进行差异测试。

### **方法**
#### **(1) 定义对应关系**
- 给定一个 API \( f \)，其对应 API 可以是单个 API 或一组 API 的组合：
  \[
  \text{counterpart}(f) = \{f_1, f_2, \dots, f_n\}
  \]
- 要求：
  - **语义等价**：输入相同数据时，输出差异应在允许误差范围内（如 \( \epsilon = 10^{-4} \)）。
  - **顺序性**：对于多API组合，执行顺序必须正确。

#### **(2) 候选 API 识别**
- 通过分析**模型转换器**（如 `tf2onnx`、`onnx2pytorch`）的**转换规则**，提取 API 对应关系。
- 例如，`tf.raw_ops.AdjustContrastv2` 在 ONNX 中被转换为 `ReduceMean`、`Sub`、`Mul`、`Add` 四个 API 的组合。
- 使用**静态分析**（如控制流图 CFG）解析转换器代码，识别目标框架中的对应 API。

#### **(3) 参数对齐**
- 由于不同框架的 API 参数可能不同（如名称、顺序、数量），需建立参数映射关系。
- 方法：
  1. 将源 API 封装为计算图 \( G_1 \)。
  2. 使用转换器将其转换为目标框架的计算图 \( G_2 \)。
  3. 比较 \( G_1 \) 和 \( G_2 \)，确定输入/输出参数的对应关系。

---

## **2. 提取约束条件（Constraint Extraction）**
### **目标**
获取 API 参数的合法取值范围，以提高测试用例的有效性。

### **方法**
#### **(1) 从 API 文档提取**
- 解析框架的 API 文档（如 `json`/`yaml` 文件），提取：
  - **数据类型**（如 `float32`、`int64`）。
  - **形状约束**（如 `shape=(2,3)`）。
  - **取值范围**（如 `min=0, max=255`）。

#### **(2) 从代码实现提取**
- 使用**静态分析工具**（如 `Semgrep` 用于 Python，`Weggli` 用于 C++）提取：
  - **断言检查**（如 `TORCH_CHECK`、`OP_REQUIRES`）。
  - **错误处理逻辑**（如 `InvalidArgument`）。
- 示例：
  ```python
  # PyTorch 中的断言
  TORCH_CHECK(kernel_size.size() == 1 or kernel_size.size() == 3, 
              "kernel_size must be a single int or a tuple of three ints")
  ```
  提取约束：`kernel_size` 的长度必须为 1 或 3。

#### **(3) 运行时补充约束**
- 如果测试时遇到错误（如 `InvalidArgument`），解析错误信息以补充新约束：
  ```python
  if handle.dtype != input_types[1]:
      raise InvalidArgument("expected dtype X, got Y")
  ```
  提取约束：`handle.dtype` 必须等于 `input_types[1]`。

---

## **3. 生成测试用例（Test Case Generation）**
### **目标**
生成能触发 API 不一致性或漏洞的输入数据。

### **方法**
#### **(1) 联合约束分析（Joint Constraint Analysis）**
- 对于同一参数在不同框架中的约束 \( C^A \) 和 \( C^B \)，计算：
  - **交集 \( C^U = C^A \cap C^B \)**：生成合法输入，测试深层逻辑。
  - **差集 \( \Delta C = C^A - C^B \)**：生成边界输入，触发不一致性。
- 示例：
  - `tf.cumsum` 允许任意维度，但 `torch.cumsum` 仅支持 4 维以下。
  - 测试 `rank=5` 的输入，可能触发 PyTorch 错误。

#### **(2) 基于 SMT 求解器的测试生成**
- 使用 **Z3Py** 求解约束，生成符合要求的输入：
  - **连续值**（如浮点数）：均匀采样。
  - **离散值**（如枚举类型）：遍历所有可能值。
  - **类别数据**（如字符串）：随机生成。

#### **(3) 特殊值测试**
- 包括：
  - **空值**（如 `shape=(0,0)`）。
  - **极值**（如 `float32` 的最大/最小值）。
  - **非法值**（如负数维度）。

---

## **4. 测试优化（Testing Optimization）**
### **目标**
提高测试效率和漏洞发现率。

### **方法**
#### **(1) 错误引导的测试用例修复（Error-guided Test Case Fixing）**
- 当测试失败时：
  1. 解析错误信息（如 `InvalidArgument`）。
  2. 反向推导正确约束（如 `dtype` 不匹配 → 修正 `dtype`）。
  3. 重新生成测试用例。

#### **(2) 范围扩展（Range Extension）**
- 如果代码覆盖率停滞：
  - **扩展数值范围**（如从 `[min, max]` 扩展到 `[min-mid, max+mid]`）。
  - **增加张量维度**（如从 `(2,3)` 到 `(2,3,1)`）。
  - **引入特殊值**（如 `NaN`、`Inf`）。

#### **(3) 测试预言（Test Oracle）**
- 定义三种测试结果：
  1. **Crash**：程序异常退出（如段错误）。
  2. **Invalid**：抛出异常（如 `ValueError`）。
  3. **Inconsistency**：输出差异超过阈值 \( \epsilon \)。

---

## **工具实现**
- **语言**：Python（3.2 KLOC）。
- **关键组件**：
  - **约束提取**：`Semgrep`（Python）、`Weggli`（C++）。
  - **约束求解**：`Z3Py`。
  - **代码覆盖率**：`coverage.py`（Python）、`llvm-cov`（C++）。
  - **错误检测**：`AddressSanitizer (ASan)`、`UndefinedBehaviorSanitizer (UBSan)`。
- **测试规模**：
  - 每个 API 生成至少 20,000 个测试用例。
  - 测试 6 个框架（TensorFlow、PyTorch、Paddle 等）的 1,658 个 API。

---

## **总结**
TensorScope 的核心创新点在于：
1. **跨框架 API 映射**：通过转换器规则建立 API 对应关系。
2. **联合约束分析**：结合不同框架的约束生成更有效的测试用例。
3. **动态优化**：利用错误信息和范围扩展提高测试效率。
4. **实际漏洞利用**：首次证明模型转换漏洞可导致分类准确率下降。

该方法在实验中发现了 257 个漏洞（包括 8 个 CVE），证明了其在深度学习框架安全测试中的有效性。