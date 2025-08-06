+++
title = 'Cipher'
date = 2025-08-06T16:52:02+08:00
draft = true
+++


## 密码的加密模式(ECB CBC CFB OFB CTR)

对称加密算法（如 AES、DES）本质上只能对固定大小的块（如 128-bit）进行加密。

为了支持加密任意长度的消息，引入了 加密模式（Cipher Mode）。

不同模式的核心区别是：每个块之间如何关联，以及如何引入随机性。

---

### ECB（Electronic Codebook）电子密码本模式

每个明文块 独立加密，不依赖其他块。

$C_i = E_K(P_i)$

<!-- ![](https://notes.andywu.tw/wp-content/uploads/2019/02/@0.5xScreen-Shot-2019-02-08-at-16.55.41-copy.png) -->
![](ecb.png)

#### ⚠️ 特点：

- 实现简单、并行友好
- 相同的明文块 → 相同的密文块
- 容易泄露明文模式，不推荐用于实际加密

#### 📉 安全性：

- 不安全！容易暴露结构信息，著名的“企鹅攻击”（BMP图像 ECB 加密仍可看出轮廓）。

### CBC（Cipher Block Chaining）密码块链接模式

每个明文块要先与 前一个密文块异或，然后再加密。


$C_i = E_K(P_i \oplus C_{i-1})$

其中：

$C_0 = IV$（初始化向量）

🔓 解密：

$P_i = D_K(C_i) \oplus C_{i-1}$

<!-- ![](https://notes.andywu.tw/wp-content/uploads/2019/02/@0.5xScreen-Shot-2019-02-08-at-22.47.50-copy.png) -->
![](cfb.png)

#### ⚠️ 特点：

- 引入链式依赖，加密无法并行，解密可以并行
- 需要随机 IV
- IV 必须唯一不可预测（不能重复使用）

#### 📉 安全性：

- 防止明文块重复导致密文重复

### CFB（Cipher Feedback）加密反馈模式

前一个密文块加密后，用其输出与明文异或生成当前密文。

$C_i = P_i \oplus E_K(C_{i-1})$

其中：

$C_0 = IV$（初始化向量）

🔓 解密：

$P_i = C_i \oplus D_K(C_{i-1})$

<!-- ![](https://notes.andywu.tw/wp-content/uploads/2019/02/@0.5xScreen-Shot-2019-02-08-at-22.50.01-copy.png) -->
![](cfb.png)

#### ⚠️ 特点：

- 类似流加密
- 可以加密小于块大小的数据（如1字节）
- 只能串行处理，适用于网络流等场景

### OFB（Output Feedback）输出反馈模式

不断加密前一个加密输出作为“密钥流”，与明文异或生成密文。

$\text{Stream}i = E_K(Stream_{i-1})$

$C_i = P_i \oplus \text{Stream}_i$

其中：

$Stream_0 = IV$（初始化向量）


🔓 解密：

$P_i = C_i \oplus \text{Stream}_i$

<!-- ![](https://notes.andywu.tw/wp-content/uploads/2019/02/@0.5xScreen-Shot-2019-02-08-at-23.01.54-copy.png) -->
![](ofb.png)

#### ⚠️ 特点：

- 加密/解密是完全对称的；
- 不影响明文的结构；
- 类似流加密，可预生成密钥流
- 不具备错误扩散性（一个字节出错不会影响后续）；

### CTR（Counter）计数器模式

使用一个不断递增的 计数器值作为输入，加密后作为密钥流，与明文异或。

$C_i = P_i \oplus E_K(\text{nonce} \| \text{counter}_i)$

🔓 解密：

$P_i = C_i \oplus E_K(\text{nonce} \| \text{counter}_i)$

<!-- ![](https://notes.andywu.tw/wp-content/uploads/2019/02/@0.5xScreen-Shot-2019-02-08-at-22.43.46-copy.png) -->
![](ctr.png)

#### ⚠️ 特点：

- 可加密任意长度数据；
- 加密解密完全一致；
- 加解密都可以并行执行，性能最好；
- 安全性强，广泛用于实际系统中（如 TLS、AES-GCM）；


## 参考
[andywu](https://notes.andywu.tw/2019/%E5%AF%86%E7%A2%BC%E7%9A%84%E5%8A%A0%E5%AF%86%E6%A8%A1%E5%BC%8Fecb-cbc-cfb-ofb-ctr)
