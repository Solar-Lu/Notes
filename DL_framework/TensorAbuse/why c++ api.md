### ✅ Why C++ API

在 TensorFlow 中，**只有那些能在模型序列化（SavedModel）中被保留的 API 才有可能被用于攻击**。而这些 API 的实际执行逻辑通常位于 C++ 内核实现中。

---

### ✅ Why Mapping

| 映射目标 | 说明 |
|----------|------|
| **确认 API 是否持久** | Python API 只是前端接口，是否能在模型中保存取决于它是否调用了 **C++ Op Kernel** 并通过 `REGISTER_KERNEL_BUILDER` 注册。 |
| **提取隐藏能力** | 某些 Python API 的文档描述模糊，但其 C++ 实现中却包含如文件读写、网络通信等敏感操作，必须通过映射到 C++ 才能发现。 |
| **构建攻击链** | 攻击者需要知道哪些 API 可以真正在模型中执行，因此必须精确建立 Python ⇄ C++ 的对应关系。 |

---

### ✅ Methodology

论文通过以下两步完成映射：

1. **Python 层 AST 分析**：识别调用底层 C++ 的 API（如 `pywrap_tfe.TFE_Py_FastPathExecute`）。
2. **C++ 层 CodeQL 分析**：通过 `REGISTER_KERNEL_BUILDER` 宏找到每个 Op 对应的 C++ 类与 `Compute()` 方法。

---

### ✅ Analogy

这就像你要确认一个 Python 函数是否真的能“写文件”，不能只看它的名字或文档，而必须**追踪到它在 C++ 中是否调用了 `fopen()` 或 `write()`**。

---

### ✅ Conclusion
> **只有将 Python API 映射到 C++ 实现，才能准确判断它是否会被持久化到模型中，并识别其潜在恶意能力。**