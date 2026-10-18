# DroidDeck 天玑 / KPad 适配

日期：2026-10-18。当前阶段：**实验分支准备与适配评估；尚未完成 DroidDeck 天玑实机验收。**

## 正在实施：进入 Steam

官方 0.4.0 已在 Redmi K Pad 上完成原版对照：运行时 r9 与 ARM64 Proton 准备成功，但 Android 显示侧选择 `turnip-sdk36` 后没有物理设备，Linux guest 的 `freedreno_icd.json` 也在 gamescope 中报找不到物理设备；两次均无首帧并返回前端。设备当前为 Android 17 / API 37、Mali-G925。

本分支开始接入和验证非 Adreno 路线。CI 已配置在 `dimensity/kpad-panvk` 提交时构建，使用 `-PdimensityBuild=true` 生成 `com.droiddeck.launcher.dimensity`，显示为 DroidDeck Dimensity，避免与官方签名的原版包发生覆盖冲突。构建仍执行原有运行时输入、APK内容、native依赖、单元测试和签名检查；构建成功与实机验收分别记录。签名发布与公告仍只在原上游的 main 条件中运行。

验收目标为：Android 显示与 Linux guest 均能使用可用驱动、实际进入 Steam 登录或大屏界面并保持出画、可操作，退出后再次启动仍正常。目前这些修复验收尚未完成。

本分支目标是 Redmi K Pad（Dimensity 9400+ / Mali-G925）。其他天玑型号、Mali 代际与内核版本需分别验证。

## 分支来源

- 上游：[Droid-Deck/DroidDeck](https://github.com/Droid-Deck/DroidDeck)，GPL-3.0。
- 用户 fork：[2658214225/DroidDeck](https://github.com/2658214225/DroidDeck)。
- 开发分支：`dimensity/kpad-panvk`。
- 基线：上游 `test/non-adreno-all`，commit [c9ebb247fe97bb0812beef07ee05c080110a56d1](https://github.com/Droid-Deck/DroidDeck/commit/c9ebb247fe97bb0812beef07ee05c080110a56d1)。
- 创建时上游 `main`：`45513c90d620249157542291432ecb479bfbe51f`。

选择非 Adreno 实验分支，是为了继承已有的 GPU 检测、开放驱动选择与显示接入工作。此基线中的 `DeviceSupport.kt` 已有 Mali 检测和开放驱动路径；单独放开设备检查不能证明图形或游戏可用。本次分支准备只新增适配说明与 README 入口，没有移植 KPad 运行时代码、加入驱动二进制或生成 APK。

基线的 `DriverPairs.kt` 已列出 `nad:panvk-mali-kbase`（名称覆盖到 Mali-G925），并对其显示驱动设置 Android 15 / API 35 下限。这说明分支已有相关驱动选择入口，不等于该二进制在 KPad 内核或游戏场景中已经验证。

## 已有公开工作与证据范围

以下为 2026-10-18 的检查结果，后续状态可能变化。

| 工作 | 当时证据 | 可得出的结论 |
|---|---|---|
| [Issue #218](https://github.com/Droid-Deck/DroidDeck/issues/218) | Dimensity 9400 / Mali-G925 实验中，guest PanVK 可被 Vulkan 工具识别，vkcube 能经 Wayland 呈现；该实验的 gamescope 因缺少 DRM 节点失败 | 有相关芯片的底层实验，不能据此称 Steam 或 Windows 游戏可玩；这也是历史现场，当前分支应重新验证 |
| [PR #978](https://github.com/Droid-Deck/DroidDeck/pull/978) | 作者报告 Pixel 10a / Tensor G4 / Mali-G715 上 Steam Big Picture 可运行，PR 说明叠加 #975，并以 `test/non-adreno-all` 为目标 | 可参考 Mali 的 PanVK 配对、Venus 与显示路线；单设备报告不能外推到天玑。本分支不把开放 PR 的改动自动视为已包含 |
| [PR #570](https://github.com/Droid-Deck/DroidDeck/pull/570) | 面向非 Adreno 的大型通用 Vulkan 后端方案，检查时尚未合并 | 是另一条研究路线；不宜未经对照把其全部变更合入天玑分支 |

PR #975 与 #978 的改动均不在上述基线中；后续需单独评估、审查和验证，不能把其设备报告当作本分支的验收结果。

## KPad 材料如何复用

KPad 已有 Dimensity 9400+ / Mali-G925 的 PanVK 接入与真实 Windows 游戏场景记录。其运行架构与 DroidDeck 不同，经验可以迁移，验收结论不能跨架构沿用。

下列 KPad 路径相对于 KPad Android 工程，未在本次分支准备中复制文件。

| KPad 材料 | 可借鉴内容 | DroidDeck 接入点与限制 |
|---|---|---|
| `app/app/src/main/java/com/winlator/core/PanVkDriver.java` | 精确驱动身份、SHA-256 固定、完整性检查、原子激活、失败关闭 | 对照 `app/src/main/java/com/droiddeck/launcher/gpu/LinuxVulkanDriverManager.java` 与 `DriverPairs.kt`；沿用 DroidDeck 自己的驱动格式和激活协议 |
| `PanVkVulkanFrontend.java` 与 `app/app/src/main/cpp/winlator/src/panvk_vulkan_frontend.c` | ELF/SONAME 验证、驱动来源追踪、WSI 错误诊断 | KPad 使用 Android/Bionic、libvkhost 与 X11/XCB 接入；DroidDeck 的 glibc guest / gamescope / Wayland 不能直接使用该共享库 |
| `VkCompatBackend.java` | 阻止配置误选后端，校验架构、哈希与依赖，避免静默回退 | KPad 的环境变量、JNI 和 host/payload 协议不是 DroidDeck 的驱动接口 |
| KPad 原版 PanVK RC3 交付与回归记录 | 保留准确驱动来源，分开证明安装、Vulkan、呈现、输入和游戏世界行为 | KPad 的 Android 驱动发行物不能直接充当 DroidDeck 的 Linux/glibc ICD；KPad 上通过不等于 DroidDeck 上通过 |

CPU 都是 ARM64、GPU 都属于 Mali，并不意味着共享库 ABI 相同。需要分别核对 Bionic/glibc、动态链接器和依赖、Vulkan ICD 入口、WSI 后端，以及 PanVK-kbase 与设备内核接口的匹配。Linux guest 驱动和 Android 显示驱动也必须记录各自身份。

## 后续实施与验收

1. **核对目标设备与驱动配对。** 记录 Android/内核、SoC/GPU、Mali 设备节点和 Vulkan 扩展；确认可用于该内核的 glibc PanVK 与 Android 显示驱动来源、版本、哈希及许可。
2. **先打通 DroidDeck 自身的显示路线。** 在 LinuxVulkanDriverManager、DriverPairs 与 `app/src/main/java/com/droiddeck/launcher/session/VenusComponent.kt` 的现有机制中选择适合目标设备的路径，分别验证 guest 渲染与 Android 侧显示。KPad 的 X11 前端不能替代 gamescope/Wayland 所需的适配。
3. **验证 Steam 大屏。** 必须观察客户端持续呈现并响应输入；Vulkan 枚举成功或一个窗口出画只属于更早的验证层。
4. **验证 Windows 游戏。** 进入实际游戏世界，分别取证渲染、声音、角色移动/视角和重复启动稳定性，再对外声明具体设备与游戏的兼容范围。BCn 等纹理格式处理应单独验证，不默认启用曾出现异常的转码路线。

目前 DroidDeck 在目标 Redmi K Pad 上的上述验收均未进行。本次没有操作平板、安装 APK、改变系统设置或操作 Steam 存档。

## 许可与资产来源

DroidDeck 保留上游 GPL-3.0 与原有版权声明。KPad 源码树的根 LICENSE 为 LGPL-2.1；实际移植前仍须逐文件核对版权、许可证与第三方来源，不能把整个工程中的资产视为自有代码。

PanVK、Wine/Proton、FEX 及配套库分别属于其上游。本次没有上传 KPad APK、预编译驱动、专有组件、设备日志或用户数据。之后分发二进制时，必须同时满足相应源码提供和许可声明义务。
