# FPS Microgame 系统报告

## 工程结构概览
- `Assets/FPS/Scripts` 依据职责拆分为 `AI`, `Game`, `Gameplay`, `UI`, `Editor` 等模块，并通过各自的 `*.asmdef` 形成可编译的 Assembly Definition，方便在 Unity 中做按域引用。
- `Game` 目录提供基础设施（事件、公共组件、管理器）；`Gameplay` 实现玩家武器、移动、拾取与目标；`AI` 负责敌人感知与巡逻；`UI` 负责 HUD/菜单。Prefab 资产（`Assets/FPS/Prefabs`）把这些脚本拼装成 `GameManager`, `Player`, `Enemies`, `Objectives` 等对象并在场景（`Scenes/MainScene.unity` 等）中实例化。

## 关键系统与协作细节

### 事件总线与管理层
游戏通过 `EventManager` 和若干 `GameEvent` 派生类实现松耦合通信，所有系统只需广播/订阅事件即可互动，典型事件包括胜负、玩家死亡、目标更新、拾取等：

```
```8:65:Assets/FPS/Scripts/Game/Events.cs
public static class Events
{
    public static ObjectiveUpdateEvent ObjectiveUpdateEvent = new ObjectiveUpdateEvent();
    public static AllObjectivesCompletedEvent AllObjectivesCompletedEvent = new AllObjectivesCompletedEvent();
    public static GameOverEvent GameOverEvent = new GameOverEvent();
    public static PlayerDeathEvent PlayerDeathEvent = new PlayerDeathEvent();
    public static EnemyKillEvent EnemyKillEvent = new EnemyKillEvent();
    public static PickupEvent PickupEvent = new PickupEvent();
    public static AmmoPickupEvent AmmoPickupEvent = new AmmoPickupEvent();
    public static DamageEvent DamageEvent = new DamageEvent();
    public static DisplayMessageEvent DisplayMessageEvent = new DisplayMessageEvent();
}
```
```

`Assets/FPS/Scripts/Game/Managers` 中的管理类（`GameFlowManager`, `ObjectiveManager`, `ActorsManager`, `AudioManager`, `EventManager`) 均挂在 `GameManager.prefab`，负责在场景级别监听事件并驱动 UI/音频/关卡状态。

### 胜负判断与目标系统
`Objective` 抽象类封装了目标文本、可选性和完成流程；`ObjectiveManager` 负责追踪所有目标并在全部阻塞型目标完成后发出 `AllObjectivesCompletedEvent`，`GameFlowManager` 则监听“全部目标完成”与“玩家死亡”事件来触发胜负结算、声音、淡出和跳转场景。该链路覆盖了题目中的“胜负判断”环节。

```
```20:57:Assets/FPS/Scripts/Game/Shared/Objective.cs
public void CompleteObjective(string descriptionText, string counterText, string notificationText)
{
    IsCompleted = true;
    ObjectiveUpdateEvent evt = Events.ObjectiveUpdateEvent;
    evt.Objective = this;
    evt.DescriptionText = descriptionText;
    evt.CounterText = counterText;
    evt.NotificationText = notificationText;
    evt.IsComplete = IsCompleted;
    EventManager.Broadcast(evt);
    OnObjectiveCompleted?.Invoke(this);
}
```
```

```
```20:35:Assets/FPS/Scripts/Game/Managers/ObjectiveManager.cs
void Update()
{
    if (m_Objectives.Count == 0 || m_ObjectivesCompleted)
        return;
    for (int i = 0; i < m_Objectives.Count; i++)
    {
        if (m_Objectives[i].IsBlocking())
        {
            return;
        }
    }
    m_ObjectivesCompleted = true;
    EventManager.Broadcast(Events.AllObjectivesCompletedEvent);
}
```
```

