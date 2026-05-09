# Axn-Plus 项目概念

---

## 是什么

Axn-Plus 是一个**不基于 Ren'Py** 的类 Ren'Py 视觉小说引擎，以 Python 为核心，面向有编程能力的开发者。

配套工具 **Axn-Editor** 提供全 GUI 编辑体验，面向不想直接写代码的创作者。

---

## 核心理念

**不妥协的 Python 能力，换取更现代的开发体验。**

Ren'Py 为了跨平台和易用性，对 Python 做了大量限制和魔改。Axn-Plus 反其道而行之——直接拥抱完整的 CPython 生态，代价是放弃部分平台。

---

## 核心特性

**全量 Python 支持**
不需要 `python:` 块切换上下文，任何地方都可以直接写 Python，完整支持第三方库和 C 扩展。

**运行时注入**
支持动态加载/替换原生库（.so / .dll），实现热重载和运行时扩展，这是放弃 iOS 和 Web 的根本原因。

**新脚本格式 `.apy`**
自研脚本格式，语法简洁度对标 Ren'Py 的 `.rpy`，但与 Python 更自然地融合，消除上下文切换的割裂感。同时原生支持纯代码工作流，`.apy` 不是必须的。

**可插拔 UI 后端**
引擎核心与渲染层解耦，初期支持 Pygame 和 PySide（Qt）。UI 描述保持后端中立，通过引擎提供的抽象类实现，不暴露任何后端特定概念。

---

## 平台支持

| 平台 | 支持 | 原因 |
|------|------|------|
| Windows | ✅ | |
| macOS | ✅ | |
| Linux | ✅ | |
| Android | ✅ | 允许动态库加载（Android 5+） |
| iOS | ❌ | 不支持运行时动态库注入 |
| Web | ❌ | 环境本身无法运行 |

---

## `.apy` 脚本格式

### 设计原则

**每行的语义由行首决定，不依赖上下文。** 消除 Ren'Py 中同一裸字符串因位置不同而产生四种语义的歧义问题。

### 执行模型

`.apy` 解析为 AST，引擎指令（`show`、`jump`、`menu` 等）走引擎自身的 dispatch；Python 块保留原始源码字符串，执行时通过 `compile()` + `exec()` 并手动传入文件名和行号偏移，使 traceback 能正确指回 `.apy` 文件位置。

**作用域模型**：引擎维护一个全局 `store` dict，贯穿整个游戏生命周期。所有 Python 块的 `exec()` 调用共享同一个 `store` 作为 `globals`，变量天然跨 label、跨 jump 持久化。引擎内置符号（`show`、`jump` 等 API）通过 `__builtins__` 注入为只读层，用户代码可见但不可覆盖。

### 基础语法

