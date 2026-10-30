# 根因：一条从来没有被加载的规则

> 这一节是整个过程中花时间最多的地方。结论只有一句，但推出来用了十几轮。

场景：让 **XRadiant**（一个作用于支付宝的 Xposed 模块，用来跑蚂蚁森林、庄园、会员这类日常任务）在锁 BL 的机器上生效。框架是 Vector 2.2 (3114)。

---

## 1. 症状

daemon 启动后注册框架的服务失败，`service list` 里看不到它。日志里没有明显的权限错误，只有注册返回失败。

第一反应是权限问题，于是去查 SELinux。查到的第一条线索非常有误导性：

```
/service_manager 的 add 被拒绝
```

顺着这条线索，"permissive 是不是假的"这个判断就成立了——因为 `enforcing` 明明已经写成 0，`exec` 也确实被放行了，为什么唯独 `service_manager` 还被拒？

这个判断是**部分正确**的，它把我们带向了正确的方向，但也让我们在"permissive 到底真不真"这个问题上耗了很久。

---

## 2. 五条事实

把状态一条条摊开，结论自然浮出来。

**事实 1**：`service list` 里没有我们注册的服务。

**事实 2**：该服务名在 `service_contexts` 里没有独立条目，因此落到默认标签 `default_android_service`。

**事实 3**：原厂策略里只有这一条与 add 相关的规则：

```
allow system_server system_server_service service_manager { add find }
```

**事实 4**：框架自己注册的服务（`serial` 这类）属于 `system_server_service` 组，所以它**一直有 add 权限**，从来不是问题所在。

**事实 5**：`default_android_service` 在整份策略里**没有任何 add 规则**。

推论：我们需要的服务名落在了一个天生没有 add 权限的标签下，而且策略里没有为我们补上这条规则。

---

## 3. 关键一步：去看框架自带的规则文件

框架发行包里带了一个 `sepolicy.rule`。打开一看：

```
allow system_server default_android_service service_manager find
```

**只有 find，没有 add。**

这条规则的来源是正常 root 环境：Magisk / KernelSU 会在开机时把模块目录下的 `sepolicy.rule` 合并进系统策略。框架作者写这条规则时，假定"加载策略"这件事由 root 管理器负责——在他们那个世界里这是成立的。

我们的世界里没有这个环节。所以这条规则**从来没有进过内核**。

---

## 4. 所以真正的卡点不是权限，是"没人加载"

到这里，之前所有异常现象都能解释了：

| 现象 | 解释 |
|---|---|
| `exec` 被放行 | permissive 是真的，exec 确实不受限 |
| `service_manager` 仍被拒 | 被厂商的 `android_rvh_selinux_avc_lookup / is_initialized` hook 单独保护，与全局 enforcing 状态分层 |
| `enforcing` 写成 0 无效（`EINVAL`） | 厂商改过 store 路径，sysfs 写入不生效，必须写内核变量 |
| 框架自注册服务失败 | 缺 `add` 规则，且策略里其余部分是原厂默认 |

**"permissive 是假的"这个判断之所以半对半错**：exec 层面它是真的，`service_manager` 层面它被 hook 架空了。这两个事实同时成立，所以只看其中一边永远得不出结论。

---

## 5. 突破口

真正的转折点是发现：`enforcing=0` 的时候，`load_policy` 的检查会被放行。

也就是说，permissive 状态下我们**可以往内核里灌一份新策略**——包括那条缺失的 `add` 规则。这是整条路唯一的入口，也是为什么顺序必须是"先 permissive，再加载策略"，而不能反过来。

加载后用策略总字节数验证：`1547892 → 1547904`，增加 12 字节，正好对得上新增规则。

---

## 6. 给别人的可迁移结论

1. **"权限被拒"和"规则不存在"是两种病，但症状一样。** 前者调策略，后者要先把策略加载进去——而后者在无 root 管理器环境下是常态。
2. **看第三方模块自带的 `sepolicy.rule` 假设了什么。** 它假设了 root 管理器存在。这个假设一旦不成立，规则就等于不存在，而且不会有任何报错。
3. **permissive 是可以分层的。** 全局开关为 0 不代表每个对象都放行，厂商 hook 可以在中间某一层单独拦截。判断时要按对象分开测，不要用一次成功推全局。