```
```31:105:Assets/FPS/Scripts/Game/Managers/GameFlowManager.cs
void OnAllObjectivesCompleted(AllObjectivesCompletedEvent evt) => EndGame(true);
void OnPlayerDeath(PlayerDeathEvent evt) => EndGame(false);
void EndGame(bool win)
{
    Cursor.lockState = CursorLockMode.None;
    Cursor.visible = true;
    GameIsEnding = true;
    EndGameFadeCanvasGroup.gameObject.SetActive(true);
    if (win)
    {
        m_SceneToLoad = WinSceneName;
        m_TimeLoadEndGameScene = Time.time + EndSceneLoadDelay + DelayBeforeFadeToBlack;
        var audioSource = gameObject.AddComponent<AudioSource>();
        audioSource.clip = VictorySound;
        audioSource.PlayScheduled(AudioSettings.dspTime + DelayBeforeWinMessage);
        DisplayMessageEvent displayMessage = Events.DisplayMessageEvent;
        displayMessage.Message = WinGameMessage;
        displayMessage.DelayBeforeDisplay = DelayBeforeWinMessage;
        EventManager.Broadcast(displayMessage);
    }
    else
    {
        m_SceneToLoad = LoseSceneName;
        m_TimeLoadEndGameScene = Time.time + EndSceneLoadDelay;
    }
}
```
```

### 武器与射击判断系统
玩家武器栈由 `PlayerWeaponsManager` 管理：在 `Start` 中注入输入组件和初始武器，`Update` 循环根据输入（射击、瞄准、切换）驱动武器，控制摇摆、后坐、瞄准 FOV 等效果。它也是“射击判断”的第一层（是否能射击、是否在换弹/切枪、是否在瞄准）。

```
```124:205:Assets/FPS/Scripts/Gameplay/Managers/PlayerWeaponsManager.cs
WeaponController activeWeapon = GetActiveWeapon();
if (activeWeapon != null && activeWeapon.IsReloading)
    return;
if (activeWeapon != null && m_WeaponSwitchState == WeaponSwitchState.Up)
{
    if (!activeWeapon.AutomaticReload && m_InputHandler.GetReloadButtonDown() && activeWeapon.CurrentAmmoRatio < 1.0f)
    {
        IsAiming = false;
        activeWeapon.StartReloadAnimation();
        return;
    }
    IsAiming = m_InputHandler.GetAimInputHeld();
    bool hasFired = activeWeapon.HandleShootInputs(
        m_InputHandler.GetFireInputDown(),
        m_InputHandler.GetFireInputHeld(),
        m_InputHandler.GetFireInputReleased());
    if (hasFired)
    {
        m_AccumulatedRecoil += Vector3.back * activeWeapon.RecoilForce;
        m_AccumulatedRecoil = Vector3.ClampMagnitude(m_AccumulatedRecoil, MaxRecoilDistance);
    }
}
```
```

单个武器由 `WeaponController` 描述多种射击模式、弹药、后坐与音效。`HandleShootInputs` 会根据模式调用 `HandleShoot`，实例化 `ProjectileBase`，播放特效并记录弹药与冷却，为“武器系统”“射击判断”提供核心逻辑。

```
```333:491:Assets/FPS/Scripts/Game/Shared/WeaponController.cs
public bool HandleShootInputs(bool inputDown, bool inputHeld, bool inputUp)
{
    m_WantsToShoot = inputDown || inputHeld;
    switch (ShootType)
    {
        case WeaponShootType.Manual:
            if (inputDown)
            {
                return TryShoot();
            }
            return false;
        case WeaponShootType.Automatic:
            if (inputHeld)
            {
                return TryShoot();
            }
            return false;
        case WeaponShootType.Charge:
            if (inputHeld)
            {
                TryBeginCharge();
            }
            if (inputUp || (AutomaticReleaseOnCharged && CurrentCharge >= 1f))
            {
                return TryReleaseCharge();
            }
            return false;
        default:
            return false;
    }
}
void HandleShoot()
{
    int bulletsPerShotFinal = ShootType == WeaponShootType.Charge
        ? Mathf.CeilToInt(CurrentCharge * BulletsPerShot)
        : BulletsPerShot;
    for (int i = 0; i < bulletsPerShotFinal; i++)
    {
        Vector3 shotDirection = GetShotDirectionWithinSpread(WeaponMuzzle);
        ProjectileBase newProjectile = Instantiate(ProjectilePrefab, WeaponMuzzle.position,
            Quaternion.LookRotation(shotDirection));
        newProjectile.Shoot(this);
    }
    if (WeaponAnimator)
    {
        WeaponAnimator.SetTrigger(k_AnimAttackParameter);
    }
    OnShoot?.Invoke();
    OnShootProcessed?.Invoke();
}
```
```

