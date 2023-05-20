---
theme: seriph
class: text-center
highlighter: shiki
---

# Koishi Talk

<div class="opacity-80">
koishi-plugin-secret
</div>

---

# koishi-plugin-secret

安全地管理、传输和存储私密数据

---

# 现状

> 有哪些数据应当作为私密数据进行保管？

假定这样一个场景：我是 il，我在 42 云上运行着一个原神 Bot。Bot 的数据库使用阿里云 MySQL，gocq 使用 Koishi 内置，我和原神群的群主两个人拥有 Bot 的管理权限。

角色说明：

- Koishi 管理员：我、原神群的群主
- 机器人用户

---

# 现状

> 有哪些数据应当作为私密数据进行保管？

假定这样一个场景：我是 il，我在 42 云上运行着一个原神 Bot。Bot 的数据库使用阿里云 MySQL，gocq 使用 Koishi 内置，我和原神群的群主两个人拥有 Bot 的管理权限。

问题：

1. 42：42 无法访问 Koishi 控制台，但他仍然可以 **通过查看文件看到 QQ 账号和密码、登录阿里云上的数据库，查看所有人的米游社密码**
1. 原神群群主：可以访问 Koishi 控制台，**通过控制台看到 QQ 账号和密码、登录阿里云上的数据库，查看所有人的米游社密码**

---

# 现状

> 有哪些数据应当作为私密数据进行保管？

假定这样一个场景：我是 il，我在 42 云上运行着一个原神 Bot。Bot 的数据库使用阿里云 MySQL，gocq 使用 Koishi 内置，我和原神群的群主两个人拥有 Bot 的管理权限。

我希望：

1. 42：42 无法访问 Koishi 控制台，**无法看到 QQ 密码、无法看到数据库密码**
1. 原神群群主：可以访问 Koishi 控制台，**无法看到 QQ 密码、无法看到数据库密码、仍然可以修改其他配置**

---

# 目标

必须实现：**私密数据的管理、传输和存储是安全的**

具体来讲，

### 目标 ①：在管理、传输和存储私密数据时，不得使用明文

- 管理：需要提供凭证才能管理数据
- 传输：使用加密流量传输数据
- 存储：需要加密存储数据

> 需要注意：由于内存加密已经超出了 Koishi 的安全性需求，内存中我们可以明文暂存私密数据。
>
> 这也意味着上文中「需要提供凭证」只需在 Koishi 启动时提供一次即可。Koishi 可将凭证暂存于内存。

### 目标 ②：配置项和凭证尽可能少

---

# 数据

## 两类持久化数据

- 全局数据：对 Koishi 管理员的私密数据：QQ 密码、数据库密码等，这些数据在 Koishi 实例内唯一
- 用户数据：对机器人用户的私密数据：米游社密码等，这些数据对用户唯一

## 临时数据

- 凭证：存储在内存，每次 Koishi 启动后需要提供

---

# secret 服务

- 全局数据/用户数据 API
- 检查凭证/提供凭证 API
- 读取、创建/修改、删除 API
- 等待读取 API

---

# secret-provider

- 任何一个 secret-provider 都实现了完整的 secret 服务，提供了全套 API
- 负责数据的存储

---

# 生态

可接入 secret 服务的插件：

- database 系，存储服务器 IP 和数据库密码
- auth，存储用户密码
- gocqhttp，存储 QQ 号和密码
- 平台插件，存储平台登录凭证（音游查分、原神、ChatGPT 等）
- 任何此前将私密数据存储在插件配置或数据库内的插件

---

# secret-provider-memory

- 实现方法：直接在内存里存储对象
- 平台：neutral
- 缺陷：实例停止后数据会丢失

---

# secret-provider-plaintext

- 实现方法：memory 的基础上序列化成 JSON，再存文件即可
- 平台：neutral
- 缺陷：不满足目标

---

# secret-provider-aes

- 实现方法：plaintext 的基础上再进行 AES 加密，然后再存文件
- 平台：neutral
- 缺陷：Koishi 启动后到 Koishi 用户输入密钥的这段时间里，secret 服务不可用

---

# secret-provider-keytar

- 实现方法：依赖 `atom/node-keytar` 和 `koishi-plugin-downloads`
- 缺陷：依赖 `koishi-plugin-downloads`；该项目已被 archive

---

# secret-provider-wcm

### 方案一

- 参考资料：https://learn.microsoft.com/windows-server/administration/windows-commands/cmdkey
- 实现方法：memory 的基础上序列化成 JSON，然后调用 cmdkey 即可
- 管理：https://support.microsoft.com/en-us/windows/accessing-credential-manager-1b5c916a-6a16-889f-8581-fc16e8165ac0
- 平台：win
- 缺陷：cmdkey 被替换则数据不安全

### 方案二

- 实现方法：依赖 `ettorer/npm-rdpcred` 和 `koishi-plugin-downloads`，解决了 cmdkey 被替换的问题
- 缺陷：依赖 `koishi-plugin-downloads`

---

# secret-provider-dpapi

- 参考资料：https://learn.microsoft.com/zh-cn/windows/win32/api/dpapi
- 实现方法：memory 的基础上序列化成 JSON，依赖 `@primno/dpapi` 和 `koishi-plugin-downloads` 进行序列化，再存文件即可
- 平台：win
- 缺陷：依赖 `koishi-plugin-downloads`

---

# secret-provider-keychain

- 实现方法：使用 `drudge/node-keychain`
- 管理：https://support.apple.com/guide/mac-help/mchlf375f392/mac
- 平台：macos
- 缺陷：security 被替换则数据不安全

---

# secret-provider-secretservice

- 前置条件：需要系统上已经配好钱包
- 平台：linux

---

# secret-provider-gpg

- 参考资料：https://www.passwordstore.org
- 实现方法：memory 的基础上序列化成 JSON，然后调用 pass 即可
- 平台：macos，linux
- 缺陷：pass 被替换则数据不安全

---

# 问题

### 设计缺陷：用户数据对 Koishi 管理员可读写

### 数据库等待读取问题
