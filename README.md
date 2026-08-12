# 数据说明

> 这里说明怎么打开、检索和看懂本项目导出的 JSON。  
> 数据来自对应版本的游戏内存，不是 Arrowhead 的官方源码，也不能单独当成机制结论。  
> **注：可能有少量泄露内容。**

## 项目简介

[HelldiversData](https://github.com/shalzuth/HelldiversData) 原先可以依据游戏自带的枚举名和组件配置表批量导出数值。官方移除这些元数据后，该项目停止更新。

本项目改为直接读取运行中的游戏内存。从 `v1.007.000`（2026-08-12）开始，结构体大小、字段偏移、数据类型、数组长度和嵌套关系都以游戏的 typelib 为准。字段原名仍然可能缺失，因此 JSON 中会保留 `unk`，或使用 4 字节类型哈希作为键名。

`2026-07-07`（`v1.006.301`）及更早的数据由手工测量得到。其中存在切分错误的结构、被误记为 padding 的有效字段，以及仅凭全零数据推断的数组。查阅数值和对照字段时，应使用 2026-08-12 及以后的数据。

目前可以查：

- 伤害、爆炸、射弹、光束、电弧、状态效果、战备等公共配置；
- 武器配件、弹匣、瞄具、涂装等自定义条目；
- 生命、状态接收、武器、装填等组件；
- 同一份数据在不同版本之间的变化。

限制：

1. 这是配置，不是结算结果。实战还受机制、硬编码、服务器和其他组件影响。
2. 字段名不是功能说明。`unk` 表示布局已知、用途未知；`0x` 后接 8 位十六进制的键名，是缺失原名的结构或枚举，保留了它的 4 字节类型哈希。
3. 版本必须对应。游戏更新后，字段位置、枚举编号和数值都可能变化。引用时注明 `game_version` 和 `patch_date`。

## Typelib

typelib 是游戏 DataLibrary 的类型库，文件头是 `LTLD`。[filediver](https://github.com/xypwn/filediver) 可以把它从游戏资源里解出来。

它写明了每个结构在 64 位进程里的总大小、对齐、成员偏移和存储类型，也写明了成员是普通值、定长内嵌数组、运行时数组、位域还是嵌套结构。官方大约在 2024 年 11 月删掉了其中的 `typeinfo_strings`，字段名、注释和一部分枚举名不在文件里，布局还在。

据此可以确定一个 4 字节区域是整数、浮点、枚举还是对齐空隙，也可以确定一大段数据由哪些嵌套结构组成，而不会因为整段数值为零就将其视为无意义数据。游戏更新后，可以先比较新 typelib 中哪些结构变长、哪些成员位移，再修改配置。测试用于确认字段用途，不再用于推断结构边界。

当前数据中的 `unk` 表示位置和类型已经确定，原名或用途尚未确认。2026-07-07 数据中的 `unk` 经常连字节边界也未确定，二者含义不同。

### 类型对应

| typelib 类型 | 项目中的读法 | JSON 中的样子 |
| --- | --- | --- |
| `uint8` / `uint16` / `uint32` | `B` / `H` / `I` | 非负整数 |
| `int8` / `int16` / `int32` | `b` / `h` / `i` | 整数，可以是负数 |
| `uint64` / `int64` | `Q` / `q`，8 字节 | 十进制字符串，避免大整数丢精度 |
| `fp32` / `fp64` | `f` / `d` | 小数 |
| `enum_uint32`、`enum_int32` 等 | 对应宽度的整数，再附上枚举名 | `1 <=> HitReactEventType_Light` |
| `bitfield` | 某个整数里的一个二进制位 | `*_detail` 里的 `0` 或 `1` |
| `struct` | `type: struct` | `{ ... }` |
| `inline_array` | `type: array`，`storage: inline` | 长度固定的数组 |
| `array` | 指针数组，`storage: pointer` | 长度不固定的数组 |
| `ptr` | 8 字节地址，或继续读取它指向的数据 | 地址字符串，或解析后的对象 |
| `str` | `type: string` | 文本 |
| 没有成员的空隙 | `padding` | 不导出 |

`array` 和 `inline_array` 只是内存位置不同。导出后通常都是普通的 `[ ... ]`。

### 键名或枚举名是 4 字节哈希

JSON 里的 `0x` 有两种：

- 很长的那串是 64 位资源哈希，例如实体键 `0xAA28CAF964D05500`，用来标识某一把武器、某一只敌人或某一个模型。
- 8 位十六进制是 32 位类型哈希，例如 `0xD6AB4E79`，用来标识某一种结构体或枚举。

类型哈希由类型原名计算。类型名称不变，哈希也不变。typelib 仍提供该类型的大小和内部字段；若原名已被删除且名称表中也没有记录，导出时就使用这个哈希，不再另行编造英文名称。

键名为哈希时，表示该嵌套结构的布局已知，原名未知。`HitReactComponentData` 中的例子：

```json
"0xD6AB4E79": [
    {
        "0xD34E38FB": [
            {
                "event_type": "1 <=> HitReactEventType_Light",
                "force_strength_threshold": 15,
                "0x4A423019": [
```

`0xD6AB4E79`、`0xD34E38FB`、`0x4A423019` 都是没有原名的嵌套结构。同一记录中的 `event_type` 可以写成 `HitReactEventType_Light`，因为该枚举名称已经匹配。

枚举名为哈希时，表示数值已经读出，枚举类型的原名缺失。`ProjectileWeaponComponentData` 中的例子：

```json
"aim_zeroing_quality": "3 <=> ProjectileZeroingQuality_High",
"unk_silence_type": "2 <=> 0xC60F6AF4_Unknown_2"
```

`aim_zeroing_quality` 的枚举类型和枚举项都有名称。`unk_silence_type` 的值是 `2`，typelib 只提供枚举类型哈希 `0xC60F6AF4`，因此写作 `0xC60F6AF4_Unknown_2`。它不是实体哈希，也不能依据相邻的已知枚举推断其含义。

## 获取数据

每次转储有两种入口：

- **Release**：直接下载阅读。
- **Commit**：看这次改了什么。

看某一版，下载对应 Release。对比变化，比较相邻 Commit 里的同名 JSON。

文件名上的日期不够。Component 的 `_metadata` 在文件根上，Settings 的 `_metadata` 在每张表里面：

```json
{
  "_metadata": {
    "patch_date": "2026-08-12",
    "game_version": "1.007.000"
  }
}
```

以这里的 `patch_date` 和 `game_version` 为准。

## 打开 JSON

JSON 是文本，不需要专用游戏工具。较大的文件可能有几十万行，应使用能够处理大文件的编辑器。

- **VS Code**：搜索、折叠、大文件都够用。
- **Notepad++**：搜索和复制片段。
- 网页 JSON 工具只适合小文件。不要上传未公开数据或包含个人信息的文件。大文件在网页中可能卡顿或截断，应改用 VS Code。

常用操作：

1. `Ctrl + F` 搜武器、敌人或字段名。
2. 只展开正在看的那一层。
3. 搜 `0xAA28CAF964D05500` 这种十六进制哈希，可以直接定位实体。
4. 看一个数时，连同它的父对象、所在数组和相邻字段一起看。
5. 大文件先搜 `name`、`name_zh`、`debug_name`、`type`、`hash_hex`。

## 一条实体记录

```json
{
  "_metadata": { "...": "..." },
  "entities": [
    {
      "0xAA28CAF964D05500": {
        "hash": "12261273158003217664",
        "name": "Spore Spewer",
        "name_zh": "孢子喷涌虫",
        "index": 0,
        "health": 2500,
        "unit_size": "3 <=> UnitSize_Massive"
      }
    }
  ]
}
```

- 外层 `0x...` 是这个实体的十六进制资源哈希。
- `hash` 是同一个数值的十进制写法。大整数保存为字符串，以避免精度丢失。
- `name` / `name_zh` 为 `N/A`，只说明名称表里还没有这条，不说明实体无效。
- `index` 是这张哈希表里的槽位，一般不是游戏里的编号。
- 其余字段才是这个组件的配置。

## 资源哈希

游戏用资源路径计算一个 64 位编号，再用这个编号引用武器、敌人、模型、配件和音效。例如：

```text
content/fac_helldivers/equipment/throwables/caltrops_grenade/caltrops_mine
```

算法是标准 MurmurHash64A，`seed = 0`，路径末尾不加空字符。

发布包里的 `Hash.csv` 是手工维护的对照表，列为 `row_type,category,hash,name,name_zh`。只有 `row_type=entry` 的行是名称：

```text
entry,主武器-突击步枪,10845250369047350884,AR-23 Liberator (Model),AR-23 解放者（主武器，模型）
```

`category` 是人工分类，不参与匹配。同一个哈希也可能写成十六进制，例如 `10845250369047350884 = 0x968211C0033DCE64`（示意）。

检索时注意：

- 同一对象可能分别拥有本体、模型、支架（空投仓）、射弹、占位符等多个哈希。名称相近并不表示它们是同一个对象。
- `payload` 等字段通常只保存另一个资源的哈希，需要再在 `Hash.csv` 中检索。
- 名称表并不完整，出现 `N/A` 是正常情况。不应依据相邻条目推断含义。
- 已收录的名称也可能过时或有误，仍需结合数据位置和游戏内的实际表现判断。

## Settings

发布包路径：`data/settings/`。

Settings 是多处共用的配置表，不是某一只敌人或某一把枪的实例。当前有：

- `generated_damage_settings.json`：伤害；
- `generated_explosion_settings.json`：爆炸；
- `generated_projectile_settings.json`：射弹；
- `generated_beam_settings.json`、`generated_arc_settings.json`：光束、电弧；
- `generated_status_effect_settings.json`：状态效果和模板；
- `generated_stratagem_settings.json`：战备；
- `generated_weapon_customization_settings.json`：武器自定义。

一个文件里可以有多张表：

```json
[
  {
    "StratagemSettings_Eagle": {
      "_metadata": { "...": "..." },
      "items": [
        {
          "index": 1,
          "type": "3 <=> StratagemType_EagleBomb",
          "debug_name": "EAGLE. 500KG BOMB",
          "cooldown_duration_success": 15.0
        }
      ]
    }
  }
]
```

阅读顺序是表名、`items` 中的单条记录，然后是字段值。

`type` 和 `index` 不是一回事：

- **`type`** 是该条目在游戏中的类型编号。上面的 `3 <=> StratagemType_EagleBomb` 表示飞鹰 500KG 的类型是 `3`。其他表引用它时，例如射弹的 `damage_info_type`、武器的 `projectile_type`，也应使用这个数值查找。
- **`index`** 只表示该记录在当前导出表中的位置，从 `0` 开始，不能用于跨表检索。

应搜索 `"type": "3`，不要搜索 `"index": 3`。两者的数字有时接近，但含义不同。

### 枚举数字为什么会变

源码中使用的是枚举名，例如 `StratagemType_AmmoBackpack`。发布后的客户端通常不再保留这些名称，内存中只剩下整数。若官方在枚举中间插入新项，其后的编号会顺延；源码中的名称引用不受影响，但从内存读取到的数字会发生变化。

因此导出值尽量写作 `数值 <=> 名称`。无法匹配名称时保留 `Unknown`。数字可能随版本变化，名称用于对应其含义，但后补的名称仍可能有误。

### 战备和武器自定义

`generated_stratagem_settings.json` 包含战备类别、指令、冷却、呼叫时间和投送内容。它可以用来核对某项配置是否存在，但不能据此推断全部实战逻辑。

`generated_weapon_customization_settings.json` 记录配件、弹匣、瞄具、枪口、下挂、扳机和涂装。条目通过 `id` 与 `add_path` 关联；`add_path` 中的哈希可在 entity deltas 中查看该配件修改的字段。

## Component

发布包路径：`data/entities/`。

一个实体包含多份组件数据。生命、武器、感知等组件各自独立，不能互相替代。

- Settings 是公共表，一条可以被很多对象引用。
- Component 按实体列出同一类组件的参数。

例如：

- `HealthComponentData.json`：生命、耐久、爆炸伤害乘数；
- `StatusEffectReceiverComponentData.json`：部位对燃烧、毒气、眩晕等效果的阈值；
- `WeaponDataComponentData.json`、`WeaponMagazineComponentData.json`、`WeaponHeatComponentData.json`：武器本体、弹匣、散热；
- `SensorEyeComponentData.json`、`SensorEarComponentData.json`、`SensorDangerComponentData.json`：感知；
- `ShieldComponentData.json`、`DamageZoneShieldComponentData.json`：护盾。

组件文件用资源哈希当键：

```json
{
  "_metadata": { "...": "..." },
  "entities": [
    {
      "0x...": {
        "name": "Spore Spewer",
        "name_zh": "孢子喷涌虫",
        "health": 2500
      }
    }
  ]
}
```

这里只包含该实体在当前组件表中的字段。该实体的其他组件需要到对应文件中查看。

`HealthComponentData.json` 和 `StatusEffectReceiverComponentData.json` 特别大，是因为里面有部位、效果和多层固定数组。这些层级现在按 typelib 切分；剩下的 `unk` 是用途还没确认。

### 从“焦土”查到射弹、直击和爆炸

其他武器也按同一路径查询：取得字段值后，到下一张表中匹配 `type`。

1. 在 `Hash.csv` 搜“焦土”：

   ```text
   entry,能量武器,"=""17196401230144941076""",PLAS-1 Scorcher (Model),PLAS-1 焦土（主武器，模型）
   ```

   十进制哈希是 `17196401230144941076`。

2. 打开 `entities/ProjectileWeaponComponentData.json`，搜这个哈希或“焦土”。其中：

   ```json
   "projectile_type": "142 <=> ProjectileType_Unknown_142"
   ```

3. 打开 `settings/generated_projectile_settings.json`，搜 `"type": "142`。不要搜 `"index": 142`。这条射弹里还有：

   ```json
   "damage_info_type": "54 <=> DamageInfoType_Unknown_54",
   "explosion_type_on_impact": "152 <=> ExplosionType_Unknown_152"
   ```

4. 打开 `settings/generated_damage_settings.json`，搜 `"type": "54`。类型 54 的 `damage` 是 `[100, 50]`，也就是标准伤害 100、耐久伤害 50。

5. 打开 `settings/generated_explosion_settings.json`，搜 `"type": "152`，看范围和爆炸标志。这条爆炸自己的 `damage_type` 当前是 `300 <=> DamageInfoType_Unknown_300`，再回伤害表搜 `"type": "300`。

```text
名称表 → 射弹武器组件 → projectile_type
      → 射弹 Settings → damage_info_type / explosion_type_on_impact
      → 伤害 Settings / 爆炸 Settings → 爆炸自己的 damage_type
```

射弹、伤害、爆炸各有数百个枚举项，目前尚未逐项补全名称，因此会显示 `Unknown`。焦土这发射弹的原名是 `ProjectileType_Plasma_Bolt_Medium`。

## entity deltas

发布包路径：`data/entity_deltas/`。

这些文件不是完整组件，而是相对基础数据的修改。其中很多记录属于武器配件，用于修改特定武器的配件、弹匣、瞄具、枪口或下挂字段。基础模板提供默认值，delta 只列出该实体被修改的位置。

### `entity_deltas_raw.json`

保留原始字节：

- `modified_components`：被改过的组件；
- `component_index`：组件类型索引；
- `type_name` / `type_hash_hex`：识别出来的组件名和类型哈希；
- `offset`：改动在组件内的字节位置；
- `size`：改了多少字节；
- `data_hex`：原始十六进制。

`data_hex: "00003444"` 本身不是可以直接阅读的数值。需要先确定组件、偏移和数据类型，才能解释其含义。

### `entity_deltas_decoded.json`

能识别的差异会写成字段路径：

```json
{
  "WeaponDataComponentData.visibility_modifier": 0.5,
  "WeaponDataComponentData.weapon_stat_modifiers[0].type": "0 <=> WeaponStatModifierType_Add_Ergonomics",
  "WeaponDataComponentData.weapon_stat_modifiers[0].value": -3.0
}
```

点号表示层级，`[0]` 表示数组的第一项。它比 raw 文件易于阅读，但有两处限制：

1. `2026-08-12` 这份 decoded 生成时只加载了 18 类组件；raw 里的组件类型更多。
2. 没有配置的差异会跳过，元数据里是 `"patches_skipped_no_config": 1982`。这些字节仍在 raw 里。后来的结构又按 typelib 校正过，decoded 路径和最新配置不一致时，以最新配置和 raw 为准。

查询配件效果时优先查看 decoded。需要核对原始字节或补充未解码的部分时，再查看 raw。delta 不是武器的完整属性，只包含与模板不同的内容。

## JSON 里的几种形状

### 对象 `{ }`

```json
{
  "health": 2500,
  "armor": 2
}
```

一组字段名和值。

### 数组 `[ ]`

```json
"button_combination": [
  "3 <=> StratagemButtonDirection_Down [↓]",
  "4 <=> StratagemButtonDirection_Left [←]"
]
```

数组保持顺序。`items[0]` 是第一项，`items[1]` 是第二项。相邻的相似项目可能对应不同部位、阶段或状态。固定长度数组末尾成片的 `None` 或全零项，通常是预留的空槽。

### 内联数组和指针数组

- **inline**：数据直接排列在当前结构中，长度通常固定。
- **pointer**：当前位置只保存地址和数量，实际数组位于其他地址，长度可以变化。

导出后，两者通常都会还原为普通数组。只有在检查空数组的原因或修改转储配置时，才需要区分它们。

### 结构体

相关字段包在一起，JSON 里就是嵌套对象：

```json
"spread_info": {
  "horizontal": 10.0,
  "vertical": 10.0
}
```

增加一层嵌套并不一定表示另一个独立对象，多数情况下只是将相关数值组合在一起。

## 枚举

枚举是用整数表示固定选项。导出时尽量写成：

```text
3 <=> UnitSize_Massive
```

左边是内存中的数值，右边是当前匹配到的名称。两者同时保留，便于核对。

也可能是：

```text
HitEffectReceiverType_Unknown_27
```

数值 `27` 已经读出，但枚举表中没有可靠名称。不能因为它位于某个已知枚举项附近，就认定其含义。

官方大约一年半前移除了原来的枚举元数据，现在的名字是用旧资料、游戏表现和手工表补回来的。

- 已有名称以后仍可能修正；
- `Unknown` 表示该枚举项没有匹配到名称，也可能是版本更新后新增的值；
- 枚举类型显示为 `0x` 加 8 位十六进制时，表示类型原名缺失，只保留 4 字节类型哈希；
- `unk` 表示用途尚未确认；键名本身为 8 位类型哈希时，表示整个嵌套结构或数组没有原名；
- 引用时保留原始数值，例如 `4 <=> StatusEffectSusceptibilityType_Fire`。

## 未确认字段

先确认文件、`game_version` 和 `patch_date`，再确认 `name`、`name_zh`、`debug_name` 和哈希。字段应查看完整路径，例如 `zones[0].susceptibilities[0].damage_multiplier`，不要只截取最后一级名称。

随后在同一组件中比较不同敌人或武器，并用相邻版本的差异确认实际变化。该数值是否影响实战，仍需通过游戏内测试验证。

字段名包含 `damage`，并不表示它就是面板上的最终伤害。数组中存在一项，也不表示游戏一定会使用它。

## 当前进度

`v1.007.000`（2026-08-12）已归档：

- 8 类 Settings：伤害、爆炸、射弹、光束、电弧、状态效果、战备、武器自定义；
- 46 类实体组件，配置和 JSON 都按这份 typelib 的布局导出；
- entity deltas 的原始差异、组件索引、索引到类型的对照，以及当时能解码的可读结果；
- 一部分枚举名和 flags 展开。

还在核对未知字段的用途、数组里哪些槽位真正有效，以及更多组件的 delta 解码。射弹、伤害、爆炸的枚举项以后可能会补。

## 速查

- **JSON**：文本数据；`{}` 是对象，`[]` 是数组。
- **Settings**：多处共用的配置表。
- **Entity / 实体**：敌人、武器、射弹、战备投送物、环境物体等。
- **Component / 组件**：实体身上的一类数据，例如生命、状态接收、武器散热。
- **资源哈希**：64 位编号，标识一个具体资源或实体。
- **类型哈希**：32 位编号，JSON 中通常写作 8 位十六进制，用于标识一种结构体或枚举。原名缺失时，以该哈希作为键名。
- **Typelib**：游戏的 `LTLD` 类型库。它给出大小、偏移和数据类型；字段名已经被官方删掉。
- **结构体**：按固定布局放在一起的一组字段。
- **枚举**：整数和固定名称的对照。
- **Flags / 位标志**：一个整数的各个二进制位；`*_detail` 是拆开后的结果。
- **Delta**：相对基础模板改过的部分，不是完整实体。
- **`unk` / `Unknown`**：布局或数值已知，名字或用途还没有确认。

## 问题反馈

名称、枚举、字段名或数据类型有错，可以反馈：

- [哔哩哔哩个人主页](https://space.bilibili.com/300385406)

致谢：

- [HelldiversData](https://github.com/shalzuth/HelldiversData)
- [filediver](https://github.com/xypwn/filediver)
