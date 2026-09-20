---
title: "用羊群理解 Entropy：从微观排列到时间箭头"
date: 2026-09-20T09:45:00+08:00
draft: false
tags: ["Entropy", "Thermodynamics", "Statistical Mechanics", "Physics", "Probability"]
categories: ["Architecture", "Tools"]
author: "Kaka"
description: "借助羊群与农场的直观模型，理解 microstate、macrostate、Boltzmann entropy、热力学第二定律，以及宏观时间为何具有方向。"
---

## 引言

冰块会在温水里融化，却不会在常温水中自动重新拼成方正的冰块；鸡蛋会摔碎，却不会从地面跃起并恢复完整。奇怪的是，如果只观察单个原子或分子的运动，把影片倒放，许多微观运动依然符合力学定律。微观规律大体不偏爱时间方向，宏观世界却明显只朝一个方向演化。

Aatish Bhatia 的交互式文章 [Entropy Explained, With Sheep](https://aatishb.com/entropy/) 用羊群解释了这个落差：所谓 entropy 增加，核心不是世界受到某种神秘力量驱使而“越来越乱”，而是系统压倒性地更可能进入拥有更多 microstate 的 macrostate。本文沿用这个比喻，以组合计数和统计力学重新讲清这条逻辑链。

## 三只羊为什么有十种分布

想象三个相同的农场分区，以及三只暂时不区分个体的羊。羊可以全部挤在一个分区，也可以分散在多个分区。我们只记录每个分区有几只羊，例如 `(3, 0, 0)`、`(1, 1, 1)`，共有十种非负整数解：

`x₁ + x₂ + x₃ = 3`

这就是 stars and bars 问题。把 `q` 个相同对象放进 `N` 个可区分容器，排列数为：

`Ω(N, q) = C(q + N - 1, q)`

代入 `N = 3`、`q = 3`：

`Ω(3, 3) = C(5, 3) = 10`

这个农场不是为了研究畜牧业。把“羊”换成离散的 energy packet，把“分区”换成 solid 中可以持有能量的 atom，它就成为简化的 Einstein solid model。加热固体相当于增加 energy packet，packet 在 atom 之间交换；atom 数或 energy packet 数一旦增大，可能的微观排列会极快增长。

例如 30 个 energy packet 分配给 30 个 atom：

`Ω(30, 30) = C(59, 30) = 59,132,290,782,430,712`

仅仅 30 个 atom 就已经有约 5.9 京种排列，而日常物体的粒子数接近 `10²³` 到 `10²⁵` 量级。宏观不可逆性正是从这种巨大数量级中出现的。

## Macrostate 与 microstate

要准确理解 entropy，必须区分两个层次：

- **Macrostate**：我们能从外部测量的整体状态，例如 pressure、volume、temperature，或者两个物体各自拥有多少 energy。
- **Microstate**：构成系统的粒子具体位于哪里、如何运动，或者每一份 energy 究竟落在哪个 atom 上。

许多不同 microstate 会表现为相同 macrostate。一只气球可以保持同样的 pressure、volume 和 temperature，内部气体分子却可以拥有数量惊人的位置与速度组合。统计力学中的 Boltzmann entropy 写作：

`S = k_B ln Ω`

其中 `S` 是 entropy，`k_B` 是 Boltzmann constant，`Ω` 是与某个 macrostate 相容的 microstate 数量。说“entropy 是排列数”有助于建立直觉，但严格来说它是 multiplicity 的对数乘以常数。取 logarithm 还有一个重要结果：两个独立系统组合时 microstate 数相乘，而 entropy 相加。

`Ω_AB = Ω_A × Ω_B  ⇒  S_AB = S_A + S_B`

“混乱度”只是一个容易误导的日常比喻。更可靠的表述是：在已知宏观约束下，系统可能对应多少种微观实现。

## 拆掉两座农场之间的围栏

现在有两座农场，每座分成三个区域，总共六只羊。最初六只羊都在农场 A，农场 B 一只也没有；对应到物理模型，就是一个 solid 拥有六份 energy，另一个处于低能状态。两者接触后可以交换 energy。

设农场 A 有 `q_A` 只羊，农场 B 有 `6 - q_A` 只。某个 macrostate 的总 multiplicity 是两边排列数的乘积：

`Ω_total(q_A) = Ω(3, q_A) × Ω(3, 6 - q_A)`

| 分配 `(q_A, q_B)` | A 的排列 | B 的排列 | 总 microstate 数 |
|---|---:|---:|---:|
| `(0, 6)` | 1 | 28 | 28 |
| `(1, 5)` | 3 | 21 | 63 |
| `(2, 4)` | 6 | 15 | 90 |
| `(3, 3)` | 10 | 10 | 100 |
| `(4, 2)` | 15 | 6 | 90 |
| `(5, 1)` | 21 | 3 | 63 |
| `(6, 0)` | 28 | 1 | 28 |

所有列相加得到 462 个 microstate。能量平均分配的 `(3,3)` 拥有 100 个，而能量全部挤在一边的状态各只有 28 个。如果每个可达 microstate 近似等可能，随机演化时最常观察到的自然是中间区域。

这里没有牧羊人把羊赶向平均分布，也没有额外的微观定律命令热量从热物体流向冷物体。只是“能量分散”对应的 microstate 远多于“能量集中”。高 entropy macrostate 因而更可能出现。

## 用 Python 验证 462 个 microstate

下面的程序直接计算每一种 energy split 的 multiplicity：

```python
from math import comb


def multiplicity(atoms: int, packets: int) -> int:
    """Einstein solid: q 个 energy packet 分配给 N 个 atom。"""
    return comb(packets + atoms - 1, packets)


atoms_per_solid = 3
total_packets = 6
states = []

for packets_a in range(total_packets + 1):
    packets_b = total_packets - packets_a
    omega = (
        multiplicity(atoms_per_solid, packets_a)
        * multiplicity(atoms_per_solid, packets_b)
    )
    states.append((packets_a, packets_b, omega))

for state in states:
    print(state)

print("total microstates:", sum(omega for _, _, omega in states))
```

输出如下：

```text
(0, 6, 28)
(1, 5, 63)
(2, 4, 90)
(3, 3, 100)
(4, 2, 90)
(5, 1, 63)
(6, 0, 28)
total microstates: 462
```

这段代码展示的不是 dynamical simulation，而是状态空间的静态计数。要从计数进一步得到真实系统的时间行为，还需要 ergodicity、interaction 和可达状态等假设；“所有 microstate 等可能”也不是脱离条件永远成立的口号。但在这个受控模型里，它足以揭示第二定律的统计基础。

## 小系统真的可能 entropy 下降

在只有六只羊的模型中，极端状态并不罕见。任一指定极端的概率是 `28 / 462`，两个极端合计约为 12.1%，接近八分之一。因此系统会在高低 entropy 之间明显 fluctuation：偶尔所有 energy 又集中到一侧，并不违反微观物理。

系统变大后，分布会迅速在 equilibrium 附近形成尖锐峰值。极端状态的数量仍可能很大，但它相对于全部 microstate 的比例会变得小到无法在现实时间尺度内观察。冰块中约有 `10²⁵` 量级的 molecule；在这种尺度上，自发从高 entropy macrostate 回到特定低 entropy macrostate 的概率不是数学上的严格零，却小得可以视为不会发生。

所以“entropy 总是增加”是宏观极限下极其可靠的统计规律。它不是说微观 fluctuation 被禁止，而是说粒子数足够大时，entropy 明显下降的概率被组合数压到近乎为零。这也解释了为什么 nanoscale system 中可以观测到短时 entropy reduction，而日常尺度中看不到碎鸡蛋自动复原。

## 时间箭头从概率中涌现

把单个粒子的轨迹倒放，通常仍能得到合法轨迹；把一杯水中所有 molecule 精确反转，理论上也可能让它重新聚成冰块。但要实现这一点，需要把系统放进一个极其特殊、极低概率的 microstate。任何微小扰动都会让它重新滑向占据绝大多数状态空间的 equilibrium 区域。

宏观的 arrow of time 因而不是每个粒子都携带一个“只能向未来运动”的标记，而是系统从低概率 macrostate 走向高概率 macrostate 的整体趋势。我们之所以能区分影片正放和倒放，是因为记忆、破碎、扩散、摩擦和热传导都留下了 entropy increase 的痕迹。

这里还隐藏着一个更深的问题：如果宇宙总是趋向更高 entropy，早期宇宙为什么处于如此特殊的 low-entropy state？第二定律能够解释给定低 entropy 初态之后的演化方向，却不能单独解释这个初态为何存在。这仍然连接着 cosmology 中关于初始条件和 gravity entropy 的开放问题。

## 生命没有违反第二定律

生命体能建立高度有序的局部结构，看起来像在降低 entropy，但地球不是 isolated system。太阳向地球提供相对集中的低 entropy energy；生物和机器用它完成工作，再把更分散的 heat 辐射到环境和太空。局部 entropy 可以下降，只要 system 与 surroundings 的总 entropy 上升。

同样，冰箱能让内部降温，却需要电力并向房间排放更多 heat。第二定律约束的是合适边界下的总系统，不能只挑一个局部区域计账。

在极遥远的未来，如果宇宙逐渐接近 thermal equilibrium，可用于做功的 temperature gradient 会越来越少。所谓 heat death 不是“所有 energy 消失”，而是 energy 过度均匀地分布，难以再驱动持续的结构变化。entropy 描述的因此不只是“有多少 energy”，还涉及这些 energy 以多少种方式分布，以及其中还有多少可利用的差异。

## 常见误解

| 误解 | 更准确的理解 |
|---|---|
| Entropy 就是肉眼看到的混乱 | Entropy 与满足宏观约束的 microstate multiplicity 有关 |
| 第二定律禁止任何 entropy decrease | 小系统可出现 fluctuation；isolated macroscopic system 的显著下降极不可能 |
| 有序结构一定违反第二定律 | 局部系统可以输出更多 entropy 到 surroundings |
| 热量从热端流向冷端是额外的神秘力 | 均匀分配对应压倒性更多的可达 microstate |
| Equilibrium 代表没有微观运动 | 粒子仍在运动，只是宏观量稳定且缺少可用 gradient |

## 总结

羊群模型把 entropy 的核心压缩成三步：宏观状态背后存在大量 microstate；不同 macrostate 对应的 microstate 数量差异巨大；系统随机探索可达状态时，几乎必然落入 multiplicity 最大的区域。Boltzmann 公式 `S = k_B ln Ω` 把这个计数变成可加的物理量，而 thermodynamic arrow of time 则从海量粒子的统计行为中涌现。

冰不会自己从温水中重组，不是因为倒放轨迹违反微观定律，而是因为那条轨迹要求不可思议地精确、不可思议地罕见的初始排列。第二定律最深刻的地方，正是它让“几乎不可能”在宏观尺度上表现得像“绝对不会”。

参考资料：

- [Entropy Explained, With Sheep — Aatish Bhatia](https://aatishb.com/entropy/)
- [Entropy Explained, With Sheep — Engineers Edge](https://www.engineersedge.com/thermodynamics/entropy_explained_with_sheep_15961.htm)

> 本文根据 Aatish Bhatia 的交互式讲解进行中文重写，并补充组合公式、Boltzmann entropy 与 Python 示例，不是逐句翻译。
