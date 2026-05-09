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

### 扩展语法

#### 叙事表达

**对话插值**

对话字符串内用 `{expr}` 插入 `store` 变量或简单表达式，引擎在渲染时求值：

```apy
eileen: "你好，{player_name}。今天是第 {day} 天。"
eileen: "好感度：{relationship['eileen']}/100"
```

插值表达式限制为单一表达式（变量、属性访问、下标、简单运算）。禁止函数调用和赋值——需要复杂计算时先用 `$` 块算好再引用。推断失败时抛出 `AxnInterpolationError`，指明文件位置和表达式内容。

**条件文本（inline conditional）**

```apy
eileen: "我们[已经|还没]见过面。" (if=flag_met)
```

`[A|B]` 语法：`if` 条件为真取 A，否则取 B。省略 B 时条件为假则显示空字符串：

```apy
eileen: "你[（有点憔悴）]看起来不错。" (if=flag_tired)
```

`[A|B]` 仅支持静态字符串片段，不支持嵌套。需要复杂条件分支时用 `if` 块。

**多语言（translate）**

```apy
translate zh:
    eileen: "你好。"
    @ "阳光透过窗户照进来。"

translate en:
    eileen: "Hello."
    @ "Sunlight streams through the window."
```

`translate` 块内只允许对话行和旁白，不允许引擎指令或 Python 块。引擎根据运行时语言设置选择对应块；当前语言无对应翻译时回退到第一个 `translate` 块并输出警告。同一 label 内的 `translate` 块必须紧跟原始对话行之后，解析器在启动时检查完整性。

---

#### 动画与演出控制

**具名动画序列（animation block）**

```apy
animation eileen_enter:
    show eileen right 0.0
    camera move 1.1 0.5
    wait 0.3
    show eileen center 0.4 (enter=slidein)

animation eileen_exit:
    hide eileen 0.4 (exit=slideout)
    camera reset 0.3
```

调用方式与 `call` 一致：

```apy
call animation eileen_enter
call animation eileen_enter as anim   # 用 as 接句柄，配合 wait for 使用
```

`animation` 块内只允许引擎指令（`show`、`hide`、`camera`、`play`、`wait` 等），不允许 Python 块、对话行、`jump`、`menu`。目的是保证 GUI 能完整解析，也防止演出片段携带业务逻辑。

**并行轨道（parallel / track）**

```apy
parallel:
    track a:
        show eileen left 0.5
        wait 0.5
        eileen: "……"
    track b:
        play music "bgm/tense.ogg" 0.8 1.0
        camera shake 5 0.3
```

`parallel` 块内的所有 `track` 并行执行，默认等所有 `track` 完成后推进（等价于 `wait for all`）。需要提前推进时：

```apy
parallel (wait=any):
    track a:
        ...
    track b:
        ...
```

`track` 可命名后用 `wait for` 精细控制：

```apy
parallel (wait=none):           # 不自动等待，手动控制
    track bgm:
        play music "bgm/tense.ogg" 0.8 1.0
    track scene as anim_scene:
        show eileen left 0.5
        camera shake 5 0.3

wait for anim_scene             # 等 scene track 完成后推进
```

`parallel` 块在 GUI 脚本区中表现为时间轴视图，每个 `track` 对应一条轨道。

**立绘状态（sprite states）**

同一角色的不同表情、服装变体通过 `states` 声明管理。引擎默认按 `{角色名}_{state}.png` 命名约定自动扫描 `sprites` 目录，也可以显式声明覆盖：

```apy
define char eileen:
    sprites "assets/eileen/"       # 自动扫描，按命名约定构建状态表
    states:                        # 显式声明，覆盖自动扫描结果
        neutral    "assets/eileen/neutral.png"
        happy      "assets/eileen/happy.png"
        sad        "assets/eileen/sad.png"
        happy_alt  "assets/eileen/casual_happy.png"   # 服装变体
```

状态切换通过对话修饰符触发，是瞬时换帧，不产生过渡动画：

```apy
eileen: "早上好！" (happy)     # 切换到 happy state
eileen: "……"     (sad)        # 切换到 sad state
```

需要带过渡的状态切换时，在修饰符内指定 `transition`：

```apy
eileen: "……" (sad, transition=dissolve)
```

**过渡（transition）**

过渡作用于出入场和场景切换，使用已有的具名参数语法触发：

```apy
show eileen left 0.3 (enter=fade)
hide eileen 0.5 (exit=dissolve)
scene bg_room 0.5 (with=wipe)
```

内置过渡由引擎标准库提供（`fade`、`dissolve`、`wipe`、`pixelate`、`slidein_left`、`slideout_right` 等）。自定义过渡通过 Python 继承实现，不引入新语法：

```python
class SlideFromTop(Transition):
    def __init__(self, duration=0.5):
        self.duration = duration

    def apply(self, surface, progress):
        # progress: 0.0 → 1.0，引擎每帧调用
        offset_y = int((1.0 - progress) * surface.height)
        return surface.offset(0, -offset_y)
```

```apy
show eileen center (enter=SlideFromTop(0.3))   # 直接传实例
```

**transform（对象属性动画）**

`transform` 描述单个显示对象的属性随时间的变化，作用域是对象本身，不涉及演出流程。对标 Ren'Py ATL，但不引入独立子语言——`transform` 块使用有限的引擎关键字，复杂逻辑退到 Python。

```apy
# 定义可复用 transform
transform shake_x:
    keyframe 0.0: x_offset 0        # 单属性：折叠到同行
    keyframe 0.1: x_offset -10
    keyframe 0.2: x_offset 10
    keyframe 0.3: x_offset -8
    keyframe 0.4: x_offset 0
    easing linear
    repeat 1

transform complex_enter:
    keyframe 0.0:                   # 多属性：展开为块
        alpha 0.0
        y_offset -30
        scale 0.9
    keyframe 0.5:
        alpha 0.8
        y_offset -5
        scale 1.02
    keyframe 1.0:
        alpha 1.0
        y_offset 0
        scale 1.0
    easing ease_out
    repeat 1

transform breathe:
    keyframe 0.0:  scale_y 1.0
    keyframe 0.5:  scale_y 1.03
    keyframe 1.0:  scale_y 1.0
    easing ease_in_out
    repeat forever                  # 持续循环
```

keyframe 折叠规则：冒号后有内容 = 单行（单属性）；冒号后为空 = 展开块（多属性）。和 Python 自身的单行/块语法直觉一致。

**transform 可用属性：**

| 属性 | 含义 |
|------|------|
| `x_offset` / `y_offset` | 位移偏移（像素） |
| `scale` / `scale_x` / `scale_y` | 缩放比例 |
| `rotate` | 旋转角度 |
| `alpha` | 透明度（0.0–1.0） |
| `color_multiply` | 颜色乘算（闪红、变灰等），接受 `(r, g, b, a)` |

`easing` 支持内置缓动名或 Python callable：

```apy
transform custom_ease:
    keyframe 0.0: alpha 0.0
    keyframe 1.0: alpha 1.0
    easing lambda t: t * t          # 直接写 Python lambda
    repeat 1
```

内置缓动：`linear`、`ease_in`、`ease_out`、`ease_in_out`、`bounce`、`elastic`。

**应用 transform：**

```apy
show eileen center (transform=shake_x)              # 单个
show eileen center (transform=[breathe, shake_x])   # 多个叠加，按列表顺序合成
```

transform 叠加时属性冲突（两个 transform 同时修改 `x_offset`）取最后一个，不做混合，引擎启动时对已知冲突输出警告。

**复杂动画退到 Python：**

`transform` 块覆盖不了的场景（帧动画、骨骼、物理），直接用 Python 类，`show` 接受任何实现了 `AnimatedSprite` 接口的对象：

```python
class LivePortrait(AnimatedSprite):
    def __init__(self, char):
        self.frames = char.load_frames()
        self.timer = 0.0

    def update(self, dt):
        self.timer += dt
        return self.frames[int(self.timer * 12) % len(self.frames)]
```

```apy
show LivePortrait(eileen) center
```

**`animation` 与 `transform` 的边界：**

- `animation`：引擎指令序列，描述演出流程（谁在哪里、镜头怎么动、什么时候等待）
- `transform`：keyframe 数据，描述单个对象的属性变化

两者组合使用：

```apy
animation eileen_enter:
    show eileen right 0.0 (transform=complex_enter)
    camera move 1.1 0.5
    wait for all
```

`transform` 块在 GUI 脚本区中表现为时间轴 keyframe 编辑器，属性列表完整可解析。Python `AnimatedSprite` 子类作为代码节点处理。

---

#### 数据与逻辑

**`with store` 块（批量状态变更）**

```apy
with store:
    flag_met_eileen = True
    day += 1
    relationship["eileen"] += 5
```

语义上等价于连续写多行 `$`，但意图更清晰——这是一组原子性的状态变更。块内只允许赋值语句，不允许流程控制（`if`、`for`、函数调用等）；违反时抛出解析错误。