### 投射物与伤害流水线
投射物（如 `ProjectileStandard`）在飞行过程中做碰撞判断并在命中时把伤害传入 `Damageable`/`Health`，同时支持范围伤害与特效。伤害值的统一入口是 `Damageable.InflictDamage` 和 `Health.TakeDamage`，从而实现题目所述的“伤害值”链路。

```
```82:195:Assets/FPS/Scripts/Gameplay/ProjectileStandard.cs
new void OnShoot()
{
    m_ShootTime = Time.time;
    m_LastRootPosition = Root.position;
    m_Velocity = transform.forward * Speed;
    m_IgnoredColliders = new List<Collider>();
    transform.position += m_ProjectileBase.InheritedMuzzleVelocity * Time.deltaTime;
    Collider[] ownerColliders = m_ProjectileBase.Owner.GetComponentsInChildren<Collider>();
    m_IgnoredColliders.AddRange(ownerColliders);
}
void OnHit(Vector3 point, Vector3 normal, Collider collider)
{
    if (AreaOfDamage)
    {
        AreaOfDamage.InflictDamageInArea(Damage, point, HittableLayers, k_TriggerInteraction,
            m_ProjectileBase.Owner);
    }
    else
    {
        Damageable damageable = collider.GetComponent<Damageable>();
        if (damageable)
        {
            damageable.InflictDamage(Damage, false, m_ProjectileBase.Owner);
        }
    }
    Destroy(this.gameObject);
}
```
```

```
```25:85:Assets/FPS/Scripts/Game/Shared/Health.cs
public void TakeDamage(float damage, GameObject damageSource)
{
    if (Invincible)
        return;
    float healthBefore = CurrentHealth;
    CurrentHealth -= damage;
    CurrentHealth = Mathf.Clamp(CurrentHealth, 0f, MaxHealth);
    float trueDamageAmount = healthBefore - CurrentHealth;
    if (trueDamageAmount > 0f)
    {
        OnDamaged?.Invoke(trueDamageAmount, damageSource);
    }
    HandleDeath();
}
void HandleDeath()
{
    if (m_IsDead)
        return;
    if (CurrentHealth <= 0f)
    {
        m_IsDead = true;
        OnDie?.Invoke();
    }
}
```
```

```
```25:44:Assets/FPS/Scripts/Game/Shared/Damageable.cs
public void InflictDamage(float damage, bool isExplosionDamage, GameObject damageSource)
{
    if (Health)
    {
        var totalDamage = damage;
        if (!isExplosionDamage)
        {
            totalDamage *= DamageMultiplier;
        }
        if (Health.gameObject == damageSource)
        {
            totalDamage *= SensibilityToSelfdamage;
        }
        Health.TakeDamage(totalDamage, damageSource);
    }
}
```
```

### 拾取礼包系统
所有拾取都继承自 `Pickup`，自带漂浮/旋转动画、触发器和拾取反馈。具体子类（`AmmoPickup`, `WeaponPickup`, `HealthPickup`, `JetpackPickup` 等）在 `OnPicked` 覆写中修改玩家状态并销毁对象，实现题目中的“拾取礼包”部分。

```
```26:86:Assets/FPS/Scripts/Gameplay/Pickup.cs
void OnTriggerEnter(Collider other)
{
    PlayerCharacterController pickingPlayer = other.GetComponent<PlayerCharacterController>();
    if (pickingPlayer != null)
    {
        OnPicked(pickingPlayer);
        PickupEvent evt = Events.PickupEvent;
        evt.Pickup = gameObject;
        EventManager.Broadcast(evt);
    }
}
protected virtual void OnPicked(PlayerCharacterController playerController)
{
    PlayPickupFeedback();
}
public void PlayPickupFeedback()
{
    if (m_HasPlayedFeedback)
        return;
    if (PickupSfx)
    {
        AudioUtility.CreateSFX(PickupSfx, transform.position, AudioUtility.AudioGroups.Pickup, 0f);
    }
    if (PickupVfxPrefab)
    {
        var pickupVfxInstance = Instantiate(PickupVfxPrefab, transform.position, Quaternion.identity);
    }
    m_HasPlayedFeedback = true;
}
```
```

