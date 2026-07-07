# 项目更新说明与阅读指南

> 本文面向希望自行查证游戏数值的普通玩家。它介绍本项目导出的 JSON 数据如何获取、如何打开，以及怎样避免常见的误读。  
> 数据来自游戏运行时的内存转储与解析，并非 Arrowhead 官方源码；它能反映转储时对应版本的配置，但不应单独作为机制结论或版本前瞻的唯一依据。  
> **注：可能有少量泄露内容。**

## 项目简介

社区可依赖游戏内附带的元数据（枚举名称、各组件配置表等）批量导出数值的项目为 [HelldiversData](https://github.com/shalzuth/HelldiversData)。官方移除这些元数据后，该项目停止更新。
本项目是以运行时内存转储并建立结构解析，通过参考过往数据与游戏中实际测试，尽量导出关键数据。因字段布局与语义已大幅变化，结果中含 `unk` / `unknown` 及待核实内容。

## 结论：

可查找以下内容：

- 伤害、爆炸、射弹、光束、电弧、状态效果、战备等基础配置；
- 武器自定义中可用的配件、弹匣、瞄具、涂装等条目与修改内容；
- 实体的生命、状态接收器、武器、装填等组件数据；
- 相同数据在不同版本的变化。

但请注意以下限制：

1. **数据是配置，不一定等于最终实战结果。** 实际效果还可能受游戏机制、部分硬编码数据、服务器逻辑及其他组件共同影响。
2. **名称不等于功能说明。** 字段名能说明的部分会尽量保留；`unk`、`unknown` 或只有数字的字段，表示其含义尚未确认。
3. **版本必须匹配。** 游戏更新后，字段位置、枚举编号和数值都有可能变化。讨论数值时，请同时注明游戏版本和转储日期。

## 一、如何获取指定版本的数据

每次更新转储时，项目会同时提供两种获取方式：

- **提交记录（Commit）**：适合查看改动。
- **发布包（Release）**：适合直接下载和阅读。

建议的使用方式：

1. 想看某一次更新后的数据：下载对应的 Release 压缩包并解压。
2. 想对比变化：在仓库中比较相邻 Commit 的同名 JSON 文件。

不要只根据文件名日期判断版本。每个导出 JSON 的开头通常有 `_metadata`，其中的 `game_version` 与 `patch_date` 才是该文件自身记录的版本信息。例如：

```json
{
  "_metadata": {
    "patch_date": "2026-07-07",
    "game_version": "1.006.301"
  }
}
```

`patch_date` 和 `game_version` 用于核对它对应的游戏更新，优先看文件内元数据。

## 二、怎样打开 JSON 文件

JSON 是纯文本格式，不需要专用游戏工具。较大的文件可能有几十万行，请使用能处理大文件的编辑器。

### 推荐方式

- **Visual Studio Code（推荐）**：免费，支持折叠、搜索、格式化和大文件阅读。安装后直接拖入 JSON 文件即可。
- **Notepad++**：适合只做搜索、快速浏览或复制片段。
- **网页 JSON 查看器/格式化器**：适合小文件或临时查看。请勿上传未公开的数据、带有个人信息的日志或来源不明的文件。

### 最实用的阅读操作

1. 用 `Ctrl + F` 搜索武器、敌人或字段名。
2. 点击对象左侧的折叠箭头，只展开当前需要的层级。
3. 搜索十六进制哈希，例如 `0xAA28CAF964D05500`，可精确定位实体。
4. 阅读一个数值时，同时查看它所在的对象、父级数组及同级字段，避免断章取义。
5. 如果文件很大，优先搜索 `name`、`name_zh`、`debug_name`、`type` 或 `hash_hex`，不要从头手动翻阅。

网页编辑器对超大文件可能卡顿、截断或导致浏览器崩溃；这种情况请改用 VS Code。

## 三、先认识一条数据

以下是经过简化的实体记录形状：

```json
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
```

- 最外层的 `0x...` 是十六进制哈希，适合精确检索。
- `hash` 是同一个哈希的十进制写法。大整数用字符串保存，以避免部分软件丢失精度。
- `name` / `name_zh` 是已匹配到的英文/中文名称；显示 `N/A` 只表示目前没有匹配到名称，不代表该实体无效。
- `index` 是此条目在对应哈希表中的内部索引。它主要用于解析与排查，通常不是游戏内的“编号”。
- 其他字段才是该组件导出的配置值。

## 四、什么是哈希值

哈希值可以把它理解为游戏给一个对象分配的“长编号”。《绝地潜兵 2》会将内部资源路径，例如：

```text
content/fac_helldivers/equipment/throwables/caltrops_grenade/caltrops_mine
```

按 **MurmurHash64A** 算法计算为一个 64 位数字，再用该数字引用武器、敌人、模型、配件、音效或其他资源。当前已验证的规则是：使用标准 MurmurHash64A、`seed = 0`，且路径末尾不附加空字符。

发布包会附带 `Hash.csv`，它是人工维护的对照表，常见格式为：

```text
10845250369047350884    AR-23 解放者（主武器，模型）    AR-23 Liberator (Model)
```

左边是十进制哈希，中间是中文名称，右边是英文名称。同一个值也可能在 JSON 中显示为十六进制形式，例如：

```text
10845250369047350884  =  0x968211C0033DCE64（示意）
```

### 查哈希时的注意事项

- 同一个实体可能存在“本体”“模型”“支架(即空投仓)”“射弹”“占位符”等多个哈希；名称相近不表示它们是同一个对象。
- 很多字段存的只是对其他资源的引用，例如 `payload`。把该数值再到 `Hash.csv` 中搜索，才可能知道它指向什么。
- `name` / `name_zh` 显示为 `N/A`，通常只表示该哈希尚未被手动收录到名称对照表，不表示该实体不存在、无效或没有名称。
- 名称表只记录了部分哈希，出现 `N/A` 是正常现象；也不应凭相邻条目猜测其含义。
- 名称对照也可能有遗漏、旧名称或待核实名称；以游戏内行为、数据位置和多版本对比为准。

## 五、Settings：游戏的“公共配置表”

路径：`data/settings/`

Settings 可以理解为一批按类型排列、可被许多对象共同引用的全局配置表。它不直接描述“某一只具体敌人”或“某一把具体武器实例”，而是保存通用规则和条目定义。

当前目录主要包括：

- `generated_damage_settings.json`：伤害相关配置；
- `generated_explosion_settings.json`：爆炸相关配置；
- `generated_projectile_settings.json`：射弹相关配置；
- `generated_beam_settings.json`、`generated_arc_settings.json`：光束与电弧相关配置；
- `generated_status_effect_settings.json`：状态效果及其模板；
- `generated_stratagem_settings.json`：战备条目；
- `generated_weapon_customization_settings.json`：武器自定义条目。

Settings 文件通常由一个或多个“表”组成。常见层级如下：

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

阅读顺序是：**表名 → `items` 中的一条记录 → 字段值**。

同一份 Settings 中，请务必分清 `type` 与 `index`：

- **`type`**：该条目在游戏中的**类型编号**。例如上面的 `3 <=> StratagemType_EagleBomb`，表示“飞鹰500KG”这个战备的类型是 `3`；本项目尽量把它翻译成可读名称。跨文件引用时（例如射弹表里的 `damage_info_type`、武器组件里的 `projectile_type`），应始终按这个 `type` 数值去另一张表里查找。
- **`index`**：该条目在**当前这张导出表**中的排列位置，从 `0` 开始计数。它只说明“这条记录排在第几个”，**不是**类型编号，也不能拿来跨表检索。

因此：搜索时必须找 `"type": "3`，绝不能找 `"index": 3`。同一张表里，`type` 与 `index` 的数字有时碰巧接近，但二者含义完全不同。

#### 为什么枚举的数字会变？

在游戏开发过程中，官方源码里保存的是**枚举名**（例如 `StratagemType_AmmoBackpack`），开发者按名称引用，名称本身通常稳定。正式发布后的游戏客户端里，这些名称大多已被去掉，内存中只剩下对应的**整数编号**。

如果官方在某个枚举类型中间插入了新选项，其后条目的编号往往会整体顺延；开发者不受影响，因为他们看的仍是枚举名。玩家和社区读到的却是变化后的数字，所以跨版本对比时会出现“同一个东西，数字变了”的情况。这也是本项目尽可能同时导出 `数值 <=> 名称`、并在名称不确定时保留 `Unknown` 的原因：数字可能随版本漂移，名称才是尽量对齐语义的线索，但人工补回的名称仍可能有误。

### 战备与武器自定义数据

`generated_stratagem_settings.json` 增加了战备相关资料，可查看战备类别、指令、冷却、呼叫时间、投送内容与若干标志位。它很适合核对“表面说明之外是否存在某项配置”，但不应据此直接推断全部实战逻辑。

`generated_weapon_customization_settings.json` 记录武器自定义相关条目，例如配件、弹匣、瞄具、枪口、下挂、扳机与涂装等。它通过`id` 与 `add_path` 进行匹配，可以通过后者的哈希值在 entity deltas 中查看配件的具体修改效果。

## 六、Component：实体身上的“模块数据”

路径：`data/entities/`

可以把实体想成一台由多个模块拼成的机器。`Component`（组件）就是其中一个模块：生命模块负责生命相关参数，武器模块负责武器相关参数，感知模块负责视听/危险感知相关参数。一个实体可以同时拥有多个组件。

与 Settings 的核心区别如下：

- **Settings**：全局、按类型排列的公共配置表；一条表项可以被多个对象引用。
- **Component**：按实体导出的组件记录；同一组件文件中会列出很多实体各自的参数。

例如：

- `HealthComponentData.json`：实体生命、耐久度、爆炸伤害乘数等生命系统数据；
- `StatusEffectReceiverComponentData.json`：实体或部位对燃烧、毒气、眩晕等状态效果的触发阈值；
- `WeaponDataComponentData.json`、`WeaponMagazineComponentData.json`、`WeaponHeatComponentData.json`：武器本体、弹匣、散热等数据；
- `SensorEyeComponent.json`、`SensorEarComponent.json`、`SensorDangerComponent.json`：实体感知相关数据；
- `ShieldComponentData.json`、`DamageZoneShieldComponentData.json`：护盾相关数据。

组件文件通常采用“哈希作为键”的形式：

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

这表示：在 `HealthComponentData` 这张组件表里，某个哈希对应的实体被解析出了 `health` 等字段。它不表示该实体只有生命组件，也不表示这个文件包含该实体的全部数据。

### 为什么有些组件文件特别长、特别复杂

`HealthComponentData.json` 与 `StatusEffectReceiverComponentData.json` 包含多层结构、部位、效果与固定长度列表，因此体积显著大于简单组件。这一批长期缺少公开更新的复杂结构现已纳入导出，不过存在不少的未知字段。

### 完整示例：从“焦土”查到射弹、直击伤害与爆炸

下面以 `PLAS-1 焦土`为例，演示如何沿着引用关系查询。这个过程同样适用于其他武器；关键是每次都根据字段值进入下一张表。

1. 在 `Hash.csv` 中搜索“焦土”，可找到：

   ```text
   能量武器,"=""17196401230144941076""",PLAS-1 焦土（主武器，模型）,PLAS-1 Scorcher (Model)
   ```

   取得十进制哈希 `17196401230144941076`。

2. 打开 `entities/ProjectileWeaponComponentData.json`，搜索这个哈希，或搜索“焦土”。对应记录的 `projectile_type` 为：

   ```json
   "projectile_type": "140 <=> ProjectileType_Unknown_140"
   ```

3. 打开 `settings/generated_projectile_settings.json`，搜索完整的字段值 `"type": "140`，定位到射弹类型 140。**务必按 `type` 查找，绝不是查找 `index: 140`。** `index` 只是该表中的记录位置，不能代替类型编号。

   此射弹记录中可继续读到：

   ```json
   "damage_info_type": "54 <=> DamageInfoType_Unknown_54",
   "explosion_type_on_impact": "146 <=> ExplosionType_Unknown_146"
   ```

   前者是直击伤害类型，后者是命中时触发的爆炸类型。

4. 打开 `settings/generated_damage_settings.json`，按 `"type": "54` 搜索（注意空格），即可查看伤害类型 54 的 `damage`、穿甲、拆毁值、元素和状态效果等字段。当前记录的 `damage` 为 `[100, 50]`；代表焦土的直击标准伤害为100，耐久伤害为50。

5. 打开 `settings/generated_explosion_settings.json`，按 `"type": "146` 搜索，即可查看爆炸类型 146 的范围、爆炸标志等字段。该爆炸记录还会以 `damage_type` 引用自己的伤害类型（当前为 `298 <=> DamageInfoType_Unknown_298`）；再按 `"type": "298` 到伤害表查询，才能得到这部分爆炸伤害的具体配置。

这条链路可以概括为：

```text
名称表 → 射弹武器组件 → projectile_type
      → 射弹 Settings → damage_info_type / explosion_type_on_impact
      → 伤害 Settings / 爆炸 Settings → 爆炸引用的 damage_type
```

射弹、伤害、爆炸的枚举名仍显示为 `Unknown`，因为射弹、伤害、爆炸的枚举类型达到了数百个，手动维护过于耗时，因此没有对上述三个枚举值进行有效名称的映射。（尽管我们知道焦土的射弹叫`ProjectileType_Plasma_Bolt_Medium"`）

## 七、entity deltas：为什么它和普通 JSON 不一样

路径：`data/entity_deltas`

这组文件记录的不是一个完整 Component，而是**实体相对于基础数据的差异补丁**。其中相当一部分记录与武器配件有关：它们会为具体武器覆写配件、弹匣、瞄具、枪口、下挂等关联组件的部分字段值。可以把它理解为：

> 基础模板说“默认值是什么”；Delta 说“这个具体实体把模板中的哪些位置改成了什么”。

### `entity_deltas_raw.json`：原始差异

该文件保留差异的原始字节内容，关键字段包括：

- `modified_components`：被修改的组件列表；
- `component_index`：组件的内部索引；
- `type_name` / `type_hash_hex`：已识别时的组件名称与类型哈希；
- `offset`：改动在组件结构中的字节位置；
- `size`：改动字节数；
- `data_hex`：原始十六进制字节。

例如 `data_hex: "00003444"` 本身不是一个可直接阅读的“伤害数值”。必须先知道它在什么组件、哪个偏移、按什么数据类型解释，才能还原其意义。

### `entity_deltas_decoded.json`：已解码差异

该文件会使用本项目已经配置好的组件结构，把能识别的原始差异翻译成可读路径：

```json
{
  "WeaponDataComponentData.visibility_modifier": 0.5,
  "WeaponDataComponentData.weapon_stat_modifiers[0].type": "0 <=> WeaponStatModifierType_Add_Ergonomics",
  "WeaponDataComponentData.weapon_stat_modifiers[0].value": -3.0,
}
```

这里的点号路径表示层级；带 `[0]` 的部分表示数组的第一个元素。它比 raw 文件更适合玩家阅读，但仍有两个限制：

1. 当前这份 decoded 文件只加载了 15 类组件的结构配置，而 raw 文件中实际出现了 27 类组件。
2. 没有配置、无法判断类型或尚未解析的部分会被跳过；本次元数据记录了 `patches_skipped_no_config": 1867`。这类内容仍可在 raw 文件中查看原始字节。

因此，查询武器配件效果时，**优先阅读 `entity_deltas_decoded.json`；只有需要验证、补充解析或排查未知字段时，才回到 `entity_deltas_raw.json`。** 不能把 Delta 当作实体的完整面板，也不能把某条配件 Delta 当作整把武器的全部属性；它只列出相对基础模板不同的内容。

## 八、读 JSON 必备的四个概念

### 1. 对象：`{ }`

花括号表示一组“字段名：值”。例如：

```json
{
  "health": 2500,
  "armor": 2
}
```

它可以理解成一张小表：生命值是 2500，护甲值是 2。

### 2. 数组：`[ ]`

方括号表示按顺序排放的多个值。例如：

```json
"button_combination": [
  "3 <=> StratagemButtonDirection_Down [↓]",
  "4 <=> StratagemButtonDirection_Left [←]"
]
```

这表示一串有顺序的战备指令。数组从第 0 项开始计数，所以 `items[0]` 指第一项，`items[1]` 指第二项。

数组中出现多个相似对象时，应注意每一项可能对应不同部位、不同阶段或不同状态。固定长度数组中，后面大量全零的`None` 项常常只是预留槽位，不视为有效条目。

### 3. 内联数组与指针数组：为什么有两种

这是转储格式为了还原游戏内存结构而保留的技术差异；普通阅读时不必纠结，但理解它能避免误读。

- **内联（inline）数组**：数据直接紧跟在当前结构中，长度通常固定。可理解为“表格里预先留好的几格”。
- **指针（pointer）数组**：当前位置只存“数据在哪里、共有多少项”，真正的数组在另一个位置，长度可以变化。可理解为“表格里放了一张写有地址和数量的便签”。

导出的 JSON 会尽量把两者都还原为普通的 `[ ... ]`。因此玩家只需阅读结果；只有在复现转储、检查配置或理解为什么某个数组为空时，才需要关心它原本是 inline 还是 pointer。

### 4. 结构体：有层级的一组固定字段

结构体可以简单理解为“打包在一起的一组相关数据”。JSON 中通常表现为嵌套对象：

```json
"spread_info": {
  "horizontal": 10.0,
  "vertical": 10.0
},
```

这里 `spread_info` 是一个结构体，内部两项共同描述武器的散布（水平方向与垂直方向）。结构体嵌套并不代表每一层都是独立游戏对象；很多时候它只是为了把相关字段归类。

## 九、什么是枚举，怎样读枚举值

枚举（enum）就是“用数字代表固定选项”。例如某个字段的数字 `3` 不只是任意数字，而可能代表“巨大体型” “状态类型” “开火模式”。

本项目会尽可能把数字翻译为名称，常见格式是：

```text
3 <=> UnitSize_Massive
```

读法是：该字段原始数值为 `3`，目前映射到 `UnitSize_Massive`。数字和名称同时保留，方便核对。

还可能看到：

```text
HitEffectReceiverType_Unknown_27
```

这表示数值 `27` 已被读到，但当前枚举表没有可靠名称。请不要因为它排在已知枚举附近，就自行断言它一定代表某种效果。

### 枚举名称的可靠性说明

官方在约一年半前移除了原有的枚举字段与元数据。现在的枚举名由项目根据旧资料、行为验证和人工维护逐步补回。因此：

- 已知枚举名称也可能需要修正；
- `unknown` 代表该枚举值目前没有映射到对应的名称，也可能是游戏更新后，枚举值超出了有效的枚举项；
- 字段名中的 `unk` 同样表示官方后续加入、目前功能未知或尚未证实的内容；
- 引用时请优先同时保留原始数值，例如 `4 <=> StatusEffectSusceptibilityType_Fire`，而不是只写名称。

## 十、如何正确解读不熟悉的字段

建议按以下顺序判断：

1. **先确认文件与版本**：它来自哪个组件/Settings 表，元数据是什么版本。
2. **确认对象是谁**：看 `name`、`name_zh`、`debug_name` 和哈希。
3. **确认字段的完整路径**：例如 `zones[0].susceptibilities[0].damage_multiplier`，不要只截取最后的 `damage_multiplier`。
4. **与同类对象横向比较**：同一组件内比较不同敌人/武器。
5. **与相邻版本纵向比较**：用 Git diff 或 JSON 对比工具确认真正变化的字段。
6. **回到实测验证**：是否影响实战，仍应通过游戏内测试或可靠机制资料交叉验证。

尤其要避免以下推论：

- “字段名有 `damage`，所以它一定是最终伤害。”
- “数组里有一项，所以游戏一定会用到它。”

## 十一、当前完成情况

### 已完成并可直接阅读的内容

截至 `v1.006.301`，已导出并随版本归档的资料包括：

- 8 类 Settings：伤害、爆炸、射弹、光束、电弧、状态效果、战备、武器自定义；
- 32 类实体组件：生命、状态接收器、武器相关组件、护盾、感知、挂载等；
- 复杂的 `HealthComponentData` 与 `StatusEffectReceiverComponentData` 结构；
- entity deltas 的原始差异、组件索引使用情况、索引到类型的对照，以及可解码组件的可读差异结果；
- 部分枚举名称与 flags（位标志）的可读展开。

此外，有组件仍在继续核对

- 已有组件中的未知字段、嵌套结构、数组实际有效长度和枚举映射；
- 更多 Component 在 entity deltas 中的字段解码。
- 如果有时间，可能会更新射弹、伤害、爆炸的枚举表，Not today but maybe one day

## 十二、速查

- **JSON**：一种文本数据格式；`{}` 是对象，`[]` 是数组。
- **Settings**：游戏共用的全局配置表。
- **Entity / 实体**：敌人、武器、射弹、战备投送物、环境物体等游戏对象。
- **Component / 组件**：实体身上的一类功能数据，例如生命、状态接收器、武器散热。
- **Hash / 哈希**：用于识别或关联资源的长编号。
- **Struct / 结构体**：一组按固定布局组合的相关字段。
- **Enum / 枚举**：数字与固定名称之间的对照。
- **Flags / 位标志**：一个数字中每一位分别代表开/关状态；`*_detail` 是本项目展开后的可读结果。
- **Delta / 差异补丁**：相对基础模板被单独改写的部分，不是完整实体数据。
- **`unk` / `unknown`**：含义尚未确认；请保留怀疑，不要当作已知机制。

## 问题反馈

如果发现名称匹配、枚举映射、字段名称、数据类型等存在错误，欢迎反馈：
- [哔哩哔哩个人主页] (https://space.bilibili.com/300385406)

特此致谢：
- [HelldiversData] (https://github.com/shalzuth/HelldiversData)
- [filediver] (https://github.com/xypwn/filediver)
