# W04 讲义：生命周期全图与 fake null（2026-11-23）

> 方法 v2：本讲义先行，任务卡只考讲过的。本关主靶：笔试 B1（顺序排错两处）、A4（fake null 答反）、B2（LateUpdate 抖动）三道错题的完整翻案。

## 第一节 · 一帧的完整解剖

```text
场景加载时（每对象一次）：
  Awake → OnEnable →（同帧稍后，所有 Awake/OnEnable 完成后）→ Start

之后每一帧：
  ① FixedUpdate ×N     ← 物理步进（N 可以是 0、1、2、3…）
  ② Update（所有脚本）
  ③ 协程恢复点（yield null 的 MoveNext 在这）
  ④ LateUpdate（所有脚本）
  ⑤ 渲染（相机、Culling、绘制）

销毁/失活：
  OnDisable → OnDestroy（Destroy 在帧末生效）
```

**Awake vs OnEnable vs Start 的分工**：Awake=本对象初始化（不管别人）；OnEnable=每次激活都跑（订阅/注册放这）；Start=首帧 Update 前、所有 Awake 都已完成（**要安全引用别的对象，放 Start 不放 Awake**——Awake 顺序不保证）。

**脚本间顺序**：同回调内顺序不可靠（可编辑 Script Execution Order）。你框架的 `DIBootstrap [DefaultExecutionOrder(-32000)]` 就是在抢"最早 Awake"——C 任务会问为什么。

## 第二节 · FixedUpdate 的真相：累积器

FixedUpdate **不由帧驱动，由时间累积驱动**：

```text
每帧：accumulator += Time.deltaTime（这一帧实际耗时）
      while (accumulator >= Time.fixedDeltaTime)
      { 跑一次 FixedUpdate; accumulator -= Time.fixedDeltaTime; }
```

推论（任务卡 B1 要亲眼看到）：
- 帧很快（deltaTime < 0.02s 默认）→ 该帧 **0 次** FixedUpdate
- 帧卡了（deltaTime = 0.5s）→ 一帧连跑 **25 次**（追账）
- `Time.fixedDeltaTime` 调小 = 物理更精细也更贵；调大反之
- **物理相关逻辑（Rigidbody 操作、力）必须进 FixedUpdate**；输入采样放 Update（FixedUpdate 一帧可能 0 次，会丢输入）

## 第三节 · fake null：Destroy 后的薛定谔对象

**笔试 A4 你答反的那道，机制全解**：

```csharp
Destroy(go);
Debug.Log(go == null);   // true！——但托管对象还在堆上躺着
```

为什么"对象还在却等于 null"？因为 UnityEngine.Object **重载了 == 运算符**：它比较的不是 C# 引用，而是内部的 **native 指针（m_CachedPtr）**。Destroy 销毁的是 C++ 侧的 native 对象，同时把托管壳的指针置空——`==` 一看指针空了，报 null。托管壳要等 GC 才真正消失。

**三个衍生陷阱**：

```csharp
// ① ?. 不能重载 → 走真实引用判空 → fake null 对象引用非空 → 通过检查 → 抛 MissingReferenceException
go?.name;                 // 💥 MRE

// ② ?? 同理（真实引用比较）
var x = go ?? fallback;   // go"非空"→ 不走 fallback → 后面用 x 照样炸

// ③ is/as 也走真实引用
if (go is Object o) ...   // fake null 会通过
```

**Destroy vs DestroyImmediate**：Destroy=帧末统一销毁（运行时唯一合法）；DestroyImmediate=立刻销毁（**只在 Editor 代码里用**，运行时用会破坏引擎内部遍历——你的 CNCore 里被注释的僵尸同步链就混过这种调用）。

**实用守则**：判 Unity 对象死活只用 `== null` / `!= null`（或 `bool` 隐式转换，同款重载）；`?.`/`??`/`is` 留给纯 C# 对象。这也是为什么 W2 的僵尸方法里"碰 Unity 成员报 MRE"——托管侧活着、native 侧死了。

## 第四节 · Transform：两个坐标系

```csharp
transform.position        // 世界坐标（相对场景根）——渲染、距离判断用
transform.localPosition   // 相对父物体——UI 摆位、子物体编排用
```

- **父动子动**：父的 position/rotation/scale 变 → 子的 **position（世界）跟着变**，但子的 **localPosition 不变**（它相对父没动）
- 父有旋转时：子的 localPosition=(1,0,0) 世界位置会被父的旋转带着转——**local 是"父的视角"，world 是"世界的视角"**
- `SetParent(parent, worldPositionStays: true/false)`：true=调 local 使世界位置不变（默认）；false=local 原样照搬（世界位置会跳）
- scale 链：`lossyScale` 是世界缩放（父×子），只读——想改要分解到各级

**笔试 B2 正解（相机跟随抖动）**：角色移动和相机跟随都写 Update 时，两脚本的 Update 顺序不定——相机可能拿到"移动前"的旧位置 → 抖。LateUpdate 保证所有 Update 完再跑，相机永远取当帧最新位置。**LateUpdate 解决不了的**：物理驱动的移动（刚体插值问题，要 Rigidbody interpolation）、动画更新时机。

## 第五节 · 实战预告

- **SNNU UIGlowEffect**：类名与文件名不一致 → Unity 按**文件名**找 MonoBehaviour 类 → 组件挂不上/引用断。修法：文件名改成类名（或反之），全项目搜索引用。F2 重命名让 Unity 自动改 meta 的 GUID 关联。
- **W3 尾巴（开卷费）**：见任务卡 B0——含两个"C2 像素疑点"自查：打开你 RSHE 的 Timer.cs 真文件，亲眼确认 `Cancel` 的守卫是不是 `!handle.IsValid`（没有 `!` 取消永远无效）、`Update` 里是不是 `if (!_paused) return;`（写反了计时器只在暂停时走）。**读自己的代码回答，一测便知**——这就是抽查答辩的形态。
