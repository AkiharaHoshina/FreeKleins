# 05 · 客户端 Lua 覆盖清单

## 5.1 覆盖是怎么生效的

游戏用 `LuaResLoader`（tolua 的 `LuaFileUtils` 子类）读 Lua，
它走 `string.Format("{0}/{1}", baseDir, fileName)` **从资源树读文件，不查进程工作目录**。

所以：**任何 Lua 覆盖必须写进 `D:\KleinsPatch\Win\lua\<相对路径>`**，
`require("logic/xxx/yyy")` 才会命中覆盖版。

> 环境的完整性校验（`CheckEnvironment`）已经被 `GameAssembly.dll` 的补丁阉割掉了，
> 所以改资源树里的 Lua **不会再被自动重置**。

## 5.2 覆盖的两种写法

所有注入脚本都遵循同一套模式，**可安全反复应用 / 还原**：

```
logic/xxx/yyy.lua        ← 覆盖版（包装层：require 原件 → 包一层 → return）
logic/xxx/yyy_orig.lua   ← 原模块的副本（供包装层 require）
_re\_lua_yyy_pristine.orig  ← 原始文件的备份（--revert 时用它还原）
```

每个 `inject_*.py` 都支持 `--revert`，还原后与原始文件**逐字节相同**。

---

## 5.3 当前生效的 13 个覆盖

| # | 覆盖文件 | 大小 | 由谁注入 | 干什么 |
|---|---|---|---|---|
| 1 | `logic/login_ui/logic_login_main.lua` | 1355 | `inject2.py` | **修登录卡死** |
| 2 | `logic/login_ui/view_game_main_ui.lua` | 667 | `inject_btns.py` | **禁用「退出登录」/「修复文件」**（用户要求） |
| 3 | `logic/module_unlock/logic_module_unlock.lua` | 336 | `inject_unlock.py` | **全模块解锁** |
| 4 | `logic/module_unlock/logic_func_unlock.lua` | 332 | `inject_unlock.py` | **全功能解锁** |
| 5 | `logic/home_cultivate/home_ui_service.lua` | 615 | `inject_home.py` | 修「大厅 3D 不加载」 |
| 6 | `logic/role_post/logic_change_scene.lua` | 442 | `inject_scene.py` | 修回大厅时 `sceneId = 0` |
| 7 | `logic/scene_state/home_scene.lua` | 1107 | — | ⚠️ **无效残留**（与 `_orig` 哈希完全相同） |
| 8 | `logic/role_queue/view_role_queue_base.lua` | 3447 | `inject_queue.py` | 编队界面崩溃兜底 |
| 9 | `logic/select_role_ui/select_role/view_role_select_base.lua` | 3465 | `inject_queue.py` | 选人界面崩溃兜底 |
| 10 | `logic/recruit_pay_ui/view_recruit_pay_ui.lua` | 4690 | `inject_recruit.py` | 抽卡退出 + 重置角色按钮 |
| 11 | `logic/role_ui/role_skin_ui/logic_role_skin.lua` | 1413 | `inject_skin.py` | 换装：置标记 + 请求角色列表 |
| 12 | `logic/role_ui/role_skin_ui/view_role_skin_base.lua` | 880 | `inject_skin.py` | 换装：记录 roleId / 皮肤 |
| 13 | `logic/role_info_ui/logic_role.lua` | 1303 | `inject_skin.py` | 换装：角色列表回包后重刷界面 |

---

### ① 登录卡死 —— `inject2.py`

原模块 `InitServerList`（proto#24）末尾：

```
UGET R3 uv1              ; uv1 本应是模块表(父函数 local0)，运行时却是 nil
TGETS R3 R3 'SetServerInfo'
CALLM ...
```

→ `attempt to index a nil value`，登录流程直接卡死。
而 `GetServerList` / `Init` 都是**模块表字段调用**，所以只要把模块表上的 `InitServerList`
换成安全版本，整条链就会走到我们的实现。

> 注：`inject.py` 是同一问题的**早期方案**（往 `InitServerList` 末尾追加），
> 现行方案是 `inject2.py`（整模块包装，放在资源树里）。

