# P2: BF16 Hadamard gated activation 测试被标记 flaky 且缺少诊断信息

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P2 |
| 组件 | `uzu-engine` / Metal-CPU kernel correctness |
| 主要文件 | `crates/uzu-engine/unit/backends/common/kernel/gated_act_mul_test.rs` |
| 状态 | 待复现 / 待诊断 |

## 概述

以下测试被长期忽略：

```rust
#[uzu_test]
#[ignore] // TODO: flaky
fn test_gated_act_mul_interleaved_hadamard_bf16() {
    let input = interleaved_input::<bf16>(ActivationType::SILU);
    let expected = run_interleaved::<bf16, Cpu>(&input, true);
    for_each_non_cpu_backend!(|B| {
        let actual = run_interleaved::<bf16, B>(&input, true);
        assert_eq_float::<bf16>(&expected, &actual, 0.02, "Hadamard gated activation mismatch");
    });
}
```

该测试比较 CPU 与非 CPU 后端在 BF16、interleaved、Hadamard、SILU 组合下的输出，容差 `0.02`。

问题不在“测试被 ignore”本身，而在于：

1. 没有记录失败硬件/后端；
2. 没有失败样本或误差分布；
3. 没有复现频率；
4. 失败断言只给出统一 message；
5. 无法判断是 kernel correctness 差异还是测试设计问题。

## 为什么重要

被忽略的 correctness 测试可能掩盖真实 kernel bug。BF16 和 Hadamard 常用于生产模型路径；如果差异超出可接受范围，可能影响生成质量。即使最终确认是精度差异，也应把它建模成明确容差，而不是随机失败。

## 可能原因

1. **BF16 计算顺序差异**
   - CPU 和 Metal 的乘加顺序不同；
   - Hadamard transform 后误差累积不同。

2. **测试数据未控制边界**
   - 输入由 `sin` 生成，范围较小；
   - 但 `f32 -> bf16` 转换与 activation 可能在特定位置放大误差。

3. **异步 command buffer 等待问题**
   - 测试显式 `wait_until_completed()`，但仍需确认 buffer readback 是否在所有后端上严格完成。

4. **Hadamard factors 数据问题**
   - 测试使用全 1 factors；
   - 可能未覆盖 kernel 实际使用的变换路径。

## 建议诊断步骤

### 1. 生成失败报告

修改断言辅助信息，输出：

- backend；
- data type；
- activation；
- input index；
- expected；
- actual；
- error；
- max error；
- mean error。

### 2. 固定输入种子

将输入生成改为确定性 PRNG，并输出 seed。当前 `sin(index)` 是确定的，但加入更丰富的输入范围时应保持可复现。

### 3. 多硬件矩阵

记录：

- CPU 型号；
- GPU 型号；
- macOS/Metal 版本；
- backend；
- 重复次数；
- 失败次数。

### 4. 分别验证 no-Hadamard 与 Hadamard

当前 no-Hadamard BF16 测试没有 ignore。若只有 Hadamard 失败，优先比较 CPU hadamard transform、Metal SIMDGroup transform 和 factor load/store。

### 5. 比较中间输出

如果最终误差超阈值，为 kernel 增加临时 debug 输出或分阶段测试：

1. gate activation；
2. hadamard transform；
3. interleaved store。

## 建议修复方向

1. **若为可接受数值差异**
   - 为 BF16 + Hadamard 定义后端特定容差；
   - 说明容差依据；
   - 移除 `#[ignore]`。

2. **若为 kernel bug**
   - 修复 Metal/Slang 实现；
   - 保留该测试作为回归；
   - 增加失败样本作为 fixture。

3. **若为测试竞争**
   - 修复 command buffer 等待/readback；
   - 在 helper 中确保 buffer lifetime 覆盖 readback。

## 验证命令

```bash
cargo +nightly test -p uzu-engine test_gated_act_mul_interleaved_hadamard_bf16 -- --ignored
cargo +nightly nextest run -p uzu-engine backends::common::kernel
```

## 提 issue 前

至少收集一次失败输出和硬件信息。否则无法判断是否只在某代 Apple GPU 出现、是否与 CI runner 相关、是否是固定容差不足。

