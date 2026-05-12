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

**项目级 UI 后端绑定**
引擎核心与渲染层解耦，初期支持 Pygame 和 PySide（Qt）两套后端。后端在项目初始化时选定，之后固定不变——这是项目配置决策，不是运行时动态切换。两套后端对应不同的 UI 子系统，定位和目标用户不同，详见 UI 系统章节。

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

### 解析器模型

**两遍扫描**：解析器分两遍处理 `.apy` 文件。第一遍只扫描顶层 `define` 块，建立全局符号表（角色名 → 类型）；第二遍完整解析，行首遇到已知角色名才走对话路径，否则走指令路径。`define` 只允许出现在文件顶层，保证第一遍扫描无需理解嵌套结构。跨文件时，引擎启动时扫描所有 `.apy` 文件，全局符号表建立后再做完整解析，这也是 label 冲突能在启动时报错的基础。

**`$` 行不支持括号续行**：`$` 后的内容在遇到换行符时立即终止，括号不触发续行。括号不平衡时解析器立即报错并提示改用 `python:` 块，不静默失败。

```
AxnParseError: Unclosed bracket in '$' line (line 12, scene.apy)
  Hint: Multi-line Python belongs in a 'python:' block.
  12 | $ x = (
  13 |     1 + 2
```

### 注释归属规则

注释分两类，归属规则不同：

**行内注释**（`代码  # 注释`）：归属同行节点，精确到行级别。

**行间注释**（单独占行）：归属当前缩进层级的父节点，空行切断与下方节点的联系。顶层行间注释（无父节点）作为独立注释节点存在。

```apy
# 顶层孤立注释，作为独立注释节点

label morning_scene:
    # 归属 morning_scene（父节点），不归属下面的 eileen 对话行
    eileen: "早上好。"  # 归属此对话行（行内注释）
    sophia: "你好。"

# 归属 morning_scene（父节点），空行切断与下方 scene 的联系

scene bg_room
```

GUI 处理：节点移动时注释跟随；节点删除时若有关联注释则弹出确认；注释在编辑器中显示为节点上方的灰色文本框，可编辑。

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

# 角色继承：子角色继承父角色所有字段，显式声明的字段覆盖父定义
define char eileen_adult extends eileen:
    sprites "assets/eileen_adult/"
    voice_prefix "vo/eileen_adult/"
    # 未声明字段全部继承：name、color、dialogue_box 等保持不变

# layers 模型下的继承：同名动态层内按 key 合并，未声明的状态继承父定义
define char eileen_casual extends eileen:
    layers:
        outfit:
            casual "outfit_casual_v2.png"   # 只覆盖此状态
            # school 继承父定义

# 继承规则：
# - 只允许单继承，不允许链式（A extends B extends C）
# - 继承只发生在编译期展开，运行时 eileen_adult 与 eileen 是完全独立的对象
# - show eileen_adult 和 show eileen 互不影响

# 分层立绘：states 和 layers 二选一，不可混用
# states：整图切换模型
define char sophia:
    name "Sophia"
    states:
        neutral  "sophia_neutral.png"
        happy    "sophia_happy.png"
        sad      "sophia_sad.png"
    default_expression "neutral"

# layers：分层叠加模型
define char eileen:
    name "Eileen"
    sprites "assets/eileen/"
    layers:
        body    "body_default.png"           # 静态层，不参与状态切换
        outfit:                              # 动态层，支持换装
            school   "outfit_school.png"
            casual   "outfit_casual.png"
        face:                                # 动态层，表情
            neutral  "face_neutral.png"
            happy    "face_happy.png"
            sad      "face_sad.png"
        brow:                                # 动态层，眉毛
            neutral  "brow_neutral.png"
            angry    "brow_angry.png"
        hair    "hair_default.png"           # 静态层
    expressions:                             # 组合映射：一个修饰符同时设置多个动态层
        happy:  (face=happy, brow=neutral)
        angry:  (face=sad,   brow=angry)
        crying: (face=sad,   brow=neutral)
    default_expression "neutral"
    # layers 叠加顺序规则：
    # 要么全部走声明顺序（不写 z_order），要么全部写 z_order，不允许混用
    # 混用时引擎启动报错

# expression 指令：无对话时切换表情（states 和 layers 模型均支持）
expression eileen happy                      # 走 expressions 映射（layers 模型）或整图切换（states 模型）
expression eileen (face=happy, brow=angry)   # 直接指定各层，绕过 expressions 映射（仅 layers 模型）
expression eileen (outfit=casual)            # 换装（仅 layers 模型）
expression eileen happy (transition=dissolve) # 带过渡效果

# 角色对话，表情作为行内修饰符；括号内支持裸关键字（布尔 flag）和具名参数
eileen: "你好。" (happy)
eileen: "今天天气不错。" (speed=0.5, voice="vo/001.ogg", nowait)
# layers 模型下可直接指定各层（绕过 expressions 映射）
eileen: "……" (face=happy, brow=angry)
eileen: "换装了。" (outfit=casual)
# 表情状态跟着角色对象走，不跟场景走：
# 对话修饰符修改角色的持久表情状态，与角色是否可见无关
# show 出场时使用角色当前表情状态，不重置为 default_expression

# voice 短路径：相对 voice_prefix 的短路径，引擎自动补全扩展名
# 等价于 voice="vo/eileen/001.ogg"（假设 voice_prefix = "vo/eileen/"）
eileen: "你好。" (voice="001")

# 旁白（三种等价写法，风格自选，同一项目内保持一致）
@ "阳光透过窗户照进来。"
narrator: "阳光透过窗户照进来。"     # 与角色行对齐，可读性更好

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

# 场景切换
# scene 默认同时清空 sprite 层（高频用法零开销）
# scene 位置参数顺序：路径 → duration
scene bg_room                           # 切换背景，默认清空 sprite 层
scene bg_room 0.5                       # 指定 duration
scene bg_room 0.5 (with=fade)           # 补全具名参数
scene bg_room (keep)                    # 保留所有立绘
scene bg_room (keep=eileen)             # 只保留 eileen，其余清除
scene bg_room (keep=[eileen, sophia])   # 保留多个

# clear：精确清除，无过渡，不受 scene 影响
clear                           # 清除 sprite 层所有元素
clear eileen                    # 清除指定角色
clear eileen sophia             # 清除多个
clear (layer=effect)            # 清除指定层所有元素
clear eileen (layer=effect)     # 清除指定层上的指定元素

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

# pause / resume（保留进度暂停，与 stop 语义不同）
# pause 位置参数顺序：fadeout
pause music                             # 立即暂停
pause music 0.3                         # 带 fadeout 的暂停（画面渐暗）
resume music                            # 立即恢复
resume music 0.3                        # 带 fadein 的恢复
pause video
resume video 0.3

# 视频（play 子命令扩展）
# play video 位置参数顺序：路径 → volume
# 默认阻塞（blocking），非阻塞时显式加 (async)
play video "cutscene/intro.mp4"                                     # 全默认，阻塞
play video "cutscene/intro.mp4" 0.8                                 # 指定音量
play video "cutscene/intro.mp4" (layer=effect)                      # 指定层
play video "cutscene/intro.mp4" (async)                             # 非阻塞，背景播放
play video "bg/rain_loop.mp4" (layer=bg, loop, async)               # 循环背景视频
stop video                                                          # 立即停止
stop video 0.5                                                      # 带 fadeout

