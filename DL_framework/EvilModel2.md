# EvilModel 2.0 阅读记录

阅读了Evilmodel2，和2021年的版本进行对比，主要是对于替换方法进行了改进并且进行了性能测试，下面将对比这两篇论文进行记录。

## 1. 相比EvilModel 1.0核心改进

### 1.1 嵌入方法  
- **Evilmodel 1.0**：仅提出Fast Substitution快替换这一种办法，并且嵌入容量有限，最高36.9MB，准确率损失<1%。  
- **Evilmodel 2.0**：新增MSB Reservation和半替换Half Substitution，嵌入率提升至48.52%，且准确率波动<0.01%。  
- Half Substitution这种方法保留浮点数前2字节，替换后2字节，实现高容量与低性能损失的完美平衡。

### 1.2 实验规模  
- **Evilmodel 1.0**：仅测试AlexNet和少量恶意样本。  
- **Evilmodel 2.0**：覆盖10个主流模型（如VGG、ResNet、MobileNet）和19个恶意样本，构建550个恶意模型。

### 1.3 隐蔽性增强  
- **抗检测能力**：通过VirusTotal扫描、隐写分析（AUC接近随机猜测）、熵分析三重验证，证明EvilModel 2.0更难被发现。  
- **触发机制**：借鉴了DeepLooker的思想，引入基于神经网络的隐式触发器，仅在特定条件下激活恶意代码，避免传统加密负载的异常特征。

### 1.4 量化评估体系  
提出嵌入质量指标（Q），综合评估嵌入率（E）、性能影响（I）、额外工作量（P），证明Half Substitution最优（Q值1.6875，远超1.0的Fast Substitution）。

### 1.5 实际攻击场景验证  
演示针对性攻击：将WannaCry嵌入人脸识别模型，通过特征匹配触发恶意代码执行。

## 2. 结论与对比

| 特性 | EvilModel 1.0 | EvilModel 2.0 |
|------|----------------|----------------|
| 嵌入方法 | Fast Substitution | Fast Substitution、MSB、Half Substitution |
| 嵌入率 | ≤36.9MB | ≤48.52%模型体积 |
| 性能影响 | 准确率损失<1% | HS接近0%损失 |
| 实验规模 | 1模型+少量样本 | 10模型+19样本+550恶意模型 |
| 攻击场景 | - | 实际演示（WannaCry+人脸识别触发器） |

## 3. 防御方法

- **降低参数精度**：针对half substitution仅保留浮点数的前2字节。  
- **白盒检测**：通过参数字节与已知恶意代码的重叠率分析（如Mobilenet的F1.1层检测）。  
- **数字签名**：为模型附加数字签名

## 4. 局限性

仍然依赖全连接层（Transformer等新架构需适配），并且触发机制仍需依赖外部输入（如摄像头捕获的人脸）。

## 5. 实验测试

### 5.1 模型性能基准测试
- 原始ResNet50在ImageNet验证集上的准确率：**76.146%**

![基准测试结果](png/2025.8.11-8.17(Evilmodel2)/Evilmodel2%20(1).png)

### 5.2 恶意软件嵌入
此次不是使用任意的模型，而是将Lazarus恶意软件嵌入到ResNet50模型中

- **Half Substitution**：逐层遍历模型参数，将恶意软件字节替换到参数的后2字节位置

![Half Substitution](png/2025.8.11-8.17(Evilmodel2)/Evilmodel2%20(2).png)

- **Fast Substitution**：强制设置指数位为0x3C（正数）或0xBC（负数），将恶意软件字节替换到参数的后3字节位置

![Fast Substitution](png/2025.8.11-8.17(Evilmodel2)/Evilmodel2%20(3).png)

- **MSB Substitution**：保留第1字节（符号位+指数位），替换后3字节

![MSB Substitution](png/2025.8.11-8.17(Evilmodel2)/Evilmodel2%20(4).png)

### 5.3 嵌入后模型评估
针对这个恶意模型的嵌入其差别都很小，但Half的方法最小

- **Half Substitution**：嵌入恶意软件后的模型准确率：**76.136%**
- **Fast Substitution**：嵌入恶意软件后的模型准确率：**76.172%**
- **MSB Substitution**：嵌入恶意软件后的模型准确率：**76.158%**

### 5.4 恶意软件提取验证
从嵌入的模型中成功提取出完整的恶意软件，三种方法的SHA256哈希值都可以完全匹配：

![提取验证结果](png/2025.8.11-8.17(Evilmodel2)/Evilmodel2%20(5).png)

