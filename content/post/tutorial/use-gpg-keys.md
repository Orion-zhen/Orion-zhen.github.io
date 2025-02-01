---
date: "2025-02-01T09:47:49+08:00"
draft: false
title: "使用 GPG 密钥"
slug: "use-gpg-keys"
description: "一种高效的安全加密工具"
image: /post/tutorial/use-gpg-keys/cover.jpg
categories:
  - 教程
tags:
  - GPG
license: GNU General Public License v3.0
---

## 名词解释

- Key ID: 又称指纹, fingerprint. 这是 GPG 密钥对的唯一标识符, 可以显示为长或短的格式
- Email: 电子邮件地址. 需要注意的是, 一旦将密钥上传到公钥服务器, 就相当于将这个电子邮件地址暴露出去. 为了避免可能收到的垃圾邮件轰炸, 可以选择 GitHub 的 noreply 邮箱
- 公钥: 公开的密钥, 使用公钥加密后的数据只能被对应的私钥解密
- 私钥: 基本等同于互联网身份证, 要小心维护, 不能泄露. 使用私钥签名的数据可以被对应的公钥验证
- 公钥服务器: 公钥服务器是 GPG 用来存储公钥的地方, 公钥服务器上的公钥可以被其他用户导入到本地
- 撤销: revoke. 想要弃用一个 GPG 密钥对, 需要用它在创建时一同创建的撤销证书来撤销. 撤销不等于删除密钥
- 删除: 一旦上传到公钥服务器, 则无法删除. 在上传前请确保一切就绪

## 使用策略

GPG 密钥有 4 个功能:

|功能 | C | S(签名) | E(加密) | A(认证) |
| --- | --- | --- | --- | --- |
| 主密钥 | 1 | 1 | 1 | 1 |
| 子密钥 | 0 | 1 | 1 | 1 |

所以一个好的密钥使用策略是: 不使用主密钥, 而是根据不同的功能生成不同的子密钥, 每个子密钥做一件事情, 而主密钥只用来做身份确认和信任度确认. 这样的话, 哪怕某个用途的子密钥出现问题, 只需要撤销并更换这个子密钥, 而不影响其它的功能

加密算法选择: 推荐选择 ECC (椭圆曲线加密). 在保持高安全性的条件下能提供更快的运算速度和更小的密钥尺寸

考虑到发布公钥是一个不可逆的操作, 所以在发布之前一定要做好准备

- [ ] 创建主密钥时填写合适的信息, 确保不会泄露敏感信息
- [ ] 导出主密钥的密钥对, 并妥善保管
- [ ] 使用不同的子密钥的 Key ID 来处理不同的事务
- [ ] 生成主密钥的撤销整数, 并妥善保管
- [ ] 一切准备就绪, 发布公钥

## 使用方法

### 生成密钥

首先确保你的系统已经安装了 gpg

要生成密钥, 只要运行:

```shell
gpg --full-generate-key
```

然后按照提示进行操作即可, 如果拿不准可以全选默认

### 导出密钥对

要导出公钥:

```shell
gpg --armor --export <key-id> > public.key
```

要导出私钥:

```shell
gpg --armor --export-secret-key <key-id> > private.key
```

导出私钥可能需要一段时间, 并且会提示输入密码

### 导入密钥

要导入公钥:

```shell
gpg --import public.key
```

要导入私钥:

```shell
gpg --import private.key
```

### 发布公钥

```shell
gpg --keyserver <keyserver.com> --send-keys <key-id>
```

### 撤销密钥

生成撤销证书:

```shell
gpg --gen-revoke <key-id>
```

上传撤销证书:

```shell
gpg --keyserver <keyserver.com> --send-keys <key-id>
```

### 交互式编辑密钥

```shell
gpg --expert --edit-key <key-id>
```

进入命令终端后输入 `help` 即可查看帮助选项

## 使用 GPG 验证 commit

将先前导出的公钥上传到 GitHub. 具体参考 [GitHub 官方文档](https://docs.github.com/zh/authentication/managing-commit-signature-verification)

当公钥上传完成后, 在本地配置 `git`:

```shell
git config --global user.signingkey <key-id>
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

如果在 commit 是遇到问题 (Windows 下), 则需要额外配置 GPG 可执行文件路径:

```shell
git config --global gpg.program "<where-is-gpg.exe>"
```

使用这样的 commit 提交方式, 会在 GitHub commit 页面上显示一个绿色的 `Verified` 标签, 表明提交者的签名有效. 有的仓库要求贡献者的 commit 必须有效.