### ② 禁用两个按钮 —— `inject_btns.py`

把 `view_game_main_ui.ExitLoginBtnClick` / `FixBtnClick` 替换成空实现，
只弹一个"此功能已禁用"提示。**用户明确要求的。**

### ③④ 全解锁 —— `inject_unlock.py`

把 `logic_module_unlock` / `logic_func_unlock` 的 `defaultUnLockAll` 置真。

### ⑤ 大厅 3D 不加载 —— `inject_home.py`

`home_ui_service.CreateFuncUI` 因为 `logic_chapter.GetCurStageData()` 返回空而挂掉。

### ⑥ 回大厅黑屏 —— `inject_scene.py`

`scene_mgr.GoHome` 里 `sceneId = logic_change_scene.GetCurSceneConfig()`，
未设置时返回 0 → 场景切不过去。

### ⑦ ⚠️ 无效残留

`logic/scene_state/home_scene.lua` 与 `home_scene_orig.lua` **MD5 完全相同**
（`C47F396943D5C592E346CFB07DB17FC4`），即当前写的是原内容，**没有任何实际作用**。
留着无害，清理时直接删掉这两个文件即可。

### ⑧⑨ 编队 / 选人界面崩溃兜底 —— `inject_queue.py`

```
view_role_queue_base.UpdateRoleItem          "attempt to index a nil value"
    ← UpdateQueueInfo ← view_queue_normal_ui.Open ← NormalPreBattleQueue
view_role_select_base.InitRankByRoleData / UpdateRoleItem   同类崩溃
```

做法：给这几处渲染函数套 `pcall`：

- 第一次失败 → 把该格的 `Data(DataIndex/isLock/hasRole/roleId/roleData)` 记到
  `_re\queue_spy.log`，然后把这一格降级成"空位"（`hasRole=false, roleData=nil`）重画一次
- 保证界面不崩，同时日志能告诉我们到底哪个角色的数据缺了

另外还会**补数据**（`fixRoleData`）：`Star` 从 `dataconfig_role_config` 补、
`TalentList` 从 `dataconfig_role_talen_rank` 补，
并修正 `UpdateRoleItem` 里的 `self.index`。

> **根因其实在服务端**：`CSRoleData.TalentList[].Step` 给了 0（见 [04 §4.3](04_服务端实现_存档与配置.md)）。
> 服务端修好后，这个兜底就只是保险了。

### ⑩ 抽卡界面 —— `inject_recruit.py`

**问题 1：抽完卡出不去（卡在翻牌界面）**
`view_multi_recruit_ui.CloseCallBack` 会重新 `ShowView` 抽卡界面 → `RecruitUpdate` → `ChangePool`，
而 `ChangePool` 里

```lua
getConfigDataByID('dataconfig_recruit_pools','pool_id',list[i].id).pool_type
```

只要有一项取不到配置就 `attempt to index a nil value`，**整个流程中断**。
→ 给 `ChangePool` 套 `pcall`，出错就记日志并返回，让流程继续。

**问题 2：卡池列表为空时 `ChangePool(nil)`**
重新打开抽卡界面时数据还没回来 → `ChangePool(nil)` 必崩。
→ 检测到列表为空就**重新请求 `1168`**。

> 这里有个坑：包装层在加载时缓存 `logic_recruit_pay` 会拿到 `nil`（它加载更晚），
> 导致提前 return 把点击卡池吞掉（表现："点任意卡池无效，只有提示音"）。
> 修法是**懒取**：`local function LRP() return rawget(_G, "logic_recruit_pay") end`。

**问题 3：加一个「重置角色数据」按钮**
把「抽卡记录」按钮改成「重置角色数据」（我们的抽卡记录本来就是空的）：

- `M.ClickHistoryBtn` 重新指向 `doReset()`
- `doReset()` 走 `1127` 命令，服务端把角色表重置回 1001..1005
- 按钮**文字**也改掉（`抽卡记录` → `重置角色数据`）

按钮的真实路径（UI 树 dump 出来的，不是猜的）：

```
PaintParent/cardpool_novice(Clone)/Information/InfoGroup/HistoryBtn
```

注意 `cardpool_novice(Clone)` 这段是**每个卡池克隆出来后才存在**的，
所以改文字要在 `Init`、`OpenUI` **以及每次 `ChangePool` 之后**都跑一遍
（用一个按名字递归查找的 `relabel(self)`）。

### ⑪⑫⑬ 换装"点一次就生效" —— `inject_skin.py`（**改三个文件**）

**问题**：点一次「更换着装」不生效，要再点一次。

**根因**（见 [07 §7.17](07_踩坑与结论.md)）：按钮状态由
`logic_role_skin.GetSkinState(roleId, 正在查看的皮肤)` 决定，它比的是
`logic_role_data` 里角色的 `RoleSkin`（来自 1010 角色列表）与"正在查看的皮肤"。
而 `SetRoleSkinRes` 回来时界面会**立刻**刷一次，那一刻角色数据还是旧的
（新皮肤要另发一次 `CSGetRoleList` 才拿得到，是异步的）→ 算出 `HaveAndNotUse`；
等列表回包时**已经没人再刷了** → 第二次点击时数据才是新的。

| 文件 | 改动 |
|---|---|
| `logic_role_skin.lua` ⑪ | `SetRoleSkinRes` 成功后置 `__kleins_skin_pending` + 请求角色列表；`SetRoleSkinReq` 顺带保留"用正在查看的皮肤纠正参数" |
| `view_role_skin_base.lua` ⑫ | 包 `UpdateSkinState(roleId, skinId)`，记住 `__kleins_skin_role` / `__kleins_want_skin` |
| `logic_role.lua` ⑬ | 包 `GetRoleListRes`，**回包之后**重调 `view.UpdateSkinState(roleId, skinId)` |

**踩过的弯**（详见 [07 §7.17](07_踩坑与结论.md)）：
`UpdateSkinState` 的调用形状是 `(roleId, skinId)`，**没有 self** —— 早期版本把第一个参数
当成界面实例存下来，导致 `attempt to index upvalue '' (a number value)`；
事件总线补发 `ROLE_SKIN_UPDATE` 实测**送不到界面**（`ok=true` 但事后没有任何 `UpdateSkinState`），
所以最后改成直接调 `view.UpdateSkinState`。

### ⑭ 卡池查表兜底 —— `inject_recruit.py`（**2026-10-14 加，修抽卡海报空白**）

**现象**：抽完卡（或切换卡池）后，角色展示区（海报）有概率变空；**有的卡池正常、有的不行**。

**根因（反汇编实证）**：`view_recruit_pay_ui` 的 **proto#6**（卡池按钮的 BtnInit / 点击回调注册）里：

```
0094  KSTR  R7  'dataconfig_recruit_ui_config'
0095  KSTR  R8  'pool_id'
0098  TGETS R9 R9 'id'          ; list[i].id
0099  CALL  R5 #2 #4            ; getConfigDataByID(...)
0100  TGETS R5 R5 'btn_icon'    ; ★ 无条件索引
...
0105  KSTR  R8  'dataconfig_recruit_pools'
0115  TGETS R10 R6 'pool_type'  ; ★ 同样无保护
```

两张本地表**任一**缺这个 `pool_id` 的行 → `attempt to index a nil value` →
整个 `ChangePool` 流程中断 → 海报不加载。**"有的卡池正常、有的不行"正是
`dataconfig_recruit_ui_config` 覆盖不全**（那张表在客户端本地，不在服务端 dump 的 58 张里，
服务端补不出来）。

**修法**：给 `config_data_center.getConfigDataByID` 加一层**只在**这两个表名上生效的兜底：
查不到就返回空壳 `{ pool_id = v }`，让流程走完。

### ⑮ 子商城 Open 保护 / 诊断 —— `inject_gift.py`（**新增**）

`dataconfig_shop_banner` 的 11 条 banner，`goto_func` 只有两类目标：