```apy
# 角色定义（静态声明，内联）
# define 默认推断类型为 char；显式声明可用 define char
define eileen:
    name "Eileen"
    color #ff8800
    sprites "assets/eileen/"
    voice_prefix "vo/eileen/"
    default_expression "neutral"
    side_image "ui/eileen_side.png"
    font "fonts/handwriting.ttf"
    type_sound "sfx/type_eileen.ogg"
    dialogue_box "ui/eileen_box.apy::EileenBox"

define char eileen:   # 显式声明，等价于上方写法，意图更清晰

# 角色对话，表情作为行内修饰符；括号内支持裸关键字（布尔 flag）和具名参数
eileen: "你好。" (happy)
eileen: "今天天气不错。" (speed=0.5, voice="vo/001.ogg", nowait)

# 旁白
@ "阳光透过窗户照进来。"

# 位置与可见性（show 不控制表情）
# 指令结构：动词 [子命令] [位置参数...] (具名参数...)
# show 位置参数顺序：角色 → 位置 → duration
show eileen left                                        # 最简写法
show eileen left 0.3                                    # 指定 duration
show eileen left 0.3 (enter=slidein_left)               # 补全具名参数
show eileen (layer=effect)                              # 显式指定层，默认为 sprite 层

# hide 位置参数顺序：角色 → duration
hide eileen                                             # 立即隐藏
hide eileen 0.5                                         # 指定 duration
hide eileen 0.5 (exit=fadeout)                          # 补全具名参数

# 多角色并行：同行逗号分隔 = 并行执行，换行 = 串行执行
show eileen left 0.3 (enter=slidein) as anim_eileen, sophia right 1.0 (enter=slideout) as anim_sophia

# 等待控制
wait                    # 等用户点击
wait 2.0                # 等 2 秒
wait for anim_eileen    # 等特定动画完成
wait for all            # 等所有动画完成（默认行为，显式写出意图更清晰）
wait for any            # 等最先完成的动画

# 场景切换（不隐式清空立绘）
# scene 位置参数顺序：路径 → duration
scene bg_room                           # 切换背景
scene bg_room 0.5                       # 指定 duration
scene bg_room 0.5 (with=fade)           # 补全具名参数

# 清空所有立绘（scene 不负责清空，需显式使用 clear）
clear                   # 立即清空
clear (with=fade)       # 带过渡

# 镜头控制（子命令结构）
# camera move 位置参数顺序：zoom → duration → angle
camera move                             # 无变化（通常配合具名参数使用）
camera move 1.2                         # 指定 zoom
camera move 1.2 0.5                     # zoom + duration
camera move 1.2 0.5 15.0               # zoom + duration + angle
camera move 1.2 0.5 (offset=(100, 0))  # 补全具名参数

# camera shake 位置参数顺序：intensity → duration
camera shake                            # 默认强度和时长
camera shake 10                         # 指定 intensity
camera shake 10 0.5                     # intensity + duration
camera shake 10 0.5 (frequency=30)      # 补全具名参数

# camera reset
camera reset                            # 立即重置
camera reset 0.5                        # 指定 duration

# 音频（子命令结构）
# play 位置参数顺序：路径 → volume → fadein → fadeout
play music "bgm/morning.ogg"                            # 全默认
play music "bgm/morning.ogg" 0.8                        # 指定 volume
play music "bgm/morning.ogg" 0.8 1.0                    # volume + fadein
play music "bgm/morning.ogg" 0.8 1.0 1.0 (loop)        # 补全具名参数
play sound "sfx/door.ogg" 0.6

# stop 位置参数顺序：fadeout
stop music                              # 立即停止
stop music 1.0                          # 指定 fadeout

# 单行 Python（单行内 Python 语法合法即可）
$ flag_met_eileen = True

# 多行 Python
python:
    for item in inventory:
        item.apply()

# 标签（默认动态）；label 签名直接使用 Python 函数签名风格
label morning_scene:
    eileen: "早上好。" (smile)

label morning_scene(mood, weather="sunny"):
    eileen: "早上好。"
    return mood + "_done"   # return 后跟任意 Python 表达式

# 条件
if flag_met_eileen:
    eileen: "好久不见。"
else:
    eileen: "初次见面。"

# 菜单
menu (timeout=10.0, default="拒绝"):
    "答应她":
        $ flag_agreed = True
        jump route_a
    "拒绝" (if=flag_can_refuse):
        jump route_b
    "询问详情" (if=flag_met_eileen, disabled=flag_tired):
        jump route_c
    "隐藏选项" (hidden=flag_secret):
        jump route_secret

# 跳转与调用
jump route_a

# call 不接返回值
call morning_scene("happy")

# call 用 as 接返回值（推荐写法）
call morning_scene("happy") as result

return
```

### 静态与动态修饰符

| 修饰符 | 适用对象 | 含义 |
|--------|----------|------|
| 无修饰符 | 全部 | 默认行为（`define` 默认静态，`label` 默认动态） |
| `sta` | `label` | 强制声明为静态，表达作者意图 |
| `dyn` | `define`、`label` | 显式声明为动态，运行时求值 |

```apy
define eileen:        # 静态（默认，无需标记）
dyn define eileen:    # 动态，运行时求值

label morning_scene:      # 动态（默认）
sta label morning_scene:  # 强制静态，非默认行为，显式标记
```

