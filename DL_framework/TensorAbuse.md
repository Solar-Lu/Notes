# TensorAbuse: 深度学习框架中的API滥用攻击

## 概述

其实阅读完这篇论文，对于模型的认知又有了一些改观。在上周阅读的models are codes认识到模型不是简单的序列化数据，本质上也是可执行代码；在这这篇论文之前，曾以为 TensorFlow 的 SavedModel 只是weights+graph的静态包裹；但是SavedModel 里每一个 `tf.raw_ops.DebugIdentityV3` 或 Save 节点，都会在推理时被 原封不动地执行，说明这些而是具备文件、网络、进程等完整能力的可执行指令。

因此，模型有着权重的代码；一旦脱离测试沙箱，它就能像脚本一样读写任意文件、对外连接、甚至写入 SSH 公钥。

从安全的角度，说明检查不能只盯着weights，必须逐节点审计计算图里的 API 行为，并且从网站上直接加载他人生成的模型的风险与运行陌生脚本没有差异。

## 阅读记录

### 一、研究背景与动机

- **Hugging Face、TensorFlow Hub等平台**推动预训练模型共享，降低开发成本，但带来安全风险。
- **现有研究的局限**：传统研究关注对抗样本、模型窃取、数据投毒，忽视模型文件本身作为攻击载体的可能性。虽然在之前的文章中提出，TensorFlow默认使用SavedModel格式，被认为不易被嵌入恶意代码，但其合法API的隐藏能力未被系统分析。

### 二、TensorAbuse攻击

通过滥用TensorFlow合法API的隐藏能力（file reading&writing, network access），在模型中嵌入恶意行为，无需利用漏洞或修改模型参数。

**攻击特点：**

1. 恶意行为嵌入在合法API调用中，模型结构无显著变化（仅增加<1%的运算节点）。
2. 攻击代码随模型序列化保存为SavedModel格式，在用户使用模型加载时自动触发。
3. Hugging Face、TensorFlow Hub及ModelScan工具均无法检测。

### 三、系统分析TensorFlow API

#### 1. PersistExt

**目标：**识别可被序列化到SavedModel中的持久API。

**方法：**
- **Python层**：通过AST解析Python代码，定位调用底层C++实现的API（理解为连接点）
- **C++层**：用CodeQL查询C++源码，提取`REGISTER_KERNEL_BUILDER`宏注册的Op Kernel类及其实现。

**结果：**从TensorFlow v2.15中提取1,083个持久API。

#### 2. CapAnalysis

**目标：**分析API的背后隐藏的能力

主要使用的时LLMs进行分析，用ChatGPT4分三阶段（注释分析、宏分析、逻辑分析）进而进行分类API能力，并且需要结合人工验证。最终能力分类为5大类（文件访问、网络通信、进程管理、代码执行、纯计算）及13子类。

**分析结果：**83%的分类准确率，发现例如`DebugIdentityV3`可发送网络数据，`ImmutableConst`可读取文件。

### 四、实际测试

为了检测利用API的攻击能力，作者进行了测试

#### 1. Primitives

将primitive分为了以下几个类型，攻击案例主要由他们进行串联而成

| Primitive | API | 功能 |
|-----------|-----|------|
| Arbitrary File Read | FixedLengthRecordDatasetV2 | 读取任意文件内容 |
| Arbitrary File Write | PrintV2 | 写入或覆盖文件 |
| Directory Read | MatchingFiles | 枚举目录结构 |
| Network Send | DebugIdentityV3 | 向远程服务器发送数据 |
| Network Receive | rpc_client | 从远程服务器接收数据 |

#### 2. 合成攻击案例

- **文件泄露**：读取`/etc/passwd`等敏感文件并外传。
- **IP暴露**：通过发送网络请求泄露用户IP。
- **代码执行**：写入恶意Python文件，利用模块加载优先级劫持执行。
- **Shell访问**：修改`~/.ssh/authorized_keys`，植入攻击者公钥实现SSH登录。

### 五、实验验证

作者在Hugging Face Hub、TensorFlow Hub、ModelScan等平台进行了测试，结果如下：

- **全部攻击绕过检测**：上传的恶意模型未被标记或警告。
- **性能影响**：Yamnet模型仅增加<1%运算节点，推理结果无异常。
- **历史版本分析**：部分攻击API自TensorFlow 1.0.0已存在，风险长期潜伏。

### 六、防御建议与影响

#### 1. 检测工具

可以利用作者发布扫描工具，通过分析SavedModel中的Op名称和参数检测可疑行为（如敏感文件路径、IP地址）；并且在实测效果中扫描3,000个Hugging Face模型，发现24个使用高风险API，但无实际恶意行为。

#### 2. 三方责任建议

- **对于框架开发者**：可以限制API参数（如文件路径仅限工作目录）、加密网络通信API的数据等；
- **对于模型平台（Hugging Face等）**：需要集成API黑名单扫描，并且警告用户模型潜在风险；
- **对于用户群体**：在大规模使用模型之前，尽量使用sandbox运行模型。

#### 3. 长期影响

在Tensorflow进行测试后，该方法可扩展到PyTorch、TensorFlow Lite等其他框架。并且从长远来看，不能仅限于对于code的测试，而是需要转向framework API的测试和保护，从而推动AI安全研究新方向。

## 实验测试

### 1. PersistExt

#### 编译Tensorflow

![TensorFlow编译流程1](jpg/tensor(1).png)

解析python的API

![TensorFlow编译流程2](jpg/tensor(2).png)

根据C++代码在TensorFlow源码文件夹中的位置，C++代码、C++宏等，并与之前获取的python接口代码一起构建出一套完整的跨语言调用链

![TensorFlow编译流程3](jpg/tensor(3).png)

例如这个，python层通过`pywrap_tfe.TFE_Py_FastPathExecute`或`_op_def_library._apply_op_helper`调用C++的`XlaCompileOnDemandOp::Compute`方法，C++层根据设备类型选择PJRT或传统XLA编译器进行即时编译和执行，实现了从用户友好的Python接口到高性能底层实现的完整调用。

### 2. LLM的CapAnalysis
RpcClient操作的网络访问能力，并且进行了可视化，说明存在一定量的高危漏洞
![LLM的CapAnalysis](jpg/tensor(5).png)
![LLM的CapAnalysis](jpg/tensor(7).png)
![LLM的CapAnalysis](jpg/tensor(8).png)

### 3. TensorAbuse攻击检测

![TensorAbuse攻击检测](jpg/tensor(4).png)

在攻击检测中，发现一个高危漏洞：攻击者尝试读取系统文件 `/etc/passwd`，即用户密码（并且使用base64编码来隐藏恶意字符串）；于此同时，发现一个中危漏洞攻击者使用printv2操作用户输出格式化的字符串，可能会导致信息泄露。

![TensorAbuse攻击检测](jpg/tensor(6).png)
tensorflow的框架问题的检测，还检测到了一个h5 model的 Lambda 层的高危漏洞，能定位到特定的函数名和模块.