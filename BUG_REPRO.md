# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

昨天创建的 ML 工程师账号一直是有效状态，今天用正确密码登录时，接口给出的 session 到期时间却还在昨天，紧接着认证就失败；当天新建的账号不会这样。这次只做定位，先不要修改目标仓库代码；生产代码、测试和配置均保持现状。请对照用户时间字段、登录时钟以及写入的 `expires_at`，给出能解释这个时间倒挂的因果证据。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/ai-02
- 仓库地址：https://github.com/zhanglei10281852-gif/ai-02.git
- parent SHA：01284be3f39f38cd4078e3d604f12afa7d189d51

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/ai-02.git bug-repro
cd bug-repro
git checkout --detach 01284be3f39f38cd4078e3d604f12afa7d189d51
go test ./internal/service -run ^TestLoginSessionTTLStartsAtLogin$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestLoginSessionTTLStartsAtLogin$ -count=1
--- FAIL: TestLoginSessionTTLStartsAtLogin (0.58s)
    annotation_core_behavior_test.go:52: session expiry = 2026-08-18 12:00:00 +0000 UTC, want 2026-08-19 12:00:00 +0000 UTC
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	0.587s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestLoginSessionTTLStartsAtLogin$ -count=1
--- FAIL: TestLoginSessionTTLStartsAtLogin (2.97s)
    annotation_core_behavior_test.go:52: session expiry = 2026-08-18 12:00:00 +0000 UTC, want 2026-08-19 12:00:00 +0000 UTC
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	3.253s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断必须定位 internal/domain/user.go 的 User.SessionExpiry，并说明 internal/service/auth.go 的 AuthService.Login 如何用该方法替代登录时钟计算过期时间；证据需证明旧账号以历史 UpdatedAt 为 TTL 起点，写入的 expires_at 已早于当前登录时间，随后认证立即失败，而当天新账号不触发该现象。定向复现完成且生产代码、测试和配置零改动。
