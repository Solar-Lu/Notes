## 📌 1. **Task**
> 解决一个大前提：**“哪些 TensorFlow API 能被序列化进 SavedModel？”**  
> 只有这些 API 才可能被攻击者利用，因此必须先**系统性地识别出全部持久 API**。

---

## 🧩 2. **Challenges**
| 挑战 | 原因 |
|------|------|
| **官方文档缺失** | TensorFlow 没告诉你哪个 API 可以/不可以被保存到模型文件里。 |
| **手动测试不可行** | API 数量上千，逐个写测试用例根本不现实。 |
| **跨语言链路复杂** | Python 层调用 → C++ 内核实现 → 注册宏 → 序列化节点，链路长，难以一眼看穿。 |

---

## ⚙️ 3. **Methodlogy（PersistExt）**

### ✅ Step 1：定义“持久 API”的特征
- **必须满足两个条件**：
  1. Python 层最终调用了 **C++ 内核**（通过 Pybind11）。
  2. 该内核被 **REGISTER_KERNEL_BUILDER** 注册到某个设备（CPU/GPU/TPU）。
- **结果**：满足这两点的 API 才能变成 **计算图节点**，从而被序列化进 SavedModel。

---

### ✅ Step 2：Python 层 AST 分析
- **工具**：Python 标准库 `ast`。
- **任务**：遍历所有 `.py` 文件，找三类特征函数：
  1. `pywrap_tfe.TFE_Py_FastPathExecute`
  2. `_apply_op_helper`
  3. `<op_name>_eager_fallback`
- **输出**：提取到 **1,585 个 Python API**，其中 149 个甚至没在官方文档里出现。

---

### ✅ Step 3：C++ 层 CodeQL 分析
- **工具**：GitHub CodeQL（专门分析大型 C++ 代码）。
- **任务**：
  1. **找所有继承 OpKernel 的类** → 得到实际实现的 `.cc` 文件。
  2. **找所有 REGISTER_KERNEL_BUILDER 宏** → 得到 “Op 名 ↔ C++ 类” 的映射。
- **输出**：  
  - 1,147 个含 `Compute()` 方法的类  
  - 11,221 个注册宏（去重后 1,083 个唯一映射）

---

## 🎯 4. **Result**
| 指标 | 数值 |
|------|------|
| **持久 API 总数** | **1,083 个** |
| **包含隐藏能力的 API** | 至少 20 个被后续用于攻击原语 |
| **准确率验证** | 手动构造 20 个测试模型，全部成功序列化，无一失败 |

---

## 🧠 5. **Conclusion**
> PersistExt 用 **AST + CodeQL** 把「Python API → C++ 内核 → 序列化节点」整条链路自动化打通，**首次完整枚举出 1,083 个能被永久写进 SavedModel 的 TensorFlow API**，为后续发现隐藏攻击面奠定了不可替代的基础。