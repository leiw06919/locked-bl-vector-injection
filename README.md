# 在锁 BL、无 KernelSU 的 Android 13 上把 LSPosed 系框架注入 system_server

> 一次可复现的实验记录，不是教程，也不是一键 root 方案。
> 目标机：荣耀 90GT（MAG-AN00）/ Android 13 (SDK 33) / MagicOS 7.2.0.131 / kernel 5.15.104-android13

---

## TL;DR

设备 BL 锁死，无法刷 Magisk 或 KernelSU，只有一次漏洞利用带来的短暂 `uid0` 窗口。
在这个前提下，把 **Vector 2.2 (3114)** 成功注入 `system_server`，**支付宝**的 5 个进程全部 `Loaded module` 成功。

```
System Server died
Got system server binder via serial on attempt 1
Injected Vector framework into system_server
Successfully injected Vector IPC binder for applications
```

具体用途是让 **XRadiant 3.7.6** 在支付宝里跑起来（自动收蚂蚁森林能量、庄园、会员这些日常任务）。也就是说，这套东西要解决的现实问题是：**一台锁了 BL 的日常主力机，能不能跑 Xposed 模块。**

卡了十几轮的真正原因不是提权、不是注入器，也不是"permissive 是假的"：

> `allow system_server default_android_service service_manager add` 这条规则**从来没有被加载进内核**。

Vector 自带的 `sepolicy.rule` 只写了 `find`，没写 `add`；而"加载策略"这件事本来是 root 管理器在开机时做的，我们没有。

---

## 目录

| 文档 | 内容 |
|---|---|
| [docs/root-cause.md](docs/root-cause.md) | 根因是怎么一步步推出来的，含五条事实链 |
| [docs/dead-ends.md](docs/dead-ends.md) | 五条死路 + 两个反直觉现象，每条带失败证据 |
| [docs/judgement.md](docs/judgement.md) | 什么叫"真成功"：三层判据体系与长期假信号 |

---

## 1. 适用范围

| 前提 | 本方案是否适用 |
|---|---|
| BL 已解锁 + 有 Magisk / KernelSU | **不需要本文**，正常刷模块即可 |
| 无持久 root，只能拿到临时 `uid0` | 适用 |
| 能长时间保持 `adb` 会话 | 适用（`uid0` 只活在会话里） |
| 需要 `su`、需要 Zygisk 原生模块 | **不适用**，本文给不了 |

机型相关性说明：四步流程里的**方法**是通用的，但**偏移和字节数只对本机成立**。同一台机器同版本系统可以直接参考，其他机型请按第 2 节的方法自己推导。

---

## 2. 打通流程（四步，顺序不能换）

### 步骤 1 · 内核层把 enforcing 置 0

`echo 0 > /sys/fs/selinux/enforce` 在本机会返回 `EINVAL`（厂商改过 store 路径），必须直接写内核变量。

偏移不是猜的，推导过程分三步：

1. 用内核自带的 BTF 解析 `struct selinux_state` 布局，确认编译时**没有开 Randstruct**（结构体随机化），`enforcing` 位于 `+0x00`；
2. 先 `echo 0 > /proc/sys/kernel/kptr_restrict` 拿到真实内核地址，取 `_text` 与 `selinux_state` 的实际地址，算出差值；
3. 写入前**先读原值校验**：原始 6 字节 `00 00 00 01 01 01`，与 BTF 推出的布局完全吻合，才敢下手。

本机实测：

```
_text                = 0xffffffdd03400000
selinux_state        = 0xffffffdd068b0888
差值（即偏移）        = 0x34b0888
+0x00 = enforcing    +0x01 = checkreqprot    +0x02 = initialized
```

> 这个偏移只对 **5.15.104-android13 这个具体内核编译**成立。换内核版本、换厂商、甚至换同型号的 OTA，都可能变。
> **不要照抄，按上面三步重新算。**

注意：`initialized` 位不能碰，见 [死路 1](docs/dead-ends.md)。

### 步骤 2 · live 加载策略

`enforcing=0` 时 `load_policy` 的检查被放行，这是整条路的唯一突破口。

用 `magiskpolicy --load <new.bin> --live` 加载含 `add` 规则的新策略。判断是否真的生效，看**策略总字节数的增量**：

```
1547892 → 1547904 bytes   (+12，正好是新增规则的大小)
system_server / audio / zygote 均未换代
```

`rc=0` 不算证据，字节数变化才算。

> `magiskpolicy` 不需要装 Magisk，直接从 Magisk 的 APK 里取 `lib/arm64-v8a/libmagiskpolicy.so` 改名即可用。

### 步骤 3 · 起 daemon，抢注 `serial_vector`

daemon 起来后注册 `serial_vector`，此时 `service list` 里能看到它。

### 步骤 4 · kill zygote64，让 daemon 自己切名

`kill` 掉 zygote 后，daemon 内的 `binderDied()` 会把 `startedAfterSystemServer` 置 false、注销 `serial_vector`、抢注 `serial`。新的 `system_server` 启动后，通过 `serial` 完成 late attach 并注入。

> Vector 3114 是**自动**判断 `afterSystemServer` 的（`ServiceManager.checkService("activity") != null`），
> 没有参数可以强制，`--late-inject` 是多余的。

---

## 3. 判据

四行日志缺一行都不算成，详见 [docs/judgement.md](docs/judgement.md)。

---

## 4. 已知限制

- **重启即失效。** `enforce` 回到 1、live 策略丢失、daemon 死掉，需要完整重跑（3–8 分钟）。
- **没有 `su`，没有常驻 root。** 临时 `uid0` 只活在 adb 会话里。
- **管理器可能显示"异常"。** daemon 在与 `system_server` 完成握手后会注销 rendezvous 服务名，管理器扫不到就判为异常——**功能是好的**，配置请走命令行而不是管理器 UI。
- **无法使用 Zygisk 原生模块。** 没有 Magisk / KSU 的模块加载机制，只能用 APK 形式的 Xposed 模块。
- **可检测。** `enforce=0` 全局可读，任何 App 读一次就能确认；且注入发生在 zygote 层，所有进程里都带着框架加载器。**不要在打通状态下使用银行、证券、政务类 App。**

---

## 5. 本文不提供什么

- 不提供漏洞利用代码，不提供提权步骤，不提供可直接运行的完整脚本。
- 不保证在其他机型 / 内核上成立。
- 不建议把本文当作长期方案。BL 锁死的前提下，"既能长期 root 又能安全支付"不存在，只能在两种状态之间切换。

---

## 6. 环境与产物

| 项 | 值 |
|---|---|
| 设备 | 荣耀 90GT / MAG-AN00 |
| 系统 | Android 13 (SDK 33) / MagicOS 7.2.0.131 |
| 内核 | 5.15.104-android13 |
| 框架 | Vector 2.2 (3114) |
| 模块 | XRadiant 3.7.6（作用于支付宝） |
| 策略增量 | 1547892 → 1547904 bytes |
| `selinux_state` 偏移 | `0x34b0888`（仅本机内核） |

验证时的目标进程（支付宝）：

```
tools / push / gpu_process / widgetProvider / sandboxed_privilege_process0
```

全部出现 `Loaded module com.leo.xposed.xradiant successfully`，maps 里能看到 `libxr_stealth.so` 与 `libquickjs-android-wrapper.so`。

---

## License

文档以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布，转载注明出处即可。
请不要把漏洞利用部分补进本仓库的 PR。
