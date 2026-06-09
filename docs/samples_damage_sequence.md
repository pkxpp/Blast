# Blast Samples AddDamage 模块间 API 调用时序图

## 1. 手动伤害 (Defer Path) — 以 Radial Damage 为例

```mermaid
sequenceDiagram
    participant User as 用户输入
    participant DTC as DamageToolController
    participant BC as BlastController
    participant BF as BlastFamily
    participant PxMgr as ExtPxManager
    participant PxScene as PhysX Scene
    participant PxActor as ExtPxActor
    participant TkActor as TkActor (NV API)
    participant Group as TkGroup (NV API)
    participant Shader as Damage Shader (NV API)

    User->>DTC: 鼠标点击 (WM_LBUTTONDOWN)
    DTC->>PxScene: raycast(eyePos, pickDir)
    PxScene-->>DTC: hit (position, normal, shape)

    DTC->>BC: overlap(PxSphereGeometry, hitPos, damageFunction)
    loop 遍历所有 BlastFamily
        BC->>BF: overlap(geometry, pose, hitCall)
        BF->>PxScene: overlap(geometry, pose, callback)
        PxScene-->>BF: hit ExtPxActor[]
        BF->>DTC: hitCall(pxActor, family)
    end

    Note over DTC: 构造 NvBlastExtRadialDamageDesc<br/>{damage, position, minRadius, maxRadius}

    DTC->>BC: deferDamage(pxActor, family, program, desc, size)
    BC->>BC: m_damageDescBuffer.push(desc)
    BC->>BC: 构造 NvBlastExtProgramParams{desc, material, accelerator}
    BC->>BC: m_damageParamsBuffer.push(programParams)
    BC->>TkActor: damage(program, programParams)
    Note over TkActor: 标记 pending 状态<br/>damage 入队不立即执行

    Note over BC: ═══ 下一帧 Animate ═══

    BC->>Group: process()
    Group->>Group: startProcess() → jobCount
    Group->>Group: acquireWorker()
    loop 每个 pending actor
        Group->>TkActor: worker.process(jobId)
        TkActor->>Shader: graphShaderFunction(commandBuffers, actor, params)
        Shader-->>TkActor: bondFractureCommands
        TkActor->>Shader: subgraphShaderFunction(commandBuffers, actor, params)
        Shader-->>TkActor: chunkFractureCommands
        TkActor->>TkActor: applyFracture(nullptr, commands)
        Note over TkActor: 执行分裂<br/>生成新 actors
    end
    Group->>Group: returnWorker()
    Group->>Group: endProcess()
    Note over Group: 派发事件:<br/>onActorCreated / onActorDestroyed

    Group-->>BC: 完成
    BC->>BC: m_damageParamsBuffer.clear()
    BC->>BC: m_damageDescBuffer.clear()
```

## 2. 手动伤害 (Immediate Path) — Impact Spread

```mermaid
sequenceDiagram
    participant DTC as DamageToolController
    participant BC as BlastController
    participant PxActor as ExtPxActor
    participant TkActor as TkActor (NV API)
    participant Shader as Damage Shader (NV API)

    DTC->>BC: immediateDamage(pxActor, family, program, desc)

    BC->>BC: 构造 NvBlastExtProgramParams{desc, material, accelerator}
    BC->>BC: getFractureBuffers(pxActor) → NvBlastFractureBuffers

    BC->>TkActor: generateFracture(&fractureEvents, program, &programParams)
    TkActor->>Shader: graphShaderFunction(...)
    Shader-->>TkActor: bondFractureCommands
    TkActor->>Shader: subgraphShaderFunction(...)
    Shader-->>TkActor: chunkFractureCommands
    TkActor-->>BC: fractureEvents 填充完成

    BC->>TkActor: applyFracture(nullptr, &fractureEvents)
    Note over TkActor: 立即执行断裂<br/>设置 pending 状态
    Note over TkActor: 不需要等 Group::process()
```

## 3. 碰撞伤害 (Impact Damage Path)