`sta` 仅在 `label` 上有意义——`label` 默认动态，加 `sta` 表示这是有意为之的静态声明，代码审查时一眼可见。`define` 本身默认静态，`sta define` 无额外语义，不支持此写法。

### 指令结构

所有指令遵循统一结构：

```
动词 [子命令] [位置参数...] (具名参数...)
```

**位置参数**按固定顺序省略键名，用于高频场景的简洁写法；**具名参数**在括号内以 `key=value` 形式提供，用于不常用或顺序不明确的参数。两者可混用。括号内无 `=` 的裸关键字视为布尔 flag（如 `loop`、`nowait`）。

**各指令位置参数顺序**：

| 指令 | 位置参数顺序 | 保留具名参数 |
|------|------------|------------|
| `show` | 角色 → 位置 → duration | `enter` `layer` |
| `hide` | 角色 → duration | `exit` |
| `scene` | 路径 → duration | `with` |
| `clear` | — | `with` |
| `play music/sound/voice/ambient` | 路径 → volume → fadein → fadeout | `loop` |
| `stop music/sound/voice/ambient` | fadeout | — |
| `camera move` | zoom → duration → angle | `offset` `easing` |
| `camera shake` | intensity → duration | `frequency` |
| `camera reset` | duration | — |

**子命令**用于同一动词下行为模式本质不同的场景（如 `camera move` / `camera shake` / `camera reset`，`play music` / `play sound`）。判断标准：参数描述"怎么做"时用具名参数；改变"做什么"时拆为子命令。

### 关键设计决策

**表情控制**：表情只能通过对话行的修饰符 `(expression)` 设置，`show` 指令仅控制位置和可见性，不影响表情状态。避免 Ren'Py 中立绘状态残留的问题。

**Python 块边界**：`$` 后允许任何在单行内 Python 语法合法的内容，包括三元表达式、多重赋值等（如 `$ x = 1 if flag else 2`、`$ a, b = b, a`）。需要换行的复合逻辑使用 `python:` 块，换行符即边界，无需记额外规则。

**角色定义**：内联在 `.apy` 文件中，不使用外部 JSON。`define` 默认推断类型，`define char` 为显式声明，语义等价但意图更清晰。支持 `dyn define` 实现运行时动态定义。

**`show` 层级**：默认操作 sprite 层，需要时通过 `(layer=effect)` 等显式指定。层级扩展需求驱动，不过早设计。

**并行与串行执行**：同行逗号分隔的指令并行执行，引擎默认等所有并行动画完成后推进。需要精细控制等待时机时，用 `as` 给动画命名，再用 `wait for` 显式控制。`wait for` 中 `for` 是介词而非子命令，`wait for all` / `wait for any` / `wait for <name>` 三种形式语义链完整。

**`scene` 不隐式清空立绘**：`scene` 只负责切换背景，不自动 hide 现有立绘。需要清空时显式使用独立的 `clear` 指令。`scene` 不接受 `clear` 参数，职责单一。

**`call` 返回值**：只支持 `as` 写法——`call label() as result`。`_return` 作为引擎内部实现细节，不对外暴露。

**`label` 签名**：直接使用 Python 函数签名风格，支持默认值、`*args`、`**kwargs`。`return` 后跟任意 Python 表达式。

**跨文件引用**：`jump` / `call` 不需要 `import`，引擎启动时自动扫描所有 `.apy` 文件，label 冲突在启动时报错。跨文件引用 `define`（如 UI 控件）需要显式 `import`，使依赖关系可见。label 命名冲突由引擎扫描和 VSCode 插件共同处理，语言层不强制约束。

**UI 控件定义**：在独立的 `.apy` 文件中定义，通过 `文件路径::控件名` 语法引用，如 `"ui/eileen_box.apy::EileenBox"`。

**推断失败行为**：所有默认推断逻辑（层级推断、类型推断等）在推断失败时抛出明确错误，不静默走错分支。

---

## 配套工具：Axn-Editor

面向不想直接写代码的创作者，提供全 GUI 编辑体验。

### 三个工作区