| banner id | `goto_func` | 目标 |
|---|---|---|
| 1（月卡） | `{3,1,706}` | shop_type=3 礼包商城，label 1，商品 706 |
| 2,3,4,5,7,9,10,11 | `{3,1}` | shop_type=3 礼包商城 |
| 6,8 | `{5,1}` | shop_type=5 异色商城 |

> ★ **"今日声讯"里的两个板块就是这些 banner** —— `view_shop_today_ui` 里
> `RightShopJinRi/TypeBar`（标签页）+ `RightShopJinRi/Banner/ScrollView`（banner 列表），
> 点击走 `BannerClick → GetGoToFunc(goto_func) → OpenSubShop`。
> 所以**今日声讯两个板块的跳转 == 礼包 banner 的跳转 == 同一个 bug**。

**反汇编实证**：`view_shop_gift_ui` 的 **`Open` = proto#2**（`FNEW proto#2` 紧跟 `TSETS 'Open'`），开头：

```
0008  TGETS R4 R4 'LabelList'
0009  UGET  R5 uv2            ; uv2 = 1
0010  TGETV R4 R4 R5          ; LabelList[1]
0011  TGETS R4 R4 'ItemList'  ; ★ 无 nil 保护
```

`LabelList` 是按**固定下标 1** 取的（同文件 proto#5 0008 / proto#10 0002 用 `TGETB R3 R3 #1`）。

**修法**：给三个子商城（`view_shop_gift_ui` / `view_shop_heterochromatic_ui` / `view_shop_today_ui`）的
`Open` 加：① 入口打日志（`shopType` / `LabelList` 类型条数 / `[1]` 的值）
② `LabelList` 是 table 且 `[1]` 为 nil 时补一个（优先取第一个可用标签）
③ 原 `Open` 套 `pcall`，出错写 `_re\gift_spy.log` —— 拿真实报错而不是让游戏崩。

> 服务端侧已确认**不是**缺数据：`type=3 → label1:x74`，`type=5 → label1:x2`。

---

## 5.4 一次还原全部

```powershell
cd D:\KleinsPatch\_re
python inject2.py --revert
python inject_btns.py --revert
python inject_unlock.py --revert
python inject_home.py --revert
python inject_scene.py --revert
python inject_queue.py --revert
python inject_recruit.py --revert
python inject_skin.py --revert
python inject_shop.py --revert
python inject_gift.py --revert
python inject_cfgdump.py --revert
python inject_queue_slot.py --revert      # 槽位制实验（已废弃，确保还原）
```

## 5.5 其他注入脚本

| 脚本 | 目标 | 状态 |
|---|---|---|
| `inject.py` | `logic_login_main` 旧方案 | 被 `inject2.py` 取代 |
| `inject_guide.py` | `logic/guide/guide_mgr.lua` | 未应用（跳过引导改由服务端 `1198` 实现） |
| `inject_chapter.py` | `logic_chapter` 锁判定 | **已还原**（见 [07](07_踩坑与结论.md)，那是个错误方向） |
| `inject_queue_slot.py` | 编队槽位制 | **已还原**，与原始备份逐字节相同 |
| `inject_queue_data.py` | `logic_queue_data` | 实验脚本 |
| `inject_activity.py` / `inject_notice.py` / `inject_dlg.py` | 活动/公告/对话框 | 未应用 |
| `inject_select_spy.py` | 选人入口探针 | 诊断用，用完还原 |

### 一次性工具（当前**已应用**，用完记得还原）

| 脚本 | 目标 | 说明 |
|---|---|---|
| `inject_cfgdump.py` | `logic/common/config_data_center.lua` | 包装 `getConfigData`，首次访问配置时把 `PROACTIVE` 清单里的表 dump 成 `cfg\<name>.txt`。**2026-10-14 追加** `dataconfig_recruit_ui_config` 等表名后重新应用 |
| `inject_shop.py` | 商店三件（见 [03 §商店](03_业务层_10301_CSPKG.md)） | 绕过支付 SDK + 月卡弹窗诊断（`_re\shop_spy.log`） |
| `inject_gift.py` | 三个子商城的 `Open` | LabelList 兜底 + `_re\gift_spy.log` |