### 敌人检测与 AI
`DetectionModule` 使用射线与距离判定来寻找玩家，并通过回调驱动不同敌人（巡逻、炮塔等）；`EnemyManager` 追踪敌人数量，死亡时广播 `EnemyKillEvent` 供目标系统使用；`Actor` 则提供 AI 识别阵营的基础数据。这些脚本组合实现敌人“发现-攻击-掉落奖励”的流程。

```
```46:140:Assets/FPS/Scripts/AI/DetectionModule.cs
public virtual void HandleTargetDetection(Actor actor, Collider[] selfColliders)
{
    if (KnownDetectedTarget && !IsSeeingTarget && (Time.time - TimeLastSeenTarget) > KnownTargetTimeout)
    {
        KnownDetectedTarget = null;
    }
    float sqrDetectionRange = DetectionRange * DetectionRange;
    IsSeeingTarget = false;
    float closestSqrDistance = Mathf.Infinity;
    foreach (Actor otherActor in m_ActorsManager.Actors)
    {
        if (otherActor.Affiliation != actor.Affiliation)
        {
            float sqrDistance = (otherActor.transform.position - DetectionSourcePoint.position).sqrMagnitude;
            if (sqrDistance < sqrDetectionRange && sqrDistance < closestSqrDistance)
            {
                RaycastHit[] hits = Physics.RaycastAll(DetectionSourcePoint.position,
                    (otherActor.AimPoint.position - DetectionSourcePoint.position).normalized, DetectionRange,
                    -1, QueryTriggerInteraction.Ignore);
                ...
                if (hitActor == otherActor)
                {
                    IsSeeingTarget = true;
                    closestSqrDistance = sqrDistance;
                    TimeLastSeenTarget = Time.time;
                    KnownDetectedTarget = otherActor.AimPoint.gameObject;
                }
            }
        }
    }
    IsTargetInAttackRange = KnownDetectedTarget != null &&
                            Vector3.Distance(transform.position, KnownDetectedTarget.transform.position) <=
                            AttackRange;
}
```
```

### 地图生成 / 关卡装配
项目预置了主场景，但也提供了关卡装配工具：`MeshCombiner` 支持把多个父节点下的静态 Mesh 合并、可配置网格切块，以优化性能；`PrefabReplacer`/`PrefabReplacerOnInstance` 用于把占位预制批量替换为正式模块；配套的 `Level`、`PBObjects` Prefab 集合就是“地图生成”素材来源。

```
```16:65:Assets/FPS/Scripts/Game/MeshCombiner.cs
public void Combine()
{
    List<MeshRenderer> validRenderers = new List<MeshRenderer>();
    foreach (GameObject combineParent in CombineParents)
    {
        validRenderers.AddRange(combineParent.GetComponentsInChildren<MeshRenderer>());
    }
    if (UseGrid)
    {
        for (int i = 0; i < GetGridCellCount(); i++)
        {
            if (GetGridCellBounds(i, out Bounds bounds))
            {
                CombineAllInBounds(bounds, validRenderers);
            }
        }
    }
    else
    {
        MeshCombineUtility.Combine(validRenderers,
            MeshCombineUtility.RendererDisposeMethod.DestroyRendererAndFilter, "Level_Combined");
    }
}
```
```

## 运行流程与入口点
1. **场景入口**：构建流程通常从 `IntroMenu`（菜单）跳转到 `MainScene`，该场景包含 `GameManager`（各类 Manager）、`Player.prefab`、`Objective`/`Enemy` 相关 Prefab。`ProjectSettings/EditorBuildSettings.asset` 定义了这些场景的 Build 顺序（赢/输结算场景也在其中）。
2. **玩家初始化**：`PlayerCharacterController` 在 `Awake/Start` 中注册自己到 `ActorsManager`、抓取输入/武器/健康组件，并监听死亡事件。其 `OnDie` 会广播 `PlayerDeathEvent`，触发 GameFlow 的失败分支。

```
```136:227:Assets/FPS/Scripts/Gameplay/PlayerCharacterController.cs
void Awake()
{
    ActorsManager actorsManager = FindObjectOfType<ActorsManager>();
    if (actorsManager != null)
        actorsManager.SetPlayer(gameObject);
}
void Start()
{
    m_Controller = GetComponent<CharacterController>();
    m_InputHandler = GetComponent<PlayerInputHandler>();
    m_WeaponsManager = GetComponent<PlayerWeaponsManager>();
    m_Health = GetComponent<Health>();
    m_Actor = GetComponent<Actor>();
    m_Health.OnDie += OnDie;
    SetCrouchingState(false, true);
    UpdateCharacterHeight(true);
}
void OnDie()
{
    IsDead = true;
    m_WeaponsManager.SwitchToWeaponIndex(-1, true);
    EventManager.Broadcast(Events.PlayerDeathEvent);
}
```
```

3. **输入管线**：`PlayerInputHandler`（同一 GameObject）锁定鼠标、读取各类轴/按钮，受 `GameFlowManager.GameIsEnding` 控制以防结算时继续接收输入。
4. **战斗循环**：输入 -> `PlayerWeaponsManager` -> `WeaponController` -> `ProjectileStandard` -> `Damageable/Health`；敌人同样通过各自的 `WeaponController`（或近战脚本）触发伤害，`DetectionModule` 提供目标数据。
5. **目标与胜负**：`Objective` 子类（如 `ObjectiveKillEnemies`, `ObjectiveReachPoint`, `ObjectivePickupItem`）监听特定事件或触发器更新进度；当所有阻塞目标完成时，`ObjectiveManager` 唤起 `GameFlowManager` 进入胜利流程。若 `Health` 归零或跌出地图，`PlayerCharacterController` 触发死亡流程。

## 系统协作示意（题目要素对应）
- **胜负判断**：`Objective` → `ObjectiveManager` → `GameFlowManager` → 切场景/播放音效。
- **武器系统**：`PlayerWeaponsManager` 维护武器槽、瞄准、切枪；`WeaponController` 定义射击模式、弹药；`ProjectileBase` 派生体负责飞行与命中反馈。
- **拾取礼包**：`Pickup` 体系在触发时修改 `Health`、`PlayerWeaponsManager` 或武器弹药，广播 `PickupEvent`。
- **射击判断**：`PlayerInputHandler` 采集射击按钮；`PlayerWeaponsManager` 校验换弹/瞄准状态；`WeaponController` 进一步检查冷却、子弹、蓄力；`ProjectileStandard` 执行命中确认。
- **伤害值**：命中后 `Damageable` 把伤害乘以倍率传给 `Health`，触发 `OnDamaged/OnDie`，再由 `PlayerCharacterController` 或敌人控制器广播事件。
- **地图生成**：关卡以 Prefab 模块化拼装，`MeshCombiner`/`PrefabReplacer`/`Level` 目录提供批量替换与合批工具，快速产出地形与掩体，同时保证烘焙 NavMesh 后可供 AI 使用。

## 管理类清单
- `GameFlowManager`, `ObjectiveManager`, `ActorsManager`, `AudioManager`, `EventManager`（全局状态）
- `PlayerWeaponsManager`, `PlayerInputHandler`, `PlayerCharacterController`（玩家本地控制）
- `EnemyManager`, `DetectionModule`, `NavigationModule`（AI 控制）
- `UI` 目录下的 `InGameMenuManager`, `HUDManager`（未展开）负责显示事件结果。

## 建议与观察
- 可以借助 `EventManager.Clear()` 在场景跳转或测试模式下重置监听，避免残留。
- 如果要扩展“地图生成”，建议把现有 `PrefabReplacer` + `MeshCombiner` 流程包装成编辑器工具（`Assets/FPS/Scripts/Editor` 已提供范例），以脚本化生成随机模块。
- 胜负逻辑目前仅依赖 `ObjectiveManager`，若要实现限时或生存模式，可扩充新的 `Objective` 或直接向 `GameFlowManager` 广播 `GameOverEvent`。


HUD 是 Heads-Up Display 的缩写，指游戏里始终浮在屏幕上的界面，用来提示玩家关键状态，比如血量、弹药、任务进度、提示文字等。FPS Microgame 的 Assets/FPS/Scripts/UI 目录下相关脚本（如 HUD 管理器）就负责把这些信息实时显示在画面上。