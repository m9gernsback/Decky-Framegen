# Steam Deck 上的 FSR4 变体与帧生成使用指南

> 整理自 2026-10-21 的问答，面向在 Steam Deck（RDNA2 核显）上通过 Decky-Loader 使用本插件的场景。

## 1. FSR4 运行时变体含义

插件提供 4 个 FSR4 变体（定义见 src/utils/constants.ts:62-83、main.py:60-126）。旧版插件的显示名与当前名称对照：

| 旧版显示名 | 当前名称 | 说明 |
|---|---|---|
| 4.0.2c \| RDNA2/3 Mod | 4.0.2c \| RDNA2/3 Compatibility | 修改版 FSR4 4.0.2c INT8 DLL，让 RDNA2/3 跑 FSR4；兜底用 |
| 4.1.0 \| RDNA4 Official | 4.1.1 \| FFX 2.3 SDK | OptiScaler 官方包自带升频器；官方支持 RDNA4(FP8)/RDNA3 桌面(INT8)，Deck 不适用 |
| 4.1.1 \| RDNA3/4 Official | 4.1.1 \| Driver Override | 官方升频器 + 单独打包的 4.1.1 amdxcffx64.dll 驱动覆盖；RDNA3/4 测试/回退路径 |
| 4.1.1 \| RDNA2 Mod | 4.1.1 \| Valve RDNA2 Compatibility | **Steam Deck 首选**：4.1.1 SDK 升频器 + Valve amdxcffx64.dll/amdxc64.dll + pre10 OptiScaler 注入器 + RDNA2 INI 覆盖（强制 Fsr4ForceModel=2） |

Steam Deck（RDNA2）建议：首选 Valve RDNA2 Compatibility；个别游戏不稳定时退回 4.0.2c。

## 2. 游戏内 OptiScaler 菜单（Insert 键）

插件已预设关键项（main.py:684-709）：FGInput=nukems、FGOutput=nukems、Fsr4Update=true、LoadAsiPlugins=true、UseHQFont=false。

菜单中只需确认：
- Dx12Upscaler → FSR 3.1（因 Fsr4Update=true 自动升级为 FSR4）
- DX11 游戏：Dx11Upscaler → FSR 2.2 或 XeSS（无 FSR4）
- Vulkan：VulkanUpscaler → FSR 3.1
- 然后在游戏图形设置里开启 DLSS + DLSS Frame Generation（FakeNVAPI 使其在 AMD 显卡上可选）

验证：开 FSR_Fsr4EnableWatermark=true，角落水印显示 FSR4 / FSR4-i8 为成功，FSR3 表示回退。

启动项预设示例：

    Dx12Upscaler=fsr31 FSR_Fsr4EnableWatermark=true ~/fgmod/fgmod %command%

## 3. 启动项中 SteamDeck=0 的作用

Steam 在 Deck 上启动游戏时自动设 SteamDeck=1，许多游戏读到后会锁定“Steam Deck 预设”、隐藏 DLSS 等高级图形选项。插件统一加 SteamDeck=0（main.py:1616-1618、defaults/assets/fgmod.sh:476）让游戏按普通 PC 显示完整图形菜单——本插件方案需要在游戏内选 DLSS，故必须如此。副作用：游戏不再自动套用 Deck 优化预设，需手动调画质。

## 4. 帧生成：OptiScaler FG vs LSFG（decky-lsfg-vk）

- **OptiScaler FG**（DLSSG→FSR3，nukems 路径）：基于运动矢量，画质好、HUD 不参与插帧、延迟低；但逐游戏配置、仅支持带 DLSS FG 的 DX12 游戏（OptiFG 为高度实验性兜底）
- **LSFG**：合成器层 shader 插帧，任意游戏可用、无需游戏支持；无运动矢量、UI 会被插帧、需稳定锁帧（典型 30→60 或 45→90）

选择建议：支持 DLSS FG 的游戏用本插件；不支持/老游戏/想省事用 LSFG。**两者不可同时开**。

## 5. DX11 游戏与 FG

DX11 基本无法用本插件插帧：DLSS FG 是 DX12 专属，无入口可转换；FSR4 升频也是 DX12 only（DX11 最多 FSR 2.2/XeSS 升频）。可实验 FGInput=fsrfg FGOutput=xefg（捆绑了 libxess_fg.dll），成功率低。Deck 上 DX11 游戏插帧直接交给 LSFG。

## 6. 为什么 YouTube 博主多用 OptiScaler 只做 FSR4 超分、插帧交给 LSFG

即使是支持 DLSS FG 的游戏（对马岛、剑星、2077），Deck 社区普遍如此，原因：
1. DLSSG→FSR3 转换依赖游戏版本，**游戏更新易失效/崩溃**，维护成本高
2. FSR3 FG 真实占用 GPU，**吃基础帧率**；LSFG 开销小，配合锁帧降低负载和功耗（掌机敏感）
3. 转换层 FG 偶有微卡顿；LSFG + 硬锁帧给出**均匀帧间隔**，体感更稳
4. FSR3 FG 需搭配 Reflex 抗延迟，转换层支持不完整；LSFG 是系统级开关、一局一设
5. FSR4 的画质提升是最显眼收益；而 30→60 区间两种插帧在 800p 小屏观感差距不大，故选省事稳定方案

桌面大显存 PC 上结论相反，通常选原生 FSR3 FG。