**`const` 声明**

```apy
const MAX_RELATIONSHIP = 100
const ROUTES = ["a", "b", "c"]
```

引擎启动时静态求值，写入只读层（与内置符号同层），用户代码不可覆盖。右值必须是字面量或字面量组合，不允许引用 `store` 变量或调用函数。尝试在运行时覆盖 `const` 时抛出 `AxnConstError`。

---

#### 模块化与复用

**`template` / `extends`（UI 组件继承）**

适用于窗口区的 UI 控件定义，在 `.apy` 文件中声明：

```apy
# ui/base.apy
template BaseBox:
    background "ui/box_default.png"
    font "fonts/default.ttf"
    padding (20, 10)
    text_color #ffffff
```

```apy
# ui/eileen_box.apy
import "ui/base.apy"

EileenBox extends BaseBox:
    background "ui/box_eileen.png"
    name_color #ff8800
    font "fonts/handwriting.ttf"
```

`extends` 只继承属性，不支持方法覆盖（UI 控件不是 Python 类）。属性冲突时子类覆盖父类，无歧义。跨文件引用 `template` 需要显式 `import`，使依赖关系可见。

**`include`（脚本片段静态包含）**

```apy
include "common/prologue.apy"
```

编译期展开，等价于将目标文件内容内联到当前位置。与 `jump`/`call` 的区别：`include` 是编译期合并，不产生运行时跳转，也不创建独立的执行上下文。

适用场景：跨章节复用的开场白、结局模板、通用菜单片段等。引擎启动时检测循环 `include`，发现时抛出错误并打印完整引用链。

---

#### Round-Trip Fidelity 补充

扩展语法对 Round-Trip 边界的影响：

| 新增语法 | GUI 处理方式 |
|----------|-------------|
| `{expr}` 插值 | 对话积木块内内联编辑，表达式作为文本字段 |
| `[A\|B]` 条件文本 | 对话积木块内内联编辑，A/B 作为独立字段 |
| `translate` 块 | 对话积木块的多语言标签页 |
| `animation` block | 脚本区独立节点，内容可完整解析为子积木序列 |
| `parallel / track` | 脚本区时间轴视图 |
| `with store` 块 | 脚本区代码节点（与 `python:` 块同等处理） |
| `const` | 脚本区只读常量节点 |
| `template / extends` | 窗口区组件继承树 |
| `transform` block | 脚本区 keyframe 时间轴编辑器，属性列表完整可解析 |
| `AnimatedSprite` Python 类 | 脚本区代码节点（与 `python:` 块同等处理） |
| `transition`（内置） | 参数选择器，下拉列表 |
| `transition`（自定义 Python 类） | 脚本区代码节点 |
| `states` 声明 | 角色定义积木块内的状态表编辑器 |

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
| `call animation` | 名称 | — |
| `parallel` | — | `wait` |
| `include` | 路径 | — |
| `show`（带 transform） | 角色 → 位置 → duration | `enter` `layer` `transform` |

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

**transform 与 animation 边界**：`transform` 描述单个对象的属性动画（keyframe 数据），`animation` 描述演出流程（指令序列）。两者职责不重叠，通过 `(transform=...)` 参数组合使用。`transform` 块内只允许有限的引擎关键字，不允许 Python，保证 GUI 可完整解析；超出表达能力的场景通过继承 `AnimatedSprite` 退到 Python。

**transform 叠加冲突**：多个 transform 同时修改同一属性时，取列表中最后一个，不做混合。引擎启动时对已知冲突输出警告，不静默。

**keyframe 折叠规则**：冒号后有内容 = 单行（单属性）；冒号后为空 = 展开块（多属性）。和 Python 自身的单行/块语法直觉一致，解析器无歧义。

**transition 扩展**：内置过渡由引擎标准库提供，自定义过渡继承 `Transition` 抽象类，通过 `apply(surface, progress)` 接口实现，直接传实例到具名参数，不引入新语法。

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
| 引擎指令层 | `show`、`hide`、`jump`、`menu`、对话行、`define`、`animation`、`parallel`、`translate`、`const`、`include` 等 | 完全解析，转换为对应积木块或蓝图节点 |
| Python 代码层 | `$` 单行、`python:` 块、`with store` 块 | 整块作为不透明代码节点，直接在节点上编辑原始代码，不转换为其他形式 |

编辑器不尝试解析 Python 内部结构，两种 Python 形式在编辑器中的呈现方式相同，差异仅在体积。`with store` 块虽然语法受限，但同样作为代码节点处理，不转换为积木块序列。

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
