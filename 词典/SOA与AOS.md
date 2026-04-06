# SOA 与 AOS

## 概念

AOS 和 SOA 是两种在内存中组织"一组对象的多个属性"的方式。

假设我们有 4 个粒子，每个粒子有 Position(x,y,z) 和 Age 两个属性。

---

## AOS — Array of Structures（结构体数组）

**思路**：每个粒子是一个结构体，所有粒子排成数组。

```cpp
struct Particle {
    float PosX, PosY, PosZ;
    float Age;
};
Particle particles[4];
```

**内存布局**（每格是一个 float）：

```
地址 →

[PosX₀][PosY₀][PosZ₀][Age₀] [PosX₁][PosY₁][PosZ₁][Age₁] [PosX₂][PosY₂][PosZ₂][Age₂] [PosX₃][PosY₃][PosZ₃][Age₃]
|_____ 粒子 0 _____|  |_____ 粒子 1 _____|  |_____ 粒子 2 _____|  |_____ 粒子 3 _____|
```

一个粒子的所有数据**紧挨着**。

---

## SOA — Structure of Arrays（数组的结构体）

**思路**：每个属性是一个独立数组，同一属性的所有粒子的值连续存放。

```cpp
struct ParticleData {
    float PosX[4];
    float PosY[4];
    float PosZ[4];
    float Age[4];
};
```

**内存布局**：

```
地址 →

[PosX₀][PosX₁][PosX₂][PosX₃] [PosY₀][PosY₁][PosY₂][PosY₃] [PosZ₀][PosZ₁][PosZ₂][PosZ₃] [Age₀][Age₁][Age₂][Age₃]
|______ 所有粒子的 X ______| |______ 所有粒子的 Y ______| |______ 所有粒子的 Z ______| |____ 所有粒子的 Age ____|
```

同一属性分量的数据**紧挨着**。

---

## 直观对比

```
AOS（按粒子排列）：          SOA（按属性排列）：

粒子0: [X Y Z Age]          X:   [X₀ X₁ X₂ X₃]
粒子1: [X Y Z Age]          Y:   [Y₀ Y₁ Y₂ Y₃]
粒子2: [X Y Z Age]          Z:   [Z₀ Z₁ Z₂ Z₃]
粒子3: [X Y Z Age]          Age: [A₀ A₁ A₂ A₃]
```

---

## 为什么 Niagara 选择 SOA？

### 原因一：SIMD 友好

CPU 的 SIMD 指令（SSE/AVX）可以一次处理 4 或 8 个 float。

**场景**：给所有粒子的 X 坐标加上速度。

SOA 下：
```
PosX: [X₀ X₁ X₂ X₃]  ← 连续的 4 个 float，一条 SIMD 指令搞定
VelX: [Vx₀ Vx₁ Vx₂ Vx₃]

SIMD_Add(PosX, VelX) → 一次操作 4 个粒子
```

AOS 下：
```
[X₀ Y₀ Z₀ Age₀] [X₁ Y₁ Z₁ Age₁] ...
 ↑                 ↑
 X₀ 和 X₁ 之间隔了 3 个 float
```

X 坐标不连续，需要先 gather（从不同位置收集数据），SIMD 效率大打折扣。

### 原因二：GPU Compute Shader 友好

GPU 的线程以 Warp/Wavefront（32/64 个线程）为单位执行。当同一 Warp 的线程访问**连续内存**时，硬件可以合并为一次内存事务（coalesced access）。

SOA 下，线程 0-31 同时读 PosX[0]-PosX[31]，地址连续 → **一次内存事务**。

AOS 下，线程 0 读地址 0，线程 1 读地址 16（跳过了 Y,Z,Age）→ **跨步访问，多次内存事务**，带宽浪费严重。

### 原因三：灵活的属性集

不同 Emitter 可能有完全不同的属性：
- Emitter A：Position, Velocity, Color, Age（4 个属性）
- Emitter B：Position, Velocity, Size, Rotation, SubImage（5 个属性）

SOA 下，每个属性是独立的数组，**增删属性只需要增删数组**，不影响其他属性的内存布局。

AOS 下，增删属性意味着**改变结构体大小**，所有粒子的数据都要重新排列。

### 原因四：按需访问，不浪费缓存

如果一个模块只需要读 Age（比如判断粒子是否死亡）：

SOA 下：只读 Age 数组，缓存里全是有用数据。

AOS 下：读一个粒子的 Age，缓存行里同时加载了 Position 和其他不需要的数据 → **缓存污染**。

---

## AOS 的优势（Niagara 没选它的原因也在这里）

| 场景 | AOS 更好 | SOA 更好 |
|---|---|---|
| 访问单个粒子的所有属性 | ✅ 数据连续 | ❌ 需要跳转多个数组 |
| 批量处理同一属性 | ❌ 跨步访问 | ✅ 连续访问 |
| SIMD/GPU 并行 | ❌ 不友好 | ✅ 非常友好 |
| 动态增删属性 | ❌ 需要重排 | ✅ 增删数组即可 |
| 序列化单个对象 | ✅ 直接拷贝 | ❌ 需要从多个数组收集 |

粒子系统的典型操作是**"对所有粒子执行同一操作"**（如更新位置、计算颜色），而不是"访问单个粒子的所有属性"。所以 SOA 是更合适的选择。

---

## Niagara 中的具体实现

在 `FNiagaraDataBuffer` 中：

```cpp
// 三个大的字节数组，按分量类型分开
TArray<uint8> FloatData;   // 所有 float 分量
TArray<uint8> Int32Data;   // 所有 int32 分量
TArray<uint8> HalfData;    // 所有 half 分量
```

内存结构（以 FloatData 为例，假设有 3 个 float 分量、4 个粒子）：

```
FloatData:
  偏移 0:                          偏移 FloatStride:                  偏移 FloatStride*2:
  [分量0: inst0 inst1 inst2 inst3]  [分量1: inst0 inst1 inst2 inst3]  [分量2: inst0 inst1 inst2 inst3]
  |_________ PosX ______________|  |_________ PosY ______________|  |_________ PosZ ______________|
```

访问第 i 个粒子的第 j 个 float 分量：
```cpp
float* value = (float*)(FloatData.GetData() + FloatStride * j) + i;
```

`FloatStride` 是每个分量列的字节宽度，对齐到 `VECTOR_WIDTH_BYTES`（通常是 16 字节 = 4 个 float），确保 SIMD 操作不会越界。