| 工作区 | 目标用户 | 内容形式 |
|--------|----------|----------|
| 文本区 | 编剧 / 美术 | 积木块：对话、图片、音频的线性组合，支持简单嵌套 |
| 窗口区 | 美术 | GUI 窗口与控件编辑器,用来替代Ren'py UI中最严重的问题,没有原生友好的GUI窗口开发方式 |
| 脚本区 | 程序员 | 蓝图：复杂逻辑、条件分支、自定义函数节点 |

**文本区**负责最终组合各种内容（show、call、文本、script 引用等），结构类似 `script.rpy`，但以积木块形式呈现，不需要用户理解语法。

**脚本区**通过蓝图节点表达逻辑，蓝图状态（节点、连线）通过嵌套和引用关系存储在同一 `.apy` 文件中。

**代码区**是 GUI 降级策略的实现：无法用蓝图或积木表达的代码，直接以可编辑代码块呈现，编辑器不尝试解析其内部结构。

### Round-Trip Fidelity

GUI 编辑和代码编辑之间的**双向同步**是核心设计约束：

- GUI 操作 → 生成 `.apy` 代码，风格一致、可读
- 代码编辑 → 反向解析回 GUI 状态，超出 GUI 表达能力的部分保留为代码节点
- 两者来回切换时，不丢失任何信息

**Round-Trip 边界**由语言层级天然划定：

| 层级 | 内容 | GUI 处理方式 |
|------|------|-------------|
| 引擎指令层 | `show`、`hide`、`jump`、`menu`、对话行、`define` 等 | 完全解析，转换为对应积木块或蓝图节点 |
| Python 代码层 | `$` 单行和 `python:` 块 | 整块作为不透明代码节点，直接在节点上编辑原始代码，不转换为其他形式 |

编辑器不尝试解析 Python 内部结构，两种 Python 形式在编辑器中的呈现方式相同，差异仅在体积。

---

## 存档机制

### 作用域与序列化

引擎维护两个持久化对象：

- **`store`**：当前存档槽的游戏状态，随存档槽保存和加载
- **`persistent`**：跨存档的全局数据（画廊解锁、总游玩时长等），游戏退出时自动保存

两者遵守相同的序列化规则。

### 序列化规则

| 场景 | 处理方式 |
|------|----------|
| 基础类型（`int`、`str`、`list`、`dict` 等） | 直接 pickle，零摩擦 |
| 自定义类，简单场景 | `@saveable` 装饰器，pickle 兜底，版本兼容由开发者自负 |
| 自定义类，需要跨版本 migration | 继承 `Saveable`，实现 `__save__` / `__load__`，支持 `__version__` |
| 未声明的自定义类 | 运行时抛 `AxnSaveError`，提示正确声明方式 |

存档格式为 pickle + 版本号头；`Saveable` 子类走自定义序列化路径，绕过 pickle。

### 两种声明方式

**`@saveable`**：轻量声明，引擎用 pickle 兜底。适合字段全为基础类型、不需要精细控制的简单类。

```python
@saveable
class QuestState:
    def __init__(self):
        self.progress = 0
        self.completed = False
```

若类中含不可 pickle 的字段（lambda、文件句柄、socket 等），存档时抛出 `AxnSaveError`，错误信息会指出问题字段并建议改用 `Saveable`。

**`Saveable`**：重量声明，开发者完全控制序列化逻辑。适合需要跨版本 migration 的场景。

```python
class QuestState(Saveable):
    __version__ = 2

    def __save__(self):
        return {'progress': self.progress, 'completed': self.completed}

    def __load__(self, data, version):
        if version < 2:
            # 旧存档没有 completed 字段，补默认值
            data['completed'] = False
        self.progress = data['progress']
        self.completed = data['completed']
```

### 未声明类的错误信息

```
AxnSaveError: Object of type 'QuestState' is not saveable.
Declare it with @saveable or inherit from Saveable.
```

---

## 不是什么

- 不是 Ren'Py 的分支或 fork
- 不是面向零编程基础用户的工具（引擎本身）
- 不追求最大化跨平台覆盖
