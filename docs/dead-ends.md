# 五条死路

> 成功路径只有四步，但真正耗掉时间的是这五条。每条都附失败证据，方便别人一眼跳过。

---

## 死路 1 · 把 `initialized` 置 0

**动机**：以为要让 SELinux 完全"未初始化"，才能绕过某些检查。

**结果**：

```
/proc/self/attr/current  →  EINVAL
binder 链断
daemon 卡死
```

**原因**：`struct selinux_state` 的前三个字节是 `enforcing` / `checkreqprot` / `initialized`。`initialized=0` 会让整个 SELinux 子系统处于非法状态，进程连自己的上下文都读不到，binder 通信依赖的标签查询随之失败。

**结论**：这个位**只能读，不能写**。改它等于把安全子系统打坏，不是"关闭"。

---

## 死路 2 · 改 `selinux_enforcing_boot`

**动机**：看到有这个变量，以为是"开机时的 enforcing 状态"，改它就能影响运行时。

**结果**：改了没有任何变化。

**原因**：本机该值**本来就是 0**。它记录的是"启动时是否 enforcing"，与当前状态无关，而且我们已经是 0 了，改它等于重复写同一个值。

**结论**：查状态先读值，别假设它是什么。

---

## 死路 3 · 用 `setprop ctl.stop / ctl.start` 重启服务

**动机**：想不重启设备就让服务重新加载配置。

**结果**：直接被拒。

**原因**：`ctl.*` 属性受 `ctl_default_prop` 之类的标签保护，非对应域写入会被 SELinux 拒绝。

**结论**：在无 root 管理器环境里，这条路基本不通；要重启服务就找该服务自己的控制通道。

---

## 死路 4 · 抢在 `system_server` 之前起 daemon

**动机**：读到框架源码里的 `afterSystemServer` 判断，直觉是"那就在 system_server 之前起，让它认为是 late"。

**结果**：无效，注入不发生。

**原因**：3114 的判断是

```java
afterSystemServer = ServiceManager.checkService("activity") != null
```

它检查的是"`activity` 服务在不在"，**没有参数可以强制这个结果**。抢跑只会让 daemon 在错误的时间点启动，检查依然返回 true。

**正确做法**：让**已经在跑的** daemon 经历一次 `binderDied`——`kill` 掉 zygote，daemon 内的 `binderDied()` 会把 `startedAfterSystemServer` 置 false、注销旧服务名、抢注新服务名。

**结论**：读源码时，看到"判断"要先确认它是**可被参数覆盖**还是**只读现状**。这一条浪费的时间最多。

---

## 死路 5 · 想靠 `execmem` 免重启完成注入

**动机**：如果能拿到 `execmem` 权限，理论上可以在不重启 zygote 的情况下完成注入。

**结果**：

```
avc denied  execmem
```

**原因**：这个权限属于 `system_server_startup` 域，**不是** `system_server`。注入发生时进程已经在 `system_server` 域里，拿不到 startup 阶段的权限。

**结论**：SELinux 的域是分阶段的，同一个进程在不同阶段属于不同域。查权限时要先确认"我现在在哪个域"。

---

## 两个反直觉现象（不是死路，但会误导判断）

### 现象 A · permissive 是"分层"的

物理写生效、`exec` 被放行之后，`service_manager` 仍然被拒。

原因：厂商用 `android_rvh_selinux_avc_lookup` / `is_initialized` 之类的 vendor hook，对特定对象做了单独保护。所以"enforcing=0 但某个操作还是失败"**不代表写入失败**，要按对象分开判断。

这个现象直接导致了一个错误结论——"permissive 是假的"。它半对半错：exec 层面是真的，`service_manager` 层面被架空了。两个事实同时成立，只看一边永远得不出结论。

### 现象 B · `rescue_boot_count` 从 3 降到 1

`kill` 掉 zygote 之后，崩溃救援计数器没有增加，**反而被重置了**。

原因：重启 zygote 不被系统算作 boot loop。

含义：这条操作路径的"崩溃预算"比想象中宽——但这不代表可以随便试，只是说明它不会因为一次 zygote 重启就触发 rescue。

---

## 一句话总结

前四条死路的共同点是**"从症状直接跳到解法"**：看到注册失败就去调 permissive，看到 `afterSystemServer` 就去抢跑，看到 `execmem` 就以为能免重启。
中间的每一跳都缺了一步——**先确认这个判断是谁做的、在什么阶段做的、能不能被覆盖**。