# camera follow（镜头跟随角色）
camera follow eileen                    # 跟随 eileen 的位置
camera follow eileen (lag=0.3)          # 带延迟跟随，更自然
camera follow none                      # 取消跟随

# 层管理
layer create effect (above=sprite)      # 创建层，指定位于 sprite 层之上
layer destroy effect                    # 销毁层
layer order sprite effect ui            # 重排层顺序（从下到上）

# say 动词（专用于说话者在运行时动态决定的场景）
# 静态说话者必须使用 角色: 或 @，用 say 传入静态角色名时报错
$ speaker = get_current_speaker()
say speaker "动态说话者。"              # speaker 是 store 变量，运行时求值
say speaker "下一句。" (happy)          # 修饰符与对话行完全一致

# choice（动态菜单，程序化生成选项列表）
# menu 是静态声明语义（编译期确定），choice 专门处理动态场景
$ options = build_options(day, relationship)
choice options (timeout=10.0)

# 禁用 / 恢复输入
# 形式一：对称写法（enable / disable 独立调用）
input disable                           # 全部禁用
input disable (skip, rollback)          # 只禁用跳过和回滚，保留点击推进
input disable (all)                     # 显式全部禁用（同无参数写法）
input enable                            # 恢复全部

# 形式二：块语法（推荐，引擎保证块结束后自动恢复，异常安全）
input disable:
    play video "cutscene/intro.mp4"
    wait for video
    camera shake 10 0.5
# 块结束后自动 enable，即使中途 jump 或异常也能正确恢复

# 模态框焦点接管
# modal show 隐含 input disable (all)，无需手动管理焦点
modal show "ui/confirm_quit.apy::ConfirmDialog"                     # 无返回值
modal show "ui/confirm_quit.apy::ConfirmDialog" as result           # 阻塞，等用户选择后写入 result
modal hide                                                          # 手动关闭（通常由模态框内部触发）

# modal 配合条件跳转
modal show "ui/confirm_quit.apy::ConfirmDialog" as result
if result == "confirm":
    jump ending_quit
else:
    jump resume_game

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

# 条件（支持 elif 链）
if flag_met_eileen:
    eileen: "好久不见。"
elif flag_heard_of_eileen:
    eileen: "久仰大名。"
else:
    eileen: "初次见面。"

# unless：if not 的语法糖，用于卫语句场景
unless flag_met_eileen:
    jump prologue

# match：多路路由，匹配 store 变量的单一值
# 简单形式（GUI 可完整解析为多路分支节点）
match day:
    1       -> day1_scene
    2       -> day2_scene
    3, 4    -> midgame_scene
    _       -> ending

# match 复杂形式（含表达式或 guard，整块降级为代码节点）
match relationship["eileen"]:
    case _ if _ >= 80:
        jump route_good
    case _:
        jump route_bad

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

# menu 内联跳转：选项只有一条 jump 时可用 -> 省略展开块
menu:
    "答应她" (if=flag_can_agree) -> route_a
    "拒绝"                       -> route_b
    "询问详情" (if=flag_met_eileen):   # 有额外逻辑时退回展开块
        $ log_choice("ask")
        jump route_c

# menu as：选完后继续执行流，选项 -> 右边是返回值表达式而非 label 名
menu as answer:
    "是"  -> "yes"
    "否"  -> "no"

eileen: "你选了 {answer}。"

# menu as 展开块：选项有前置逻辑时用 -> 显式 return
menu as answer:
    "答应她":
        $ log_choice("agree")
        -> "yes"        # 块内显式返回值
    "拒绝" -> "no"      # 简单场景仍用内联写法

# menu as 内不允许 jump，混用时解析器报错
# GUI 处理：menu as 对应独立的"菜单返回值"节点，与跳转型菜单节点分开

# with char：连续对话锁定角色和默认修饰符
# 块内裸字符串自动归属当前角色；行级修饰符按槽位覆盖块级默认值（表情槽、具名参数槽、Flag 槽各自独立）
with eileen (happy):
    "第一句。"
    "第二句。"
    "第三句。" (sad)          # 覆盖表情
    "第四句。" (speed=0.5)    # 只覆盖具名参数，表情继承 happy

# narrate：连续旁白块，替代重复的 @
narrate:
    "第一段。"
    "第二段。"
    "第三段。" (speed=0.5)

# 跳转与调用
jump route_a

# 条件跳转短路写法：行末 if / unless 作为守卫条件，等价于 if 块但更轻量
jump route_a if flag_agreed
jump route_b if day >= 3
call morning_scene() if day == 1
return if flag_done

# unless 对称写法（if not 的语法糖）
jump prologue unless flag_met_eileen
return unless flag_can_continue

# 不支持 call ... as result if ...（条件不满足时返回值语义不明），退回 if 块处理
# GUI 处理：条件跳转节点显示为带条件标签的跳转箭头，视觉权重轻于完整 if 块

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

`parallel` 块内的 track 分两类，用显式标记区分：

- **`(interactive)` 轨道**：允许对话行，独占用户输入，每次只能有一个。其他 track 在等待点击时继续运行。
- **普通轨道**：不允许对话行，遇到时解析器报错，不静默通过。

```apy
parallel:
    track dialogue (interactive):   # 交互轨道，可以有对话行，独占输入
        show eileen left 0.5
        eileen: "……"                 # 正常等待点击
        sophia: "是啊。"
    track bgm:                      # 非交互轨道，只允许引擎指令
        play music "bgm/tense.ogg" 0.8 1.0
        camera shake 5 0.3
```

同一 `parallel` 块只允许一个 interactive track，多个时解析器报错。

`parallel` 块内的所有 `track` 并行执行，默认等所有 `track` 完成后推进（等价于 `wait for all`）。需要提前推进时：

```apy
parallel (wait=any):
    track a:
        ...
    track b:
        ...
```

`wait=any` 下，interactive track 未完成的对话会被打断——语义合法，但编辑器给出警告。

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

`parallel` 块在 GUI 脚本区中表现为时间轴视图，每个 `track` 对应一条轨道；interactive track 以特殊标记区分。

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

**时间单位：绝对秒数**

keyframe 时间值为绝对秒数。`transform` 块内可声明 `duration` 作为默认总时长，apply 时可覆盖；省略 `duration` 时引擎取最后一个 keyframe 的时间值作为总时长。覆盖时引擎等比缩放所有 keyframe 时间点，相对节奏不变。

```apy
# 定义可复用 transform
transform shake_x:
    duration 0.4                     # 可选，省略时取最后 keyframe 时间
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
    keyframe 0.5 (easing=ease_out): # 逐段 easing，优先级高于全局声明
        alpha 0.8
        y_offset -5
        scale 1.02
    keyframe 1.0 (easing=bounce):
        alpha 1.0
        y_offset 0
        scale 1.0
    repeat 1

transform breathe:
    keyframe 0.0: scale_y 1.0
    keyframe 1.0: scale_y 1.03
    easing ease_in_out
    repeat forever pingpong          # 正反交替循环，无需手写对称 keyframe
```

keyframe 折叠规则：冒号后有内容 = 单行（单属性）；冒号后为空 = 展开块（多属性）。和 Python 自身的单行/块语法直觉一致。

**`repeat` 语法：**

```apy
repeat 1                  # 播放一次
repeat forever            # 无限循环（正向）
repeat forever pingpong   # 无限循环，正反交替
repeat 3 pingpong         # 播放 3 次，正反交替（奇数次正向，偶数次反向）
```

`repeat N pingpong` 中 N 指完整循环次数，不是单程次数。

**逐段 easing：**

`easing` 支持全局声明和逐 keyframe 声明，逐段优先级高于全局；全局未声明时默认 `linear`。

```apy
transform slide_in:
    keyframe 0.0: y_offset -50
    keyframe 0.6 (easing=ease_out): y_offset -5    # 这一段 ease_out
    keyframe 1.0 (easing=bounce):   y_offset 0     # 这一段 bounce
    repeat 1
```

**transform 可用属性：**

| 属性 | 含义 |
|------|------|
| `x_offset` / `y_offset` | 位移偏移（像素，相对初始位置） |
| `x` / `y` | 绝对位置（像素，相对锚点）；与 `x_offset` / `y_offset` 共存时取加法合成 |
| `scale` / `scale_x` / `scale_y` | 缩放比例 |
| `rotate` | 旋转角度 |
| `alpha` | 透明度（0.0–1.0） |
| `color_multiply` | 颜色乘算（闪红、变灰等），接受 `(r, g, b, a)` |
| `blur` | 高斯模糊半径（像素） |
| `anchor_x` / `anchor_y` | 锚点（0.0–1.0），影响旋转和缩放中心 |

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
show eileen center (transform=shake_x)                              # 单个
show eileen center (transform=[breathe, shake_x])                   # 多个并行叠加
show eileen center (transform=[enter_scale, hover_scale], compose=sequence)  # 串行
show eileen center (transform=shake_x(duration=0.2))               # 覆盖 duration
```

**叠加模型（`compose`）：**

| `compose` 值 | 行为 |
|---|---|
| `parallel`（默认） | 所有 transform 同时运行，属性冲突取列表最后一个，引擎启动时对已知冲突输出警告 |
| `sequence` | 串行，前一个 `repeat 1` 完成后启动下一个；遇到 `repeat forever` 则阻塞在此，之后的 transform 永远不会执行，引擎启动时输出警告 |

**触发与停止模型：**

新的 `show eileen (transform=X)` 替换当前所有 transform，不叠加。需要追加时使用 `transform+=`：

```apy
show eileen (transform=breathe)       # 启动 breathe
show eileen (transform=shake_x)       # 停止 breathe，启动 shake_x
show eileen (transform+=shake_x)      # 在现有基础上追加 shake_x
show eileen (transform=none)          # 显式停止所有 transform
```

| 事件 | 行为 |
|---|---|
| `hide eileen` | 立即停止所有附属 transform |
| `show eileen`（无 transform 参数） | 保留当前 transform，不中断（移动位置不打断循环动画） |
| `show eileen (transform=X)` | 替换全部 transform |
| `show eileen (transform+=X)` | 追加，保留现有 transform |
| `show eileen (transform=none)` | 显式停止所有 transform |
| `repeat 1` 动画自然结束 | 停止该 transform，不影响其他 |

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

提供真正的原子语义：执行前对涉及的 key 做快照，中途抛出异常时自动回滚，保证状态变更要么全部完成、要么完全不发生。块内只允许赋值语句，不允许流程控制（`if`、`for`、函数调用等）；违反时抛出解析错误。由于受限语法使涉及的 key 在编译期静态确定，快照成本极低。

**`const` 声明**

```apy
const MAX_RELATIONSHIP = 100
const ROUTES = ["a", "b", "c"]
```

引擎启动时静态求值，写入只读层（与内置符号同层），用户代码不可覆盖。右值必须是字面量或字面量组合，不允许引用 `store` 变量或调用函数。尝试在运行时覆盖 `const` 时抛出 `AxnConstError`。

**`flag` 声明块**

集中声明游戏状态变量，使引擎可在启动时建立完整变量列表，并自动纳入存档管理：

```apy
flag:
    met_eileen = False
    agreed = False
    can_refuse = True
```

`flag` 块只允许出现在文件顶层，不允许嵌套在 `label` 或其他块内。右值必须是字面量（`bool`、`int`、`str`、`None`），不允许表达式。引擎启动时静态扫描所有 `flag` 块，生成全局变量注册表；引用了未声明变量时，引擎输出警告但不阻止运行（兼容直接用 `$` 赋值的工作流）。

`flag` 块声明的变量直接写入 `store`，访问方式与普通 `store` 变量完全一致，无命名空间前缀。

有类型注解的变量在 debug 模式下触发即时类型检查：引擎通过 `Store.__setitem__` 钩子，在赋值时立即验证类型是否匹配，不匹配时抛出 `AxnTypeError` 并指明声明位置，而不是等到存档时才发现。release 模式下 `Store` 退化为普通 `dict`，零开销。

**`set` 指令（GUI 友好写法）**

专门用于修改 `flag` 块声明的变量，使 GUI 能建立声明与赋值之间的归属关系：

```apy
set met_eileen = True
set agreed = False
set can_refuse = some_func()    # 右值复杂时，GUI 降级为代码节点，但归属关系保留
```

`set` 是推荐写法，不强制要求。`$` 永远可用作退路，两者语义等价：

| | `set` | `$` |
|---|---|---|
| 作用对象 | 只能修改 `flag` 声明的变量，修改未声明变量时引擎输出警告 | 任意 Python 赋值 |
| GUI 处理 | 专用积木块，显示变量名 + 值；右值复杂时降级代码节点 | 代码节点 |
| 静态分析 | 引擎启动时检查变量是否已在 `flag` 块声明 | 不检查 |

**`checkpoint` 存档点**

在脚本中显式标记存档点，触发自动存档并在存档列表中生成可回溯节点：

```apy
label chapter2_start:
    checkpoint "第二章·清晨"
    scene bg_morning
    eileen: "新的一天。"
```

支持具名参数扩展：

```apy
checkpoint "第二章" (thumbnail=current, bgm_preview="bgm/morning.ogg")
```

GUI 脚本区对应存档点积木块，章节结构一眼可见。

**`assert` 调试断言**

开发期用于验证游戏状态，发行版自动剥离：

```apy
assert flag_met_eileen, "进入此路由前必须已见过 eileen"
assert relationship["eileen"] >= 0, f"好感度不能为负：{relationship['eileen']}"
```

语义与 Python `assert` 完全一致。引擎在 debug 模式下执行，release 模式跳过，不产生任何运行时开销。

---

#### 事件钩子

**`on` 块**

用于响应引擎事件，只允许定义在文件顶层，不允许出现在 `label` 或任何其他块内。GUI 在独立的"事件钩子"面板中展示，与脚本流程图完全分离，不影响节点连线。

```apy
# 进入指定 label 时触发
on enter morning_scene:
    play music "bgm/morning.ogg" 0.8 1.0

# 全局按键绑定
on key "escape":
    jump pause_menu
```

块内允许引擎指令和 Python 块，Python 块按常规规则降级为代码节点。

支持的事件类型：

| 事件 | 语法 | 触发时机 |
|------|------|---------|
| label 进入 | `on enter <label>` | 执行流进入指定 label 的第一行之前 |
| 按键 | `on key "<key>"` | 玩家按下对应键时 |

`on change`（响应式变量监听）因实现成本高，暂不支持，后续考虑。

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
| `transform` block | 脚本区 keyframe 时间轴编辑器，属性列表完整可解析；`compose`、`repeat pingpong`、逐段 easing 均可解析 |
| `transform+=` | 脚本区追加节点，与 `transform=` 节点同类 |
| `AnimatedSprite` Python 类 | 脚本区代码节点（与 `python:` 块同等处理） |
| `transition`（内置） | 参数选择器，下拉列表 |
| `transition`（自定义 Python 类） | 脚本区代码节点 |
| `states` 声明 | 角色定义积木块内的状态表编辑器 |
| `layers` 声明 | 角色定义积木块内的分层立绘编辑器；静态层与动态层分别呈现；`expressions` 映射为独立映射表编辑器 |
| `expression` 指令 | 脚本区表情切换积木块；`states` 模型显示表情名下拉列表；`layers` 模型显示各动态层的独立下拉列表；直接指定层时与 `expressions` 映射积木块同类 |
| `layer create (persistent)` | 层管理面板，持久层以特殊标记区分 |
| `elif` / `unless` | 条件积木块的分支节点，与 `if` / `else` 同等处理 |
| `match`（简单形式） | 脚本区多路分支节点，值 → label 完整可解析 |
| `match`（复杂形式，含表达式或 guard） | 整块降级为代码节点 |
| `menu ->` 内联跳转 | 选项积木块内联跳转字段，与展开块形式同等处理 |
| `with char` 块 | 文本区"角色段落"积木块，折叠展示 |
| `narrate` 块 | 文本区旁白段落积木块 |
| `voice` 短路径 | 对话积木块内 voice 字段，显示短路径，存储时保留短路径形式 |
| `flag` 声明块 | 脚本区变量注册表面板，完整可解析 |
| `set` 指令 | 专用积木块，显示变量名 + 值；右值复杂时降级代码节点，归属关系保留 |
| `checkpoint` | 脚本区存档点积木块 |
| `assert` | 脚本区断言积木块，显示条件 + 消息；release 模式下标记为灰色（已剥离） |
| `on enter / on key` | 编辑器独立事件钩子面板，与流程图分离展示 |
| `pause / resume music/sound/video` | 音频积木块的暂停/恢复节点，与 `play` / `stop` 同类 |
| `play video` | 文本区视频积木块；`(async)` 标记为非阻塞节点 |
| `camera follow` | 脚本区镜头跟随节点，与 `camera move` 同类 |
| `layer create/destroy/order` | 脚本区层管理节点，独立面板展示层栈结构 |
| `say` 动词 | 脚本区动态说话者节点，角色变量字段可编辑；传入静态角色名时编辑器报错提示改用 `角色:` |
| `choice` 动词 | 脚本区代码节点（选项列表为运行时数据，GUI 不解析内容） |
| `input disable` / `input enable` | 脚本区输入控制节点；块语法对应包裹节点，对称写法对应独立节点 |
| `modal show/hide` | 脚本区模态框节点；`as result` 显示返回值变量名 |
| `menu as` 返回值 | 脚本区独立"菜单返回值"节点，与跳转型菜单节点分开；`->` 右侧显示返回值表达式字段 |
| `define extends` 角色继承 | 角色定义积木块显示继承关系；子角色字段列表中继承字段以灰色标注来源 |
| `jump/call/return if/unless` 条件短路 | 脚本区带条件标签的跳转箭头节点，视觉权重轻于完整 `if` 块 |
| `track (interactive)` | 时间轴视图中以特殊标记区分交互轨道与普通轨道；普通轨道内出现对话行时编辑器报错 |

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

**位置参数**按固定顺序省略键名，用于高频场景的简洁写法；**具名参数**在括号内以 `key=value` 形式提供，用于不常用或顺序不明确的参数。两者可混用。括号内无 `=` 的裸关键字视为布尔 flag（如 `loop`、`nowait`、`async`）。

**位置参数填充规则**：位置参数必须从第一个开始**连续**提供，中间不能跳过。需要跳过任何一个位置参数时，该参数及之后的所有参数改用具名参数。

```apy
camera move 1.2 0.5          # zoom + duration，连续，OK
camera move (duration=0.5)   # 只要 duration，跳过 zoom，改具名，OK
camera move 1.2 (angle=15.0) # zoom 用位置，跳过 duration 改具名，OK
camera move _ 0.5            # 错误：不支持占位符，改用具名写法
```

**`show` 位置约束**：`show` 的位置参数只接受预定义关键字（`left`、`center`、`right` 等），数值坐标通过具名参数 `pos=` 传入，与 duration（数字类型）不产生歧义。

```apy
show eileen left 0.3                 # 关键字位置 + duration
show eileen (pos=(100, 200))         # 数值坐标，具名参数
show eileen 0.3 (pos=(100, 200))     # duration + 数值坐标
```

**子命令**用于同一动词下行为模式本质不同的场景（如 `camera move` / `camera shake` / `camera reset`，`play music` / `play sound` / `play video`）。子命令集合由引擎硬编码，不可由用户扩展，解析器行为完全可预测。判断标准：参数描述"怎么做"时用具名参数；改变"做什么"时拆为子命令。子命令不可省略。

| 指令 | 位置参数顺序 | 保留具名参数 |
|------|------------|------------|
| `show` | 角色 → 位置 → duration | `enter` `layer` `transform` `compose` |
| `hide` | 角色 → duration | `exit` |
| `scene` | 路径 → duration | `with` `keep` |
| `clear` | 角色列表（可选） | `layer` |
| `expression` | 角色 → 表情名（可选） | 动态层具名参数（`face` `brow` `outfit` 等）`transition` |
| `play music/sound/voice/ambient` | 路径 → volume → fadein → fadeout | `loop` |
| `play video` | 路径 → volume | `layer` `loop` `async` `blocking` |
| `stop music/sound/voice/ambient` | fadeout | — |
| `stop video` | fadeout | — |
| `pause music/sound/video` | fadeout | — |
| `resume music/sound/video` | fadein | — |
| `say` | 角色变量 → 文本 | 与对话行修饰符一致 |
| `choice` | 选项列表变量 | `timeout` `default` |
| `camera move` | zoom → duration → angle | `offset` `easing` |
| `camera shake` | intensity → duration | `frequency` |
| `camera reset` | duration | — |
| `camera follow` | 角色或 `none` | `lag` |
| `layer create` | 层名 | `above` `below` |
| `layer destroy` | 层名 | — |
| `layer order` | 层名序列（从下到上） | — |
| `input disable` | — | flag 列表：`skip` `rollback` `all` |
| `input enable` | — | — |
| `modal show` | UI 控件路径 | — |
| `modal hide` | — | — |
| `call animation` | 名称 | — |
| `parallel` | — | `wait` |
| `include` | 路径 | — |
| `checkpoint` | 显示名 | `thumbnail` `bgm_preview` |
| `assert` | 条件表达式 → 消息字符串 | — |
| `set` | 变量名 = 值 | — |
| `on enter` | label 名 | — |
| `on key` | 键名字符串 | — |
| `menu as` | — | — （选项内 `->` 右侧为返回值表达式） |
| `jump if/unless` | 目标 label | 条件表达式（行末 `if`/`unless` 后） |
| `call if/unless` | label 调用 | 条件表达式（行末 `if`/`unless` 后） |
| `return if/unless` | — | 条件表达式（行末 `if`/`unless` 后） |

**子命令**用于同一动词下行为模式本质不同的场景（如 `camera move` / `camera shake` / `camera reset`，`play music` / `play sound` / `play video`）。子命令集合由引擎硬编码，不可由用户扩展，解析器行为完全可预测。判断标准：参数描述"怎么做"时用具名参数；改变"做什么"时拆为子命令。子命令不可省略。

**各指令位置参数顺序**：

### 关键设计决策

**表情控制**：表情状态跟着角色对象走，不跟场景走。对话修饰符修改角色的**持久表情状态**，与角色是否可见无关；`show` 出场时使用角色当前表情状态，不重置为 `default_expression`。`show` 指令仅控制位置和可见性，不影响表情状态。无对话时切换表情使用独立的 `expression` 指令，支持可选的 `transition` 参数。

**Python 块边界**：`$` 后允许任何在单行内 Python 语法合法的内容，包括三元表达式、多重赋值等（如 `$ x = 1 if flag else 2`、`$ a, b = b, a`）。需要换行的复合逻辑使用 `python:` 块，换行符即边界，无需记额外规则。

**角色定义**：内联在 `.apy` 文件中，不使用外部 JSON。`define` 默认推断类型，`define char` 为显式声明，语义等价但意图更清晰。支持 `dyn define` 实现运行时动态定义。

**`show` 层级**：默认操作 sprite 层，需要时通过 `(layer=effect)` 等显式指定。层级扩展需求驱动，不过早设计。

**并行与串行执行**：同行逗号分隔的指令并行执行，引擎默认等所有并行动画完成后推进。需要精细控制等待时机时，用 `as` 给动画命名，再用 `wait for` 显式控制。`wait for` 中 `for` 是介词而非子命令，`wait for all` / `wait for any` / `wait for <name>` 三种形式语义链完整。

**`scene` 默认清空 sprite 层**：`scene` 切换背景时默认同时清空 sprite 层（高频用法零开销）。需要保留立绘时显式使用 `(keep)` 或 `(keep=角色名)` 具名参数；`(keep=[eileen, sophia])` 支持保留多个。`scene` 只清非持久层，持久层（如 `ui`）完全不受影响。

**`clear` 指令**：精确清除，无过渡，定位是"批量/精确移除"而非"退场"。支持指定角色（`clear eileen`）、多个角色（`clear eileen sophia`）、指定层（`clear (layer=effect)`）、指定层上的指定元素（`clear eileen (layer=effect)`）。无参数时清除 sprite 层所有元素。`clear` 不支持过渡动画，需要过渡退场时使用 `hide`。`clear` 可以显式清除持久层，但需要明确指定 `layer=`，不会误伤持久层。

**`hide` 与 `clear` 的语义区别**：`hide eileen 0.5 (exit=fadeout)` 隐藏单个角色，支持过渡动画，强调"退场"；`clear eileen` 立即移除，无过渡，强调"清除"。需要过渡时用 `hide`，需要批量或精确无过渡移除时用 `clear`。

**`call` 返回值**：只支持 `as` 写法——`call label() as result`。`_return` 作为引擎内部实现细节，不对外暴露。

**`label` 签名**：直接使用 Python 函数签名风格，支持默认值、`*args`、`**kwargs`。`return` 后跟任意 Python 表达式。

**跨文件引用**：`jump` / `call` 不需要 `import`，引擎启动时自动扫描所有 `.apy` 文件，label 冲突在启动时报错。跨文件引用 `define`（如 UI 控件）需要显式 `import`，使依赖关系可见。label 命名冲突由引擎扫描和 VSCode 插件共同处理，语言层不强制约束。

**UI 控件定义**：在独立的 `.apy` 文件中定义，通过 `文件路径::控件名` 语法引用，如 `"ui/eileen_box.apy::EileenBox"`。

**推断失败行为**：所有默认推断逻辑（层级推断、类型推断等）在推断失败时抛出明确错误，不静默走错分支。

**transform 与 animation 边界**：`transform` 描述单个对象的属性动画（keyframe 数据），`animation` 描述演出流程（指令序列）。两者职责不重叠，通过 `(transform=...)` 参数组合使用。`transform` 块内只允许有限的引擎关键字，不允许 Python，保证 GUI 可完整解析；超出表达能力的场景通过继承 `AnimatedSprite` 退到 Python。

**transform 时间单位**：keyframe 时间值为绝对秒数，与 `show`/`hide` 的 duration 单位一致。`duration` 参数可覆盖总时长，引擎等比缩放所有 keyframe 时间点，相对节奏不变。省略 `duration` 时取最后一个 keyframe 的时间值。

**transform repeat**：`repeat 1` / `repeat forever` / `repeat forever pingpong` / `repeat N pingpong`。`pingpong` 下奇数次正向、偶数次反向，N 指完整循环次数，不是单程次数。

**transform easing**：支持全局声明和逐 keyframe 声明（`keyframe T (easing=X):`），逐段优先级高于全局，全局未声明时默认 `linear`。

**transform 叠加冲突**：`compose=parallel`（默认）时属性冲突取列表最后一个，不做混合，引擎启动时输出警告。`compose=sequence` 时串行执行，`repeat forever` 之后的 transform 永远不会执行，引擎启动时输出警告。

**transform 触发与停止**：`show eileen (transform=X)` 替换全部现有 transform；`show eileen` 无 transform 参数时保留当前 transform（移动位置不打断循环动画）；`transform+=X` 追加；`transform=none` 显式停止所有；`hide` 时自动停止所有附属 transform。

**keyframe 折叠规则**：冒号后有内容 = 单行（单属性）；冒号后为空 = 展开块（多属性）。和 Python 自身的单行/块语法直觉一致，解析器无歧义。

**transition 扩展**：内置过渡由引擎标准库提供，自定义过渡继承 `Transition` 抽象类，通过 `apply(surface, progress)` 接口实现，直接传实例到具名参数，不引入新语法。

**`elif` / `unless`**：`elif` 补全条件链，语义与 Python 完全一致。`unless` 是 `if not` 的语法糖，仅用于卫语句场景（块内通常只有一条 `jump` 或 `return`），不支持 `unless ... elif ...` 链式写法，避免语义混乱。

**`match` 简单形式与复杂形式**：`match <store变量>:` + `值 -> label` 为简单形式，GUI 完整解析为多路分支节点。含表达式或 guard 的复杂形式整块降级为代码节点，不做部分解析，规则明确无歧义。

**`menu ->` 内联跳转**：选项只有一条 `jump` 时用 `->` 省略展开块；有额外逻辑时退回展开块写法。同一 `menu` 内两种形式可混用，不要求统一。

**`with char` 块**：块内裸字符串自动归属当前角色，修饰符继承块级声明。行级修饰符按**槽位覆盖**块级默认值——修饰符分为表情槽、具名参数槽、Flag 槽三类，行级只覆盖显式指定的槽位，未指定的槽位继承块级默认值。适合连续独白场景，不适合多角色交叉对话。

```apy
with eileen (happy, speed=1.0):
    "第一句。"                    # 表情=happy, speed=1.0
    "第二句。" (sad)              # 表情=sad,   speed=1.0  ← 只覆盖表情槽
    "第三句。" (speed=0.5)        # 表情=happy, speed=0.5  ← 只覆盖 speed 槽
    "第四句。" (sad, speed=0.5)   # 表情=sad,   speed=0.5  ← 两个槽都覆盖
```

**`narrate` 块**：连续旁白的语法糖，替代重复的 `@`。块内裸字符串全部作为旁白处理，支持修饰符。GUI 对应旁白段落积木块。

**`narrator` 保留关键字**：`@` 和 `narrator:` 是单行旁白的两种等价写法，`with narrator:` 与 `narrate:` 块等价。`narrator` 是引擎保留关键字，不允许用户通过 `define` 覆盖；尝试 `define char narrator` 时引擎在启动时报错。三种旁白写法（`@`、`narrator:`、`narrate:` 块）风格自选，同一项目内保持一致即可。

**`voice` 短路径**：对话修饰符中 `voice="001"` 自动展开为 `voice_prefix + "001" + 默认扩展名`（由 `define` 中的 `voice_prefix` 和可选的 `voice_ext` 字段决定）。完整路径写法永远有效，短路径是语法糖。推断失败时抛出 `AxnVoiceError`，不静默回退。

**`flag` 声明块**：只允许顶层声明，右值只允许字面量。支持可选类型注解（`name: type = value`），不写则不检查，保持向后兼容。引用未声明变量时输出警告不报错，保持与 `$` 工作流的兼容性。`flag` 声明的变量直接写入 `store`，无命名空间前缀，访问方式与普通变量完全一致。类型注解的作用：存档时做类型验证（不匹配抛 `AxnSaveError`）；GUI 变量面板按类型渲染控件（`bool` → 开关，`int`/`float` → 数字输入框，`str` → 文本输入框，`list`/`dict` → 折叠代码节点）；VSCode 插件可做悬停类型提示和赋值类型检查。

```apy
flag:
    met_eileen: bool = False   # 有类型注解，存档时验证
    day: int = 1
    player_name: str = ""
    relationship: dict = {}
    agreed = False             # 无类型注解，不检查，向后兼容
```

**`set` 指令**：推荐写法，不强制。`$` 永远可用。`set` 修改未在 `flag` 块声明的变量时，引擎输出警告（与引用未声明变量一致）。`set` 的存在价值是让 GUI 能追踪变量归属，`$` 的存在价值是不限制 Python 能力，两者定位不重叠。

**`checkpoint`**：引擎指令层语法，GUI 完整解析为存档点积木块。`thumbnail=current` 表示截取当前帧作为存档缩略图，为引擎保留关键字，不暴露为 Python 值。

**`assert`**：语义与 Python `assert` 完全一致，引擎在编译期识别并在 release 模式下剥离。右值允许任意 Python 表达式，消息部分允许 f-string。GUI 以灰色标记表示"此节点在 release 下不存在"。

**`on` 块作用域**：强制顶层定义，不允许出现在 `label` 或任何块内。GUI 在独立事件钩子面板中展示，与脚本流程图完全分离。块内 Python 代码按常规规则降级为代码节点。`on change` 暂不支持，响应式语义对 `store` 代理的实现成本与 GUI 追踪复杂度不值得现阶段引入。

**位置参数连续填充规则**：位置参数必须从第一个开始连续提供，跳过任何一个则之后全部改具名参数。不支持占位符语法（`_`）。规则全局统一，适用于所有指令，用户学一条规则即可推导所有指令行为。

**`show` 位置与坐标扩展**：`show` 的位置参数只接受预定义关键字（`left`、`center`、`right` 等），数值坐标通过具名参数 `pos=(x, y)` 传入。两者类型不同（关键字 vs 数字），解析器无歧义。此约束显式锁定，不允许在位置参数位置传入数值坐标。

**`pause` / `resume`**：独立动词，不是 `play` / `stop` 的子命令。语义区别：`stop` 停止并丢弃进度，`pause` 保留进度暂停，`resume` 从保留位置恢复。适用于 `music`、`sound`、`video`，子命令语法与 `play` / `stop` 对称。`pause` 接受 `fadeout` 位置参数（画面/音量渐暗），`resume` 接受 `fadein` 位置参数。

**`play video` 默认阻塞**：与 `play music`（默认非阻塞）相反，`play video` 默认阻塞执行流，播完后才推进。非阻塞时显式加 `(async)`。理由：视频大多数时候是过场动画，播完才推进是高频用法；背景循环视频是少数场景，需要显式声明意图。`(blocking)` 关键字保留但冗余，不推荐写。

**`say` 动词**：专用于说话者在运行时动态决定的场景。静态说话者必须使用 `角色:` 或 `@`，`say` 传入静态角色名（编译期可确定的标识符）时报错，不允许作为 `角色:` 的等价写法。此限制保证代码风格统一，消除"两种写法都能用"带来的歧义。修饰符与对话行修饰符完全一致。

**`choice` 动词**：`menu` 是静态声明语义，选项在编译期确定，GUI 完整解析为菜单节点。`choice` 专门处理动态场景，接受运行时生成的选项列表（`list[dict]`），整体作为代码节点处理，GUI 不尝试解析列表内容。两者定位不重叠，`choice` 不是 `menu` 的超集。

**`input disable` 两种形式**：对称写法（`input disable` / `input enable`）和块语法（`input disable: ...`）均支持，两者语义等价。块语法是推荐写法——引擎保证块结束后自动恢复，即使块内发生 `jump` 或异常也能正确还原输入状态，无需手动配对 `enable`。对称写法保留，适合跨 label 的长期禁用场景。`input disable` 支持细粒度 flag 列表（`skip`、`rollback`、`all`），无参数时等价于 `(all)`。

**`modal` 焦点模型**：`modal show` 激活时，引擎自动屏蔽底层场景输入（等价于 `input disable (all)`）、将焦点锁定到模态框内控件，模态框关闭时自动恢复，无需手动管理。`modal show ... as result` 阻塞执行流，等用户在模态框内做出选择后将结果写入 `result` 变量再推进。`modal` 与 `input disable` 的区别：`input disable` 用于演出期间屏蔽输入（无返回值、不转移焦点），`modal` 用于 UI 交互等待用户选择（有返回值、转移焦点）。

**`camera follow`**：`camera` 的新子命令，与 `move` / `shake` / `reset` 平级。`camera follow none` 取消跟随，恢复静止镜头。`lag` 参数控制跟随延迟（秒），值越大镜头越"懒"，0 为即时跟随。`camera follow` 与 `camera move` 可共存——`follow` 设定跟随目标，`move` 在此基础上叠加偏移。

**`layer` 管理**：`layer` 作为独立动词，子命令 `create` / `destroy` / `order`。`create` 的 `above` / `below` 具名参数指定新层相对于已有层的位置；`order` 接受层名序列（从下到上），一次性重排所有层顺序。层的创建和销毁在引擎启动时静态检查，运行时销毁非空层时输出警告。

**持久层**：`layer create` 支持 `persistent` flag，声明为持久层的层不受 `scene` 的默认清除影响。内置层中 `ui` 默认持久，`bg` / `sprite` / `effect` 默认非持久。`clear (layer=ui_hud)` 可以显式清除持久层，但 `clear`（无 `layer` 参数）只清非持久层，不误伤持久层。

**分层立绘**：角色立绘支持两种模型，二选一，同一 `define` 块内不可混用，混用时引擎启动报错。`states` 模型：整图切换，每个状态对应一张完整立绘图片。`layers` 模型：多图层叠加，静态层（单文件）不参与状态切换，动态层（有子状态列表）通过修饰符切换；`expressions` 映射将一个修饰符名映射到多个动态层的组合状态，`expressions` 映射必须覆盖所有动态层，漏写时引擎启动报错。图层叠加顺序：要么全部走声明顺序，要么全部写 `z_order`，不允许混用，混用时引擎启动报错。

**`expression` 指令**：无对话时切换表情的专用指令，`show` 不承担此职责。`states` 模型下 `expression eileen happy` 整图切换；`layers` 模型下走 `expressions` 映射。`layers` 模型支持直接指定各层（`expression eileen (face=happy, brow=angry)`）绕过映射，也支持换装（`expression eileen (outfit=casual)`）。可选 `transition` 具名参数控制过渡效果。两套模型下用户侧语法完全一致，差异由引擎内部按角色声明类型分派。

**`menu as` 返回值**：`menu as result` 选完后继续当前执行流，选项 `->` 右侧为返回值表达式而非 label 名。`menu as` 内不允许 `jump`，混用时解析器报错。需要前置逻辑时用展开块 + 显式 `->` 返回。GUI 对应独立的"菜单返回值"节点，与跳转型菜单节点分开，不混用。

**`define extends` 角色继承**：子角色继承父角色所有字段，显式声明的字段覆盖父定义。`layers` 模型下同名动态层内按 key 合并，未声明状态继承父定义。只允许单继承，不支持链式，避免字段来源追踪困难。继承只发生在编译期展开，运行时两个角色是完全独立的显示对象。

**条件跳转短路写法**：`jump`/`call`/`return` 行末可接 `if`/`unless` 条件，条件表达式为完整 Python 表达式。不支持 `call ... as result if ...`（条件不满足时返回值语义不明），退回 `if` 块处理。GUI 对应带条件标签的跳转箭头节点，视觉权重轻于完整 `if` 块，与 `unless` 卫语句设计意图一致。

**`parallel` 交互轨道模型**：`track (interactive)` 显式标记允许对话行的交互轨道，独占用户输入，每个 `parallel` 块只允许一个。普通轨道不允许对话行，遇到时解析器报错。`wait=any` 下 interactive track 未完成的对话被打断是合法行为，但编辑器给出警告。此设计消除了对话行与并行执行之间的交互模型歧义。

**`with store` 真正的原子语义**：执行前对涉及 key 做快照，异常时自动回滚。块内只允许赋值语句使得涉及 key 在编译期静态确定，快照成本极低。"原子性"是有实现保证的语义，不是注释。

**`flag` debug 模式类型检查**：有类型注解的变量在 debug 模式下通过 `Store.__setitem__` 钩子即时验证类型，错误信息包含声明位置。release 模式下 `Store` 退化为普通 `dict`，零开销。避免类型错误拖到存档时才暴露。

**`menu` 的 `default` 参数**：使用选项 `id` 而非选项文本，避免多语言环境下文本匹配失败。选项通过可选的 `(id="...")` 声明标识符；未声明 `id` 时以选项文本作为 fallback，引擎启动时输出警告提示多语言风险。

**`on key` 组合键**：字符串格式为 `"修饰键+key"`，修饰键小写，顺序固定为 `ctrl → shift → alt → key`，用 `+` 连接（如 `"ctrl+s"`、`"shift+f5"`）。解析器在启动时验证格式合法性。

**`camera reset` 清除 follow 状态**：`camera reset` 同时清除 follow 状态，恢复静止镜头。reset 后需要继续 follow 须显式重新声明，避免隐式状态残留。

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

## UI 系统

### 三套子系统

Axn-Plus 的 UI 系统按后端分为两条路线，Pygame 后端下细分为两个层级：

| 关键字 | 后端 | 定位 | 目标用户 |
|--------|------|------|----------|
| `screen` | Pygame | 顶层窗口容器，定义完整 UI 画面 | 轻量项目，Ren'Py 迁移用户 |
| `gui` | Pygame | 精细控件，可复用，可定义模板 | 需要复杂控件或自定义绘制的项目 |
| `window` | Qt | 声明式控件体系，包一层抽象 | 需要原生级复杂 UI 的项目 |

**后端在项目初始化时选定，之后固定。** Pygame 项目使用 `screen` + `gui`，Qt 项目使用 `window`，两套路线不混用。

---

### `screen`（Pygame 顶层容器）

`screen` 定义一个完整的 UI 画面，对标 Ren'Py 的 screen 语法，参考其设计但不照搬。绝对定位完全可用，同时扩展了相对定位能力。

```apy
screen hud:
    pin top_right:
        text "00:30"
    pin bottom center:
        dialogue_box
```

---

### `gui`（Pygame 精细控件）

`gui` 定义可复用的控件组件，默认在 `ui` 层，可通过 `layer=` 指定其他层。层级决定持久性——跟层走，不跟控件走。

```apy
gui health_bar(value, max_value, color=#ff0000):
    size (200, 20)
    background #333333
    fill:
        width value / max_value * 200
        color color
    text "{value}/{max_value}":
        anchor right
        font_size 12
```

**`gui` 可以放在非 `ui` 层：**

```apy
gui effect_overlay:
    layer effect        # 跟随场景，scene切换时清除

gui hud:                # 默认ui层，持久
    ...
```

**`screen` 和 `gui` 可以互相嵌套：**

```apy
screen hud:
    health_bar(80, 100)             # 在screen内调用gui控件
    health_bar(60, 100, color=#0000ff)
```

#### 控件定义语法

直接使用函数签名风格，和 `label` 保持一致：

```apy
gui base_button(label, width=120):
    size (width, 40)
    background #444444
    text label:
        anchor center

gui confirm_button(label) extends base_button:
    background #226622    # 覆盖父控件属性
    # size 和 text 继承 base_button
```

#### Python 逃逸

`gui` 块内可直接写 Python/Pygame 代码，Python 块拿到的是控件自己的局部 surface，坐标从 `(0, 0)` 开始：

```apy
gui custom_widget(color):
    size (100, 100)
    background #333333
    python:
        pygame.draw.circle(surface, color, (50, 50), 30)
        pygame.draw.line(surface, (255, 255, 255), (0, 0), (100, 100), 2)
```

---

### 布局能力

绝对定位（Ren'Py 风格）完全可用。在此基础上扩展了语义化相对定位：

#### 堆叠（`stack`）

```apy
stack vertical gap=8:
    button "选项一"
    button "选项二"

stack horizontal gap=4 wrap:
    tag "标签一"
    tag "标签二"
    tag "标签三"
```

#### 锚定（`pin`）

```apy
gui hud:
    pin top_right:
        text "00:30"
    pin bottom center:
        dialogue_box
    pin (0.3, 0.7):
        inventory_icon
```

#### 跟随（`follow`）

```apy
gui tooltip(text):
    follow mouse offset=(10, 10):
        panel:
            text text

gui damage_number(value):
    follow eileen offset=(0, -20):
        text value
```

#### 填充（`grow`）

```apy
gui sidebar:
    size (200, fill)

    stack vertical:
        header_area
        content_area grow:
            ...
        footer_area
```

#### 分割（`split`）

```apy
gui layout:
    split horizontal ratio=(1, 2, 1):
        left_panel
        main_panel
        right_panel
```

#### 尺寸约束

```apy
gui dialog_box:
    size (clamp(300, auto, 600), auto)
    size (fill, auto)
    min_size (200, 100)
    max_size (600, none)
```

---

### 扩展特性

以下特性是对 Ren'Py screen 系统的设计补全，覆盖 Ren'Py 的核心缺失：

#### 1. 控件局部状态

控件内部状态不污染全局 `store`：

```apy
gui collapsible(title):
    state expanded = False

    button title:
        on_click: expanded = not expanded

    if expanded:
        stack vertical:
            slot children
```

#### 2. 控件间事件系统

控件之间通过 `emit` / `on_event` 通信，与顶层 `on enter` / `on key` 钩子命名不冲突：

```apy
gui tab_bar(tabs):
    state active = tabs[0]

    stack horizontal:
        for tab in tabs:
            button tab:
                on_click: emit("tab_changed", tab)

gui tab_content:
    on_event "tab_changed": (tab):
        show_content(tab)
```

事件作用域：`emit` 向父容器冒泡，不广播到全局。需要跨控件树通信时退回 `store`。

#### 3. 控件过渡动画

复用现有 `transform` 语法，不引入新机制：

```apy
gui notification(text):
    appear: transform slide_in_top 0.3
    disappear: transform fade_out 0.2

    panel:
        text text
```

#### 4. 响应式尺寸

见布局能力中的尺寸约束部分（`clamp`、`fill`、`auto`）。

#### 5. children 插槽

```apy
gui card(title):
    panel:
        text title:
            font_size 18
        divider
        slot children

# 使用时
card("今日任务"):
    text "完成剧情A"
    text "解锁CG"
    button "查看详情"
```

#### 6. 焦点管理

支持键盘导航和手柄支持：

```apy
gui menu_panel:
    focus_group "main_menu"
    focus_default confirm_btn
    focus_order (option_a, option_b, option_c, confirm_btn)

    button "确认" as confirm_btn:
        on_focus: ...
        on_blur: ...
```

#### 7. 条件样式

根据状态动态改变样式，不需要 if/else 切换整个控件：

```apy
gui option_button(label, selected):
    background #444444
    when selected:
        background #226622
        border (2, #ffffff)
    when hovered:
        background #555555
    when focused:
        border (2, #aaaaff)
```

#### 8. 滚动容器

```apy
gui scroll_panel:
    scroll vertical:
        momentum True
        scrollbar (width=4, color=#ffffff44)
        overscroll bounce
        stack vertical gap=8:
            ...
```

#### 9. 控件级 z_order

控件内部的层叠顺序，与场景层系统不冲突：

```apy
gui dropdown(options):
    button "选择":
        on_click: toggle expanded

    if expanded:
        stack vertical:
            z_order 100
            for opt in options:
                button opt
```

#### 10. overflow 与裁剪控制

```apy
gui avatar_frame:
    size (64, 64)
    overflow clip
    border_radius 32
    image portrait

gui text_preview:
    size (300, 60)
    overflow hidden
    text long_content

gui scroll_area:
    size (300, 200)
    overflow scroll
    text very_long_content
```

`overflow` 三种值：

| 值 | 行为 |
|---|---|
| `clip` | 裁剪超出部分，支持 `border_radius` 定义裁剪形状 |
| `hidden` | 隐藏溢出内容，无滚动条 |
| `scroll` | 溢出时自动出现滚动条 |

---

### `window`（Qt 后端）

`window` 使用声明式语法，包一层抽象屏蔽 Qt 概念，足够简单。复用 `template` / `extends` 进行组件继承：

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

`window` 的 `template` / `extends` 与 `gui` 的继承语义不同：

| | `gui extends` | `window template/extends` |
|---|---|---|
| 后端 | Pygame | Qt |
| 参数化 | 支持，函数签名风格 | 不支持 |
| 继承 | 属性覆盖 + 参数继承 | 属性覆盖 |
| 实例化 | 直接调用 `health_bar(80, 100)` | 引用路径 `"ui/x.apy::X"` |

---

### 关键设计决策（UI 系统）

**后端绑定**：后端在项目初始化时选定，之后固定。Pygame 项目使用 `screen` + `gui`，Qt 项目使用 `window`，不混用。

**`screen` 与 `gui` 的职责边界**：`screen` 是顶层画面容器，层级固定；`gui` 是控件，可放在任意层。两者可互相嵌套，混用合法。

**持久性跟层走**：`gui` 控件放在哪一层，就遵守那一层的持久性规则。`clear` 不清除 `ui` 层（持久），但会清除 `effect` / `sprite` 层上的 `gui` 控件。需要持久但不在 `ui` 层时，显式声明 `persistent`。

**Python 逃逸的 surface 作用域**：`gui` 块内 Python 块拿到的是控件局部 surface，坐标从 `(0, 0)` 开始，引擎负责合成到正确位置。保证控件封装性，不暴露全局 surface。

**事件作用域**：`emit` 向父容器冒泡，不广播到全局。`on_event` 与顶层 `on enter` / `on key` 使用不同关键字，无命名冲突。

**`when` 条件样式**：只允许引擎预定义的状态关键字（`selected`、`hovered`、`focused`、`disabled`、`pressed`），不允许任意 Python 表达式，保证 GUI 编辑器可完整解析。复杂条件样式退回 `if` 块或 Python。

**布局关键字语义化**：关键字描述意图（`pin`、`stack`、`grow`、`split`），不描述实现，不照搬前端术语。绝对定位（Ren'Py 风格）永远可用作退路。

---

### Round-Trip Fidelity 补充（UI 系统）

| 新增语法 | GUI 处理方式 |
|----------|-------------|
| `screen` | 编辑器顶层画面容器节点 |
| `gui` 定义 | 编辑器控件模板库，参数列表完整可解析 |
| `gui extends` | 控件继承树，继承字段以灰色标注来源 |
| `stack` / `pin` / `follow` / `split` | 布局积木块，对应可视化布局编辑器 |
| `grow` / `fill` / `auto` / `clamp` | 尺寸约束字段，编辑器内联编辑 |
| `state` 局部状态 | 控件节点内的状态变量面板 |
| `emit` / `on_event` | 事件连线，编辑器以箭头表示事件流向 |
| `appear` / `disappear` | 控件过渡动画字段，复用 transform 编辑器 |
| `slot children` | 编辑器显示插槽占位符，调用方可拖入子控件 |
| `focus_group` / `focus_order` | 焦点管理面板，独立于布局展示 |
| `when` 条件样式 | 样式编辑器内的条件分支，与普通样式字段并列 |
| `scroll` 容器 | 滚动容器积木块，scrollbar 样式独立编辑 |
| `z_order` | 控件层叠顺序字段 |
| `overflow` + `border_radius` | 溢出控制字段，clip 模式下裁剪形状可视化编辑 |
| `window` / `template` / `extends` | 窗口区组件继承树（与 Pygame 路线完全分离） |

---

## 不是什么

- 不是 Ren'Py 的分支或 fork
- 不是面向零编程基础用户的工具（引擎本身）
- 不追求最大化跨平台覆盖