```mermaid
sequenceDiagram
    participant PhysX as PhysX Engine
    participant CB as EventCallback
    participant ImpactMgr as ExtImpactDamageManager (NV API)
    participant BC as BlastController
    participant PxActor as ExtPxActor
    participant TkActor as TkActor (NV API)
    participant Group as TkGroup (NV API)
    participant StressSolver as ExtPxStressSolver (NV API)

    Note over PhysX: 物理模拟产生碰撞

    PhysX->>CB: onContact(pairHeader, pairs, nbPairs)
    CB->>ImpactMgr: onContact(pairHeader, pairs, nbPairs)
    Note over ImpactMgr: 内部累积冲量信息<br/>按 actor 分组存储

    Note over BC: ═══ 同帧 Animate 开始 ═══

    BC->>ImpactMgr: applyDamage()
    ImpactMgr->>ImpactMgr: 计算 damage = impulse / hardness
    ImpactMgr->>ImpactMgr: 过滤: damageThresholdMin ≤ damage ≤ damageThresholdMax

    alt 默认路径 (damageFunction == nullptr)
        ImpactMgr->>ImpactMgr: 选择 shader: shearDamage ? ShearShader : ImpactSpreadShader
        ImpactMgr->>ImpactMgr: ensureBuffersSize()
        ImpactMgr->>TkActor: generateFracture(&fractureBuffers, program, params)
        TkActor->>Shader: graphShaderFunction / subgraphShaderFunction
        Shader-->>TkActor: fractureCommands
        ImpactMgr->>TkActor: applyFracture(nullptr, &fractureBuffers)
        Note over TkActor: 立即执行断裂（同 immediateDamage 路径）
    else 自定义路径 (stressDamage)
        ImpactMgr->>BC: customImpactDamageFunction(data, actor, shape, pos, force)
        BC->>StressSolver: addForce(llActor, position, force)
        Note over StressSolver: 力加入应力求解器
    end
```

## 4. 每帧主循环 (BlastController::Animate) 总览

```mermaid
sequenceDiagram
    participant App as Application
    participant BC as BlastController
    participant ImpactMgr as ExtImpactDamageManager
    participant StressSolver as ExtPxStressSolver
    participant BF as BlastFamily
    participant Group as TkGroup
    participant PxScene as PhysX Scene
    participant ExtPxMgr as ExtPxManager

    App->>BC: Animate(dt)

    rect rgb(255, 240, 230)
        Note over BC: ① 碰撞伤害处理
        BC->>ImpactMgr: applyDamage()
        ImpactMgr-->>BC: damage 请求入队到 TkActors
    end

    rect rgb(230, 240, 255)
        Note over BC: ② 拖拽应力更新
        BC->>StressSolver: addForce(dragVector)
    end

    rect rgb(230, 255, 230)
        Note over BC: ③ PhysX 同步
        BC->>PxScene: simualtionSyncEnd()
    end

    rect rgb(255, 255, 230)
        Note over BC: ④ Family 预分裂
        loop 每个 BlastFamily
            BC->>BF: updatePreSplit(dt)
            BF->>ExtPxMgr: spawn (首次)
            BF->>BF: 收集 pending actors
        end
    end

    rect rgb(255, 230, 230)
        Note over BC: ⑤ ★ 核心: Group 处理
        BC->>Group: process()
        Note over Group: 执行所有 deferred damage<br/>调用 shader → fracture → split<br/>派发 actor 创建/销毁事件
        Group-->>BC: 完成
        BC->>BC: damageBuffer.clear()
    end

    rect rgb(230, 255, 255)
        Note over BC: ⑥ 开始物理模拟
        BC->>PxScene: simulationBegin(dt)
    end

    rect rgb(240, 230, 255)
        Note over BC: ⑦ Family 后分裂
        loop 每个 BlastFamily
            BC->>BF: updateAfterSplit(dt)
            BF->>BF: 更新渲染 transform
            BF->>StressSolver: update(stressDamageEnabled)
            BF->>BF: 更新 bond health 可视化
        end
    end
```

## 5. 模块层次关系

```
┌─────────────────────────────────────────────────────┐
│                    用户层 (Application)                │
│  Input → DamageToolController / PhysXController      │
├─────────────────────────────────────────────────────┤
│               Sample 模块层 (BlastController)         │
│  BlastController  ← 管理 damage 入口、buffer、生命周期  │
│  BlastFamily      ← 管理渲染、stress solver、overlap   │
│  BlastAsset       ← 管理 asset + DamageAccelerator    │
├─────────────────────────────────────────────────────┤
│            NV Blast Toolkit API (C++)                 │
│  TkActor::damage()           — deferred damage 入队   │
│  TkActor::generateFracture() — 立即计算 fracture       │
│  TkActor::applyFracture()    — 立即应用 fracture       │
│  TkGroup::process()          — 批量处理所有 damage     │
│  ExtImpactDamageManager      — 碰撞伤害自动管理        │
│  ExtPxManager                — PhysX↔Blast 桥接       │
│  ExtPxStressSolver           — 应力求解                │
├─────────────────────────────────────────────────────┤
│            NV Blast Low-Level API (C)                 │
│  NvBlastActorGenerateFracture()  — shader 执行        │
│  NvBlastActorApplyFracture()     — bond/chunk 断裂     │
│  NvBlastActorSplit()             — island 分裂         │
├─────────────────────────────────────────────────────┤
│                  PhysX SDK                            │
│  PxScene::overlap() / raycast()  — 空间查询            │
│  PxSimulationEventCallback       — 碰撞回调            │
│  PxRigidDynamic                  — 刚体模拟            │
└─────────────────────────────────────────────────────┘
```
