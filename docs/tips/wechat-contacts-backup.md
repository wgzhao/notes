---
title: 微信通讯录备份
description: 这篇摘要讲述了如何通过技术手段导出微信里的通讯录
tags: ["wechat", "contact"]
---

# 微信通讯录备份

[Source](https://pwfee.com/2020/03/back-up-wechat-contacts "Permalink to 数据备份系列（一）：微信通讯录备份")

身处数字时代，我们每天都在社交媒体产生内容。由于特殊的国情，我们的数字资产很难得到应有的保障，所以及时备份非常重要。一个月前，我的新浪微博被封，庆幸之前进行过 “备份”，微博内容得以保存。

在众多社交平台当中，我们每天使用最多的毫无疑问是微信了。深度使用过 Telegram 的朋友应该都知道微信之恶，我甚至想过像李如一那样[告别使用微信](https://blog.yitianshijie.net/2018/02/27/2nd-anniversary-of-ditching-wechat/)，可是这对于我和很多人来说都是不现实的。因为现在手机通讯录形同虚设，我们大部分时间都通过微信和朋友沟通，微信号就如同手机号。

要备份这些微信联系人一点也不容易，微信用尽全力保护（禁止用户直接访问）个人数据：在微信网页版看不到好友微信号、加密保存微信客户端备份后的数据内容等。网上有第三方备份微信通讯录的工作和付费备份的方案，但基于微信通讯录隐私保护的考虑，还是自己动手吧。

以下方法在 macOS 下测试可用

微信在本地存储了一个以 Hash 串命名的文件夹，每个登录过的用户都有一个独立的文件夹，内有子文件夹对应不同类别的 SQLite 数据库，且通过 [sqlcipher](https://github.com/sqlcipher/sqlcipher) 加密，我们需要利用 lldb 调试获得 `sqlite3_key`，然后对通讯录 SQLite 解密，提取数据。

文件夹名作用 Message 聊天记录 Contact 微信通讯录 Group 群组数据 Favorites 微信收藏

#### 1\. 先找到微信通讯录路径

    $ ls -l ~/Library/Containers/com.tencent.xinWeChat/Data/Library/Application\ Support/com.tencent.xinWeChat/*/*/Contact/*.db

找到 `wccontact_new2.db` 文件

#### 2\. 将找到的 wccontact\_new2.db 复制到临时目录

    $ cp wccontact_new2.db ~/tmp

#### 3\. 使用 lldb 调试 Wechat 获得密钥

* 注销当前 mac OS 微信登录
* 打开微信，不要登录
* Terminal 执行 `$ lldb -p $(pgrep WeChat)`
* Terminal 执行 `br set -n sqlite3_key`
* Terminal 执行 `continue`
* 登录 mac OS 微信
* Terminal 执行 `memory read --size 1 --format x --count 32 $rsi`，此时会输出

      0x600002457b60: 0xb7 0x14 0x95 0xab 0xc5 0x69 0x41 0x35
    0x600002457b68: 0x99 0x26 0x36 0xae 0xce 0x04 0x90 0xef
    0x600002457b70: 0x5b 0x5c 0x66 0xde 0x1a 0x36 0x4a 0x0a
    0x600002457b78: 0xba 0x71 0xf6 0xc5 0x8e 0xa7 0xc8 0x0d
* Terminal 执行 `Exit` 退出调试
* 将上面读到的内存内容粘贴到 Python 解密 [[代码](https://paste.ubuntu.com/p/M7qc2zkcSP/)][[在线运行](https://repl.it/@pwf/sqlite3keydecode)]

      Key: `0xb71495abc5694135992636aece0490ef5b5c66de1a364a0aba71f6c58ea7c80d`

#### 4\. 使用密钥打开 SQLite

* 下载 [DB Browser for SQLite](https://sqlitebrowser.org/)
* 使用 DB Browser for SQLite 打开 `wccontact_new2.db`，选择 SQLCipher 3 defaults / Raw key，然后输入密钥![](https://cdn.mingsiyu.com/usr/uploads/2020/03/3729047219.png)
* 即可打开 SQLite 文件

#### 5\. 导出数据

可以直接通过 DB Browser for SQLite 导出 CSV，当然你在上面做 SQL Query 也行。

![](https://cdn.mingsiyu.com/usr/uploads/2020/03/4130520350.jpg)

#### 6\. WCContact 数据表分析

在这张名为 WCContact 的数据表中，只有 `m_nsUsrName` 和 `m_nsAliasName` 这两个字段有用。我的微信号对应这两个字段都可以通过微信搜索找到同一个微信帐号，我猜测这两个字段是用来解决微信早期 QQ 登录和手机绑定之间的字段冲突问题。

我们可以将 `m_nsUsrName` 字段作为微信号，上图可见有些用户会使用 `wxid_` 代替微信号，这可能是因为用户前期没有注册微信 ID，通过 [wxid](https://wxid.ltd/) 可以转换成二维码添加该好友。[[mirror](https://repl.it/@pwf/wxidtoqrcode)]

灵魂拷问：咱们微信能不能做开放点？ @Allen Zhang

Reference:
[1] <https://www.v2ex.com/t/466053>