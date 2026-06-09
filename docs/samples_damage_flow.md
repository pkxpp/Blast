# Blast Samples Damage 流程分析

本文档梳理 `samples/SampleBase` 中 damage（伤害）从触发到破碎的完整流程。

## 概览：两条 Damage 路径

Samples 中存在两条独立的 damage 路径：

| 路径 | 触发方式 | 入口 | 时机 |
|------|---------|------|------|
| **手动伤害** | 鼠标点击/拖拽 | `DamageToolController` → `BlastController::deferDamage` / `immediateDamage` | 用户交互时 |
| **碰撞伤害** | PhysX 碰撞回调 | `ExtImpactDamageManager::onContact` → `applyDamage` | 每帧 `Animate` 自动执行 |

两条路径最终都通过 `NvBlastDamageProgram`（包含 graph shader + subgraph shader）作用于 `TkActor`。

---

## 路径一：手动伤害（DamageToolController）

### 流程

```
用户鼠标点击
    │
    ▼
DamageToolController::Animate()
    │  射线检测 (raycast) 命中 Blast actor
    │  计算命中位置、法线、武器方向
    │
    ▼
BlastController::overlap(geometry, pose, hitCall)
    │  遍历所有 BlastFamily
    │  每个 Family 用 PhysX overlap 查询命中的 ExtPxActor
    │
    ▼
hitCall(actor, family) ─── 即 damageFunction lambda
    │  构造 damageDesc (如 NvBlastExtRadialDamageDesc)
    │  归一化伤害: material->getNormalizedDamage(damage)
    │
    ├──► BlastController::deferDamage()     [大多数 damage 类型]
    │       │  将 damageDesc 拷入 m_damageDescBuffer
    │       │  构造 NvBlastExtProgramParams { desc, material, accelerator }
    │       │  拷入 m_damageParamsBuffer
    │       │
    │       └──► TkActor::damage(program, programParams)
    │               │  标记 damage，不立即计算 fracture
    │               │  等待 Group::process() 统一处理
    │
    └──► BlastController::immediateDamage() [Impact Spread 类型]
            │  构造 NvBlastExtProgramParams
            │
            ├──► TkActor::generateFracture(&fractureEvents, program, params)
            │       立即计算哪些 bond/chunk 需要断裂
            │
            └──► TkActor::applyFracture(nullptr, &fractureEvents)
                    立即应用断裂结果
```

### Damage Tool 类型

`DamageToolController` 注册了 7 种 damage 工具（按键 1-7 切换）：

| # | 名称 | Shader | DamageDesc 类型 | 执行方式 |
|---|------|--------|----------------|---------|
| 1 | Radial Damage (Falloff) | `NvBlastExtFalloffGraphShader` | `NvBlastExtRadialDamageDesc` | defer |
| 2 | Radial Damage (Cutter) | `NvBlastExtCutterGraphShader` | `NvBlastExtRadialDamageDesc` | defer |
| 3 | Slice Damage | `NvBlastExtTriangleIntersectionGraphShader` | `NvBlastExtTriangleIntersectionDamageDesc` | defer |
| 4 | Capsule Damage (Falloff) | `NvBlastExtCapsuleFalloffGraphShader` | `NvBlastExtCapsuleRadialDamageDesc` | defer |
| 5 | Impact Spread Damage | `NvBlastExtImpactSpreadGraphShader` | `NvBlastExtImpactSpreadDamageDesc` | **immediate** |
| 6 | Shear Damage | `NvBlastExtShearGraphShader` | `NvBlastExtShearDamageDesc` | defer |
| 7 | Stress Damage | 无 shader | 无 | stress solver |

### defer vs immediate 的区别

- **deferDamage**: `TkActor::damage()` 只是将 damage 请求入队。实际的 `generateFracture` + `applyFracture` 在后续 `Group::process()` 中批量执行（支持多线程）。
- **immediateDamage**: 当场调用 `generateFracture` + `applyFracture`，结果立即生效。用于 Impact Spread 这类需要单线程 + DamageAccelerator 的 shader。

---

## 路径二：碰撞伤害（Impact Damage）

### 流程

```
PhysX 碰撞检测
    │
    ▼
EventCallback::onContact(pairHeader, pairs, nbPairs)
    │  (注册为 PxScene 的 SimulationEventCallback)
    │
    ▼
ExtImpactDamageManager::onContact(...)
    │  内部累积碰撞产生的冲量信息
    │  将 damage 请求入队
    │
    ▼ (每帧在 BlastController::Animate 中调用)
ExtImpactDamageManager::applyDamage()
    │  对累积的 damage 进行处理:
    │  1. 计算 damage = impulse / hardness
    │  2. 应用 damageThresholdMin / damageThresholdMax 过滤
    │  3. 在 damageRadiusMax 范围内查找受影响的 bond/chunk
    │  4. 选择 shader: shearDamage ? ShearShader : RadialFalloffShader
    │
    │  如果设置了 custom damageFunction (damageFunction != nullptr):
    │  ──► customImpactDamageFunction(data, actor, shape, pos, force)
    │       ──► BlastController::stressDamage()
    │           将力传递给 ExtPxStressSolver 而非直接破坏 bond
    │
    │  如果 damageFunction == nullptr（默认路径）:
    │  ──► damageActor()
    │       构造 shear 或 impactSpread shader
    │       直接调用 TkActor::generateFracture() + applyFracture()
    │       （与 immediateDamage 相同的即时路径）
```

### ExtImpactSettings 参数

```cpp
struct ExtImpactSettings {
    bool  shearDamage;              // 使用剪切伤害 (默认 true)
    float hardness;                 // 材料硬度, damage = impulse / hardness (默认 10.0)
    float damageRadiusMax;          // 全伤害半径 (默认 2.0)
    float damageThresholdMin;       // 最小伤害阈值, 过滤微小碰撞 (默认 0.1)
    float damageThresholdMax;       // 最大伤害阈值, 限制单次伤害 (默认 1.0)
    float damageFalloffRadiusFactor; // 衰减半径因子 (默认 2.0)
    ExtImpactDamageFunction damageFunction; // 自定义 damage 函数 (可选)
};
```

---

## 每帧主循环（BlastController::Animate）

```cpp
void BlastController::Animate(double dt)
{
    // 1. 应用碰撞伤害 (将累积的 impact damage 转为 TkActor damage 请求)
    m_extImpactDamageManager->applyDamage();

    // 2. 更新拖拽应力
    updateDraggingStress();

    // 3. 同步 PhysX
    getPhysXController().simualtionSyncEnd();

    // 4. 更新 impact damage 开关设置
    updateImpactDamage();

    // 5. Family 预分裂更新 (spawn、收集待更新 health 的 actor)
    for (family : m_families)
        family->updatePreSplit(dt);

    // 6. ★ 核心: Tk Group 处理 (执行所有 deferred damage + 分裂)
    m_extGroupTaskManager->process();   // 多线程执行 damage/fracture/split
    m_extGroupTaskManager->wait();      // 等待完成

    // 7. 清理 damage buffer
    m_damageParamsBuffer.clear();
    m_damageDescBuffer.clear();

    // 8. 开始新的 PhysX 模拟
    getPhysXController().simulationBegin(dt);

    // 9. Family 后分裂更新 (更新渲染、stress solver)
    for (family : m_families)
        family->updateAfterSplit(dt);
}
```

---

## NvBlastDamageProgram 与 Shader 体系

### 结构

```cpp
struct NvBlastDamageProgram {
    NvBlastGraphShaderFunction    graphShaderFunction;     // 作用于 support graph 的 bond
    NvBlastSubgraphShaderFunction subgraphShaderFunction;  // 作用于 subchunk 内部
};
```

每个 damage 操作都需要一对 shader：graph shader 处理 bond 间的伤害，subgraph shader 处理 chunk 内部子结构的伤害。任一可为 nullptr 跳过。

### Shader 参数传递

```cpp
struct NvBlastExtProgramParams {
    const void* damageDesc;             // 具体伤害描述 (RadialDamageDesc 等)
    const void* material;               // NvBlastExtMaterial* (health, threshold)
    NvBlastExtDamageAccelerator* accelerator; // AABB 加速结构 (可选)
};
```

Shader 函数通过 `params->damageDesc` 获取具体伤害参数，通过 `params->material` 获取材质属性。

### NvBlastExtMaterial 归一化

```cpp
struct NvBlastExtMaterial {
    float health;               // 基础血量 (默认 100)
    float minDamageThreshold;   // 最小伤害阈值 [0,1] (默认 0.0)
    float maxDamageThreshold;   // 最大伤害阈值 [0,1] (默认 1.0)

    float getNormalizedDamage(float damageInHealth) const {
        float damage = damageInHealth / health;
        return clamp(damage, minDamageThreshold, maxDamageThreshold);
    }
};
```

伤害值在传入 shader 前需要归一化到 [0, 1] 范围，其中 1.0 = 完全摧毁一个 bond。

---

## Stress Damage 路径

Stress damage 不走 shader 体系，而是通过应力求解器间接造成伤害：

```
碰撞/拖拽 产生力
    │
    ▼
ExtPxStressSolver::addForce(actor, position, force)
    │  将力加入应力求解器
    │
    ▼ (每帧)
ExtPxStressSolver::update(stressDamageEnabled)
    │  迭代求解应力分布
    │  如果 stressDamageEnabled:
    │      当应力超过 bond 强度时，自动生成 damage 请求
    │
    ▼
TkActor::damage(...)
    │  进入标准 damage 流程
```

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `samples/SampleBase/ui/DamageToolController.cpp` | 手动伤害工具，注册 7 种 damage 类型 |
| `samples/SampleBase/blast/BlastController.cpp` | damage 入口 (defer/immediate/impact/stress) |
| `samples/SampleBase/blast/BlastController.h` | EventCallback、FixedBuffer、UI 参数 |
| `samples/SampleBase/blast/BlastFamily.cpp` | overlap 查询、stress solver 管理 |
| `sdk/extensions/shaders/include/NvBlastExtDamageShaders.h` | 所有内置 shader 声明 + DamageDesc 结构 |
| `sdk/extensions/physx/include/NvBlastExtImpactDamageManager.h` | 碰撞伤害管理器接口 |
| `sdk/toolkit/include/NvBlastTkActor.h` | `damage()` / `generateFracture()` / `applyFracture()` |
| `sdk/lowlevel/include/NvBlastTypes.h` | `NvBlastDamageProgram` 定义 |
