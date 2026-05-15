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

**三遍扫描**：解析器分三遍处理所有 `.apy` 文件，均在引擎启动时一次性完成，不在运行时重复。

- **第一遍**：轻量扫描，只收集所有 `define` 的名字，不解析字段内容和 `extends` 关系。目标是建立全局名字集合，使第二遍能识别跨文件引用。
- **第二遍**：扫描所有 `define extends` 关系，建立继承有向图，拓扑排序确定解析顺序。检测循环继承并在此阶段报错（`AxnParseError: circular inheritance`）。同时建立全局符号表（角色名 → 类型），供第三遍使用。
- **第三遍**：按拓扑顺序完整解析所有文件，展开继承字段，构建完整 AST。行首遇到已知角色名走对话路径，否则走指令路径。label 冲突检查也在此阶段完成。

`define` 只允许出现在文件顶层，保证第一、二遍扫描无需理解嵌套结构，各遍成本极低。第一、二遍合计开销相对于第三遍可忽略不计；瓶颈始终在第三遍的 AST 构建。对未修改的文件可跳过第三遍，直接使用缓存的 AST，进一步降低重启开销。

**为什么需要三遍而不是两遍**：引入 `define extends` 后，跨文件的继承依赖关系在解析前无法确定顺序——父类定义必须先于子类解析，但依赖关系本身要解析才能知道。三遍扫描将"收集名字"、"建立依赖图"、"完整解析"三个阶段显式分离，避免了解析顺序的不确定性。Ren'Py 通过不支持 `define` 继承来回避这个问题，Axn-Plus 选择支持继承，对应的代价是多一遍轻量扫描。

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
# - 运行时修改 eileen 的层状态（表情、换装等），eileen_adult 完全不受影响，反之亦然
# - layers 模型下的 key 合并也发生在编译期，运行时两个角色的层状态互不共享

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
menu (timeout=10.0, default="refuse"):
    "答应她" (id="agree"):
        $ flag_agreed = True
        jump route_a
    "拒绝" (id="refuse", if=flag_can_refuse):
        jump route_b
    "询问详情" (id="ask", if=flag_met_eileen, disabled=flag_tired):
        jump route_c
    "隐藏选项" (id="secret", hidden=flag_secret):
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

`as anim` 得到的是 `AnimationHandle` 对象，只暴露以下接口：

```python
anim.done        # bool，动画是否完成
anim.cancel()    # 立即停止动画
```

`wait for anim` 是引擎层语法糖，底层轮询 `anim.done`。不允许在 Python 块里直接操作 `AnimationHandle` 对象——此限制保证 GUI 能完整解析 `wait for` 的依赖关系。

`animation` 块内只允许引擎指令（`show`、`hide`、`camera`、`play`、`wait` 等），不允许 Python 块、对话行、`jump`、`menu`。目的是保证 GUI 能完整解析，也防止演出片段携带业务逻辑。

**`animation` 参数化**

`animation` 块支持函数签名风格参数，和 `label` 保持一致：

```apy
animation char_enter(char, position="center", duration=0.3):
    show char position 0.0
    camera move 1.1 0.5 (easing=ease_out)
    wait for all

animation char_exit(char, duration=0.3):
    hide char duration (exit=fadeout)
    wait for all
```

调用时传入参数：

```apy
call animation char_enter(eileen, "left")
call animation char_enter(eileen, "left") as anim
call animation char_enter(eileen)          # 使用默认参数
```

参数类型限制：只允许角色名、位置关键字、数值、字符串字面量，不允许传入 Python 表达式——`animation` 块内不允许 Python，参数也不应该是 Python 表达式，否则 GUI 无法解析调用点。需要动态参数时退回 `label` + Python 块处理。

**`show` 不阻塞执行流**

`show eileen center (transform=shake_x)` 之后立即推进到下一行，transform 在后台运行，对话行不等 transform 完成。需要等待时显式使用 `as` + `wait for`：

```apy
show eileen center (transform=shake_x) as anim_shake   # 单行 show + as，合法
wait for anim_shake
eileen: "你好。"    # shake_x 完成后才显示对话
```

不等待时 transform 跑着，对话同时出现，两者互不阻塞。`as` 在单行 `show` 上与并行写法上均有效。

**前台动画与后台动画**

`transform` 按 `repeat` 类型自动区分前台/后台身份，不需要用户额外声明：

- **前台动画**：`repeat 1` / `repeat N`，参与 `wait for all` 的等待判断
- **后台动画**：`repeat forever` / `repeat forever pingpong`，不参与 `wait for all`，持续运行直到对象被 `hide` 或显式 `transform=none`

```apy
animation eileen_enter:
    show eileen center (transform=[complex_enter, breathe])
    # complex_enter: repeat 1       → 前台，参与等待
    # breathe:       repeat forever → 后台，不参与等待
    wait for all    # 等 complex_enter 完成即推进，breathe 继续跑
```

边界情况：`wait for all` 时若所有 transform 均为后台动画（无前台动画），立即满足，debug 模式输出警告：

```
AxnWarning: 'wait for all' has no finite transforms to wait for.
  All transforms are 'repeat forever'. Did you mean to use 'wait'?
  Line 3, eileen_enter animation block.
```

**`transform` 归属对象，不归属 track**

`scheduler.py` 维护全局 transform 注册表，key 为显示对象 id，value 为当前运行的 transform 列表。`parallel` track 的调度与 transform 调度完全独立，互不感知。track 结束时不自动停止该 track 内 `show` 触发的 transform——transform 的生命周期跟对象走，不跟 track 走，`hide` 对象时才停止其所有附属 transform。此规则在 `parallel` 场景下与单轨道场景完全一致。

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

`wait=any` 与 interactive track 共存时，解析器直接报错，不允许此组合。理由：interactive track 正在等用户点击时，"最先完成的 track"触发推进，点击事件时机不可预测，会产生误触或输入状态污染。需要提前推进的场景，改用 `wait=none` + 手动 `wait for` 精细控制。

`track` 可命名后用 `wait for` 精细控制：

```apy
parallel (wait=none):           # 不自动等待，手动控制
    track dialogue (interactive):
        eileen: "第一句"
        eileen: "第二句"
    track bgm:
        play music "bgm/tense.ogg" 0.8 1.0
    track scene as anim_scene:
        show eileen left 0.5
        camera shake 5 0.3

wait for anim_scene             # 等 scene track 完成后推进
wait for dialogue               # 等 interactive track 完成后推进
```

**`wait=none` + interactive track 的输入路由规则**：`wait for <track>` 等待期间，interactive track 的输入路由规则与 `parallel` 块内完全一致——interactive track 独占用户输入，点击推进的是 track 内部当前等待的对话行，不影响外部 `wait for` 的等待状态。`wait for` 只观察 track 的完成信号（`track.done`），不接管输入。用户点击推进对话 → interactive track 内部状态前进 → track 执行完毕 → `wait for dialogue` 自然满足，执行流继续。外部 `wait for` 是纯被动观察者，不与输入系统交互。

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
    relationship = new_rel      # ✅ 整体替换顶层变量
    # relationship["eileen"] += 5  ← 解析期报错，改用 python: 块先算好再赋值
```

提供真正的原子语义：执行前对所有涉及的顶层 key 做快照，中途抛出异常时自动回滚，保证状态变更要么全部完成、要么完全不发生。

**原子性边界**：`with store` 只保证顶层 `store` 变量的原子性。块内只允许顶层变量的赋值语句（`x = ...`、`x += ...`），不允许下标访问（`dict["key"]`）、属性访问（`obj.attr`）、方法调用（`list.append(...)`）或任何流程控制。违反时解析期报错：

```
AxnParseError: 'with store' only allows top-level store assignments.
  Use a 'python:' block for nested mutations, then assign the result.
  12 | relationship["eileen"] += 5
```

需要批量修改 `dict` 子项时，先在 `python:` 块里算好新值，再用 `with store` 整体赋值：

```apy
python:
    new_rel = dict(relationship)
    new_rel["eileen"] += 5

with store:
    relationship = new_rel
    day += 1
```

由于块内只允许顶层赋值，涉及的 key 在编译期静态确定，快照成本极低。

**快照的浅拷贝语义**：快照保存的是 `store` 顶层 key 指向的对象引用，而非深拷贝。回滚时，引擎将这些 key 指回快照保存的旧引用——如果旧对象在块执行期间被外部代码（如通过 Python 直接持有引用并原地修改）改动，回滚无法恢复旧对象的内容。在纯 `.apy` 工作流下此场景不会发生。**回滚保证的语义是：`store` 顶层 key 指向执行前的对象，不保证该对象内容的深度一致性。** 需要深度一致性时，在 `python:` 块里手动构造新对象（如 `new_rel = dict(relationship)`），再通过 `with store` 整体替换顶层 key。

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
| `with store` 块 | 脚本区代码节点（与 `python:` 块同等处理）；块内只允许顶层赋值，违反时编辑器解析期报错 |
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
| `checkpoint` | 脚本区存档点积木块；存档在指令执行后、下一行执行前触发，记录执行位置而非状态快照 |
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

静态 label 与动态 label 的具体区别：

| | 动态 label（默认） | 静态 label（`sta`） |
|---|---|---|
| `dyn define` 角色引用 | 允许 | 编译期报错 |
| `$` / `python:` 块 | 允许 | 编译期报错 |
| GUI 解析 | 可能含代码节点 | 保证完整解析为积木块，无代码节点 |
| 引擎优化 | 无额外优化 | 编译期预处理，跳转目标缓存 |

`sta` 的核心价值：一是给 GUI 一个保证——此 label 无代码节点，可完整可视化；二是代码审查信号——作者明确声明此处不应有动态行为，后续若有人加入 `$` 块，编译器报错而不是静默通过。`sta` 不是性能优化手段。

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

**`voice` 短路径**：对话修饰符中 `voice="001"` 自动展开为 `voice_prefix + "001" + 扩展名`。扩展名推断规则：若 `define` 中声明了 `voice_ext` 字段则直接使用；未声明时引擎按 `.ogg` → `.mp3` → `.wav` 优先级扫描，找到第一个存在的文件即用，全部找不到时抛出 `AxnVoiceError`。完整路径写法永远有效，短路径是语法糖。引擎构建发布包时完整打包 `voice_prefix` 目录，扫描行为在发布包里与开发期一致。

```apy
define eileen:
    voice_prefix "vo/eileen/"
    voice_ext ".ogg"        # 可选；显式指定跳过扫描，性能更好；不填时按优先级自动推断
```

**`flag` 声明块**：只允许顶层声明，右值只允许字面量。支持可选类型注解（`name: type = value`），不写则不检查，保持向后兼容。引用未声明变量时输出警告不报错，保持与 `$` 工作流的兼容性。`flag` 声明的变量直接写入 `store`，无命名空间前缀，访问方式与普通变量完全一致。类型注解的作用：存档时做类型验证（不匹配抛 `AxnSaveError`）；GUI 变量面板按类型渲染控件（`bool` → 开关，`int`/`float` → 数字输入框，`str` → 文本输入框，`list`/`dict` → 折叠代码节点）；VSCode 插件可做悬停类型提示和赋值类型检查。

**类型注解验证边界**：`dict` / `list` 类型注解只验证顶层类型（`isinstance` 检查），不验证内部结构。例如 `relationship: dict = {}` 只保证 `relationship` 是一个 `dict`，不保证其 key/value 的类型。需要结构验证时，继承 `Saveable` 并在 `__load__` 里手动校验，不引入额外语法。

```apy
flag:
    met_eileen: bool = False   # 有类型注解，存档时验证
    day: int = 1
    player_name: str = ""
    relationship: dict = {}
    agreed = False             # 无类型注解，不检查，向后兼容
```

**`set` 指令**：推荐写法，不强制。`$` 永远可用。`set` 修改未在 `flag` 块声明的变量时，引擎输出警告（与引用未声明变量一致）。`set` 的存在价值是让 GUI 能追踪变量归属，`$` 的存在价值是不限制 Python 能力，两者定位不重叠。

**`checkpoint`**：引擎指令层语法，GUI 完整解析为存档点积木块。`thumbnail=current` 表示截取当前帧作为存档缩略图，为引擎保留关键字，不暴露为 Python 值。存档时 call 栈被丢弃，读档后以 checkpoint 下一行作为新的顶层执行起点。`checkpoint` 出现在被 `call` 的子 label 里时引擎给出编译期警告，建议移至顶层 label 入口处。

**`checkpoint` 存档时机**：存档在 `checkpoint` 指令本身执行完毕后、下一行执行前触发。存档记录的是"执行位置"（即 `checkpoint` 之后的下一行），而不是某个状态快照。读档后从 `checkpoint` 的下一行开始重新执行，`checkpoint` 之后的赋值语句会在读档后重新运行，不会丢失。

**`checkpoint` 与 `call` 栈**：`checkpoint` 存档时 call 栈被丢弃，读档后以 checkpoint 下一行作为新的顶层执行起点，不尝试恢复调用关系。因此强烈建议将 `checkpoint` 放在顶层 label 入口处，不要放在被 `call` 的子 label 里。引擎对后者给出编译期警告：

```
AxnWarning: 'checkpoint' inside a called label discards the call stack on load.
  Consider moving 'checkpoint' to the top-level call site instead.
  Line 3, chapter2_start.apy
```

此模型与 Ren'Py 一致。开发者需注意：`checkpoint` 之前的副作用（如外部 API 调用）在读档后不会重新执行，有副作用的操作应放在 `checkpoint` 之后。

**`assert`**：语义与 Python `assert` 完全一致，引擎在编译期识别并在 release 模式下剥离。右值允许任意 Python 表达式，消息部分允许 f-string。GUI 以灰色标记表示"此节点在 release 下不存在"。

**`on` 块作用域**：强制顶层定义，不允许出现在 `label` 或任何块内。GUI 在独立事件钩子面板中展示，与脚本流程图完全分离。块内 Python 代码按常规规则降级为代码节点。`on change` 暂不支持，响应式语义对 `store` 代理的实现成本与 GUI 追踪复杂度不值得现阶段引入。

**`on key` 与 `input disable` 的优先级**：`input disable` 生效期间，所有 `on key` 绑定被屏蔽，无例外。`input disable` 的 flag 列表控制屏蔽粒度——需要保留部分按键响应时，通过 flag 精确指定：

```apy
# 只禁用跳过和回滚，escape 仍然触发 on key 绑定
input disable (skip, rollback):
    play video "cutscene/intro.mp4"
    wait for video

# 全部禁用（含 on key），过场动画期间无法跳过
input disable:
    play video "cutscene/intro.mp4"
    wait for video
```

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

**`parallel` 交互轨道模型**：`track (interactive)` 显式标记允许对话行的交互轨道，独占用户输入，每个 `parallel` 块只允许一个。普通轨道不允许对话行，遇到时解析器报错。`wait=any` 与 interactive track 共存时解析器直接报错——interactive track 等待用户点击期间，`wait=any` 触发推进的时机不可预测，会产生输入状态污染，此组合没有合理的使用场景。需要提前推进时改用 `wait=none` + 手动 `wait for`。`wait=none` 下使用 `wait for <interactive_track>` 时，输入路由规则不变：interactive track 仍然独占用户输入，`wait for` 是纯被动观察者，只轮询 `track.done`，不接管输入；用户点击推进对话 → track 内部前进 → track 完成 → `wait for` 自然满足。此设计消除了对话行与并行执行之间的交互模型歧义。

**`with store` 真正的原子语义**：只允许顶层 `store` 变量的赋值语句（`x = ...`、`x += ...`），不允许下标访问、属性访问、方法调用或任何流程控制。违反时解析期报错，不静默通过。原子性边界明确：快照和回滚只针对顶层 key，`dict`/`list` 子项的内部修改不在保证范围内——需要修改子项时，先在 `python:` 块里构造好新值，再用 `with store` 整体赋值。由于块内只允许顶层赋值，涉及的 key 在编译期静态确定，快照成本极低，"原子性"是有实现保证的语义，不是注释。快照保存的是对象引用而非深拷贝——回滚保证 `store` 顶层 key 指向执行前的对象，不保证该对象内容的深度一致性；在纯 `.apy` 工作流下此边界不可见，通过 Python 直接持有 `store` 变量引用并原地修改时需开发者自行注意。外部引用检测不做运行时实现（`sys.getrefcount()` 不可靠，`gc.get_referrers()` 有 GC 暂停开销），此边界通过文档说明清楚即可。

**`flag` debug 模式类型检查**：有类型注解的变量在 debug 模式下通过 `Store.__setitem__` 钩子即时验证类型，错误信息包含声明位置。release 模式下 `Store` 退化为普通 `dict`，零开销。避免类型错误拖到存档时才暴露。

**`menu` 的 `default` 参数**：使用选项 `id` 而非选项文本，避免多语言环境下文本匹配失败。选项通过可选的 `(id="...")` 声明标识符；未声明 `id` 时以选项文本作为 fallback，引擎启动时输出警告提示多语言风险。

**`on key` 组合键**：字符串格式为 `"修饰键+key"`，修饰键小写，顺序固定为 `ctrl → shift → alt → key`，用 `+` 连接（如 `"ctrl+s"`、`"shift+f5"`）。解析器在启动时验证格式合法性。

**`camera reset` 清除 follow 状态**：`camera reset` 同时清除 follow 状态，恢复静止镜头。reset 后需要继续 follow 须显式重新声明，避免隐式状态残留。

**`animation` 参数化**：`animation` 块支持函数签名风格参数，和 `label` 保持一致。参数类型限制为角色名、位置关键字、数值、字符串字面量，不允许 Python 表达式——保证 GUI 能完整解析调用点。需要动态参数时退回 `label` + Python 块处理。

**`show` 永不阻塞执行流**：`show` 指令（含 `transform` 参数）之后立即推进到下一行，transform 在后台运行。需要等待时显式使用 `as` + `wait for`。`as` 在单行 `show` 和并行写法上均有效。

**前台动画与后台动画**：`transform` 按 `repeat` 类型自动区分身份，不需要用户额外声明。`repeat 1` / `repeat N` 为前台动画，参与 `wait for all`；`repeat forever` / `repeat forever pingpong` 为后台动画，不参与 `wait for all`，持续运行直到对象被 `hide` 或 `transform=none`。`wait for all` 时若无前台动画立即满足，debug 模式输出警告。

**`transform` 归属对象不归属 track**：`scheduler.py` 维护全局 transform 注册表，key 为显示对象 id。`parallel` track 调度与 transform 调度完全独立，track 结束不停止其内 `show` 触发的 transform，`hide` 对象时才停止所有附属 transform。此规则在 `parallel` 场景下与单轨道场景完全一致。

**存档分层快照策略**：显示状态按"可序列化层"（直接快照）和"不可序列化层"（`AnimatedSprite`）分层处理，不做全量快照也不做脚本重放。`AnimatedSprite` 通过 `@restorable` 装饰器提供 `__snapshot__` / `__restore__` 扩展点，不加 `@restorable` 时以初始状态重建并输出警告。`@restorable` 与 `@saveable` 职责不重叠：前者处理显示对象重建，后者处理 `store` 游戏数据序列化。

**`transform` 读档恢复**：前台动画（`repeat 1` / `repeat N`）读档不恢复；后台动画（`repeat forever`）读档恢复但从头开始播，存档只保存 transform 名称列表，不保存进度；`AnimatedSprite` 走 `@restorable` 机制。

**`parallel` 块与存档的边界**：`checkpoint` 禁止出现在 `parallel` 块内（解析期报错）；`checkpoint` 须在所有 track 完全结束后执行，否则运行时跳过并警告；`parallel` 执行期间自动存档挂起，手动存档挂起并显示 UI 提示，两者均在块完全结束后触发。

**`on enter` 读档触发行为**：默认（`restore=auto`）读档后不重新触发，依赖状态快照恢复；`restore=always` 读档后重新触发，开发者自行保证幂等性；`restore=never` 读档后永不触发。`restore=always` 时 debug 模式输出幂等性提醒。

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

### 存档内容清单

存档保存以下状态：

**执行位置**：当前执行到哪一行（label + 行号）。call 栈在存档时丢弃，读档后以 checkpoint 下一行作为新的顶层执行起点。

**`store` 状态**：所有游戏变量，包括 `flag` 块声明的变量和通过 `$` 直接写入的变量。

**显示状态**（分层快照，见下文）：
- 每个角色当前可见性、位置、表情状态（`states` 当前值，`layers` 各层当前值）
- 当前背景（`scene`）
- 各层（`sprite`、`effect` 等）上的所有元素及自定义层列表
- camera 状态（zoom、offset、angle、follow 目标）
- `repeat forever` 后台 transform 列表（只保存名称，不保存进度）

**音频状态**：各通道当前播放文件、进度、队列、音量及淡入淡出状态。`music` / `ambient` 读档静默恢复，`sound` / `voice` 读档时清空不恢复。

**`persistent`**：跨存档全局数据，独立处理。

### 显示状态重建：分层快照

引擎采用**分层快照**策略重建显示状态，不做全量快照，也不做脚本重放。

**可序列化层**（直接快照）：角色可见性、位置、表情状态、当前背景、各层元素列表、camera 状态。这些全是基础类型或引擎内部对象，读档后直接重建显示状态，不重新执行脚本。

**不可序列化层**（`AnimatedSprite` Python 对象）：引入 `@restorable` 装饰器，专门处理读档后的重建：

```python
@restorable
class LivePortrait(AnimatedSprite):
    def __init__(self, char):
        self.char = char
        self.frames = char.load_frames()
        self.timer = 0.0

    def __snapshot__(self):
        # 存档时调用，只保存必要的重建数据
        return {"timer": self.timer}

    def __restore__(self, data):
        # 读档后调用，data 是 __snapshot__ 返回的数据
        self.timer = data["timer"]
        # frames 不需要存，重新从 char 加载即可
```

不加 `@restorable` 的 `AnimatedSprite` 读档后以初始状态重建，引擎 debug 模式输出警告：

```
AxnWarning: 'LivePortrait' is not @restorable.
  It will be restored to its initial state after loading.
  Consider adding @restorable and implementing __snapshot__ / __restore__.
```

`@restorable` 与现有的 `@saveable` / `Saveable` 设计语言一致，职责不重叠：`@saveable` 处理 `store` 内的游戏数据序列化，`@restorable` 处理显示对象的重建。

### `transform` 读档恢复策略

- **前台动画**（`repeat 1` / `repeat N`）：读档时不恢复。这类动画是演出过程的一部分，读档后演出已经过去，强行恢复到中途状态视觉上更奇怪。
- **后台动画**（`repeat forever` / `repeat forever pingpong`）：恢复，但从头开始播，不保存进度。`breathe` 这类循环动画从哪个帧开始都不影响视觉连贯性，没必要保存精确进度。存档只保存 transform 名称列表，读档后对每个可见对象重新 apply 对应的 transform，从头开始。
- **`AnimatedSprite`**：走 `@restorable` 机制，由开发者决定恢复策略。

### `parallel` 块与存档

**`checkpoint` 禁止出现在 `parallel` 块内**：解析期报错。`parallel` 执行中途其他 track 的状态无法完整快照，读档后无法正确重建。

```
AxnParseError: 'checkpoint' is not allowed inside a 'parallel' block.
  Move 'checkpoint' to after the parallel block completes.
  Line 5, scene.apy
```

**`checkpoint` 须在 `parallel` 块完全结束后执行**：`parallel` 启动后，直到所有 track 都完成（包括 `wait=none` 下未被 `wait for` 等待的 track），才允许 `checkpoint`。解析期无法静态检测此场景，改为运行时检测——`checkpoint` 执行时若有任何 `parallel` 块尚未完全结束，跳过存档并输出警告：

```
AxnWarning: 'checkpoint' skipped: a 'parallel' block has not fully completed.
  Ensure all tracks finish before placing a 'checkpoint'.
  Line 8, scene.apy
```

**自动存档**：`parallel` 块执行期间禁用自动存档，块完全结束后恢复。行为对用户透明，不需要任何语法支持。

**手动存档**：存档请求挂起，`parallel` 块完全结束后自动执行。挂起期间 UI 显示"存档将在当前演出完成后执行"的提示。可在 `options_window.apy` 中配置改为拒绝行为（弹提示后丢弃请求）。

### `on enter` 读档后的触发行为

读档后引擎不重新触发 `on enter`，依赖显示状态快照直接重建场景（音频状态同样通过快照恢复，不需要重新触发 `on enter` 里的 `play music`）。

对于 `on enter` 块内有额外副作用的场景，支持 `restore` 模式声明：

```apy
on enter chapter2_start (restore=auto):
    play music "bgm/chapter2.ogg"   # 纯副作用，读档后音频状态自动恢复，不需要重触发

on enter chapter2_start (restore=always):
    play music "bgm/chapter2.ogg"
    $ analytics.track("chapter2_enter")   # 有额外副作用，读档后也要触发
```

| `restore` 值 | 行为 |
|---|---|
| `auto`（默认） | 读档后不重新触发，依赖状态快照恢复 |
| `always` | 读档后重新触发，开发者自行保证幂等性 |
| `never` | 读档后永不触发 |

`restore=always` 时引擎 debug 模式输出提示，提醒开发者确保幂等性：

```
AxnWarning: 'on enter chapter2_start' has restore=always.
  Ensure the handler is idempotent to avoid side effects on load.
```

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

### 内置控件系统

#### 设计原则

**不做 `textbutton` / `imagebutton` 这种分类。** Ren'Py 的控件体系把内容类型混进了控件类型，导致组合场景没有标准写法。Axn-Plus 的内置控件只定义结构和交互语义，内容通过 `slot children` 填充，样式通过样式系统注入。

**控件本身不预设视觉。** `button` 不携带任何默认背景或颜色，`style button` 全局推导注入项目默认样式，不写则自然是素的。调用点不允许内联样式属性，需要临时覆盖时先定义 `mixin`，再通过 `style=` 具名参数传入。

**兼容写法**：`button` 的 `style=` 参数也接受内联样式属性，但不推荐。内联样式绕过样式系统的优先级链，产生的覆盖冲突由开发者自行负责。推荐写法始终是先定义 `mixin`，再通过 `style=` 传入。GUI 编辑器对调用点使用内联样式时以黄色警告标注"建议改用 mixin"，但不阻止保存。

**三条样式传递路径：**

```
路径一：全局 style 自动推导（零配置，按命名约定自动绑定）
路径二：gui 定义内 apply mixin（项目级复用）
路径三：调用点 style= 参数（实例级覆盖，只接受 mixin，不接受 style）
```

优先级：`调用点 style= > gui 内 apply > 自动推导 style > theme token`

**`style=` 只接受 `mixin`，不接受 `style`。** `style` 是全局推导用的，`mixin` 是手动 apply 用的，两者职责不混。

#### 控件分层

控件按职责分三层：

| 层级 | 说明 | 例子 |
|------|------|------|
| 原语控件 | 引擎内置，直接映射渲染调用，不可再拆分 | `text`、`image`、`rect`、`canvas` |
| 交互控件 | 引擎内置，提供交互语义，内容由 `slot children` 填充 | `button`、`toggle`、`slider`、`input_field` |
| 复合控件 | 用 `gui` + `extends` 定义，引擎标准库或项目级提供 | `text_button`、`image_button` 等便利封装 |

复合控件是可选的便利封装，不是独立控件类型。`text_button` 本质上等价于 `button` 内放 `text`，引擎标准库提供它只是为了减少样板代码。

#### 原语控件

```apy
text "内容"                         # 文本渲染
text "内容" (font_size=18, color=#ffffff, anchor=center)

image "assets/bg.png"               # 图片渲染
image "assets/bg.png" (size=(200, 200), anchor=top_left)

rect (size=(200, 40), color=#444444, border_radius=8)   # 矩形

canvas (size=(100, 100)):           # Python 逃逸绘制，局部 surface 坐标从 (0,0) 开始
    python:
        pygame.draw.circle(surface, (255, 0, 0), (50, 50), 30)
```

#### 交互控件

**`button`**

第一个位置参数是 `label`（字符串字面量），自动渲染为内部 `text`。需要图片或自定义内容时不传 `label`，走 `slot children`。

```apy
# 文字按钮（高频写法）
button "确认" on_click: jump route_a

# 图片按钮（走 slot children）
button on_click: jump gallery_001
    image "cg/001.png"

# 组合内容
button on_click: jump route_a
    hstack:
        image "icon.png"
        text "确认"

# 调用点样式覆盖（只接受 mixin）
button "删除" (style=danger_style) on_click: jump delete

# 素的按钮：不定义 style button，控件自然无视觉
button "跳过" on_click: jump skip
```

`button` 不携带任何默认视觉，样式完全由 `style button` 全局推导或 `style=` 参数决定。

**`toggle`**：双态开关，绑定 bool 变量

```apy
toggle "音效" bind=store["sfx_on"]
toggle "音效" bind=store["sfx_on"] (style=my_toggle_mixin)
```

**`slider`**：数值滑条，可拖动

```apy
slider bind=store["volume"] min=0.0 max=1.0
slider bind=store["volume"] min=0 max=100 step=1
```

**`input_field`**：单行文本输入框

```apy
input_field bind=store["player_name"] placeholder="输入名字" max_length=12
```

**`textarea`**：多行文本输入框

```apy
textarea bind=store["note"] placeholder="写点什么" (size=(300, 120))
```

**`number_input`**：数字专用输入，带步进按钮

```apy
number_input bind=store["age"] min=0 max=99 step=1
```

**`radio_group`**：单选组，选项列表静态声明

```apy
radio_group bind=store["difficulty"] options=["简单", "普通", "困难"]
```

**`checkbox_group`**：多选组

```apy
checkbox_group bind=store["unlocked"] options=["结局A", "结局B", "结局C"]
```

**`dropdown`**：下拉选择，Ren'Py 原生缺失的场景

```apy
dropdown bind=store["lang"] options=["中文", "English", "日本語"]
```

#### 展示控件

**`rich_text`**：富文本，支持内联样式标签

```apy
rich_text "这是<b>粗体</b>和<color=#ff0000>红色</color>文字"
```

**`typewriter`**：打字机效果文本，底层与脚本层对话文本共享 `TextRenderer` 实现

```apy
typewriter text="你好，世界。" speed=0.5
typewriter text=store["dialogue"] speed=store["text_speed"] on_complete: emit "dialogue_done"
```

与脚本层对话行的关系：两者共享同一套 `TextRenderer` 核心模块，脚本层走对话修饰符入口，UI 层走控件参数入口，行为完全一致（富文本标签、语音同步、`nowait` 等特性在两个入口同步生效）。

**`progress_bar`**：进度条，只读，不可交互

```apy
progress_bar value=80 max=100
progress_bar value=store["hp"] max=store["max_hp"] (color=#ff0000)
```

与 `slider` 的区别：`slider` 可拖动，`progress_bar` 纯展示。

**`countdown`**：倒计时展示，配合 `menu (timeout=)` 使用

```apy
countdown duration=10.0 bind=store["menu_timer"]
```

**`tooltip`**：悬停提示，内置 `follow mouse` 行为

```apy
button "？" on_click: ...:
    tooltip "点击查看详情"

# 自定义 tooltip 内容
button "装备" on_click: ...:
    tooltip:
        vstack:
            text item.name (font_size=16)
            text item.description (color=#aaaaaa)
```

**`badge`**：角标，叠加在父控件右上角

```apy
button "邮件":
    badge count=store["unread_count"]       # 数字角标
    badge text="NEW"                        # 文字角标
```

**`avatar`**：头像控件，内置圆形裁剪

```apy
avatar src="portraits/eileen.png" size=64
avatar src=store["player_avatar"] size=48 (border=(2, #ff8800))
```

**`divider`**：分割线

```apy
divider                             # 水平线
divider (color=#444444, thickness=1)
```

**`spacer`**：间距占位

```apy
spacer (size=16)
spacer grow                         # 占满剩余空间，配合 hstack/vstack 对齐用
```

**`video`**：UI 层视频控件（区别于脚本层 `play video`，此控件用于 UI 内嵌视频）

```apy
video "cutscene/intro.mp4" (loop, muted)
video "bg/rain_loop.mp4" (loop, muted, size=(fill, fill))
```

#### 导航控件

**`tab_bar`**：标签页导航

```apy
tab_bar bind=store["active_tab"] tabs=["道具", "状态", "地图"]:
    slot content

# 配合 match 切换内容
tab_bar bind=store["tab"] tabs=["道具", "状态"]:
    slot content:
        match store["tab"]:
            "道具" -> inventory_panel()
            "状态" -> status_panel()
```

**`pagination`**：分页控件

```apy
pagination bind=store["page"] total=store["total_pages"]
```

#### 容器控件

**`dialog`**：模态框视觉容器（命名刻意区别于脚本层 `modal` 动词，职责不重叠）

脚本层 `modal show` 负责执行流控制（阻塞、焦点接管、返回值），`dialog` 只负责视觉容器。

```apy
gui ConfirmDialog:
    dialog:
        background "ui/panel.png"
        padding (20, 20)
        vstack gap=16:
            slot body
            hstack gap=8:
                slot footer

# 脚本层调用
modal show "ui/confirm.apy::ConfirmDialog" as result
```

**`drawer`**：侧边抽屉

```apy
drawer side=left bind=store["drawer_open"]:
    slot children
```

**`accordion`**：折叠面板（文档示例中 `collapsible` 的内置升级版）

```apy
accordion title="高级设置":
    slot children
```

**`card`**：卡片容器

```apy
card:
    slot header
    slot content
    slot footer
```

**`popover`**：浮层容器，锚定到触发控件

```apy
button "更多":
    popover trigger=click anchor=bottom_left:
        vstack:
            button "编辑" on_click: ...
            button "删除" on_click: ...
```

**`grid`**：网格布局，背包、图鉴、CG 画廊必需

```apy
grid columns=4 gap=8:
    for item in inventory:
        item_card(item, key=item.id)

grid columns=3 gap=4 (row_gap=8):
    for cg in gallery:
        cg_thumb(cg, key=cg.id)
```

`grid` 自动计算行数，超出时与 `scroll` 配合使用：

```apy
scroll vertical:
    grid columns=4 gap=8:
        for item in inventory:
            item_card(item, key=item.id)
```

#### Round-Trip Fidelity 补充（内置控件）

| 控件 | GUI 处理方式 |
|------|-------------|
| `button` | 交互控件积木块；有 `label` 时显示文字字段，无 `label` 时显示 slot 占位符 |
| `button (style=mixin)` | 样式字段显示 mixin 名，点击跳转 mixin 定义 |
| `toggle` | 开关积木块，bind 变量名字段可编辑 |
| `slider` | 滑条积木块，min/max/step 字段可编辑 |
| `input_field` / `textarea` | 输入框积木块，placeholder/max_length 字段可编辑 |
| `number_input` | 数字输入积木块，步进按钮配置字段 |
| `radio_group` / `checkbox_group` | 选项组积木块，options 列表可编辑 |
| `dropdown` | 下拉选择积木块，options 列表可编辑 |
| `rich_text` | 富文本积木块，内联标签高亮显示 |
| `typewriter` | 打字机积木块；speed 字段可编辑；标注"底层共享 TextRenderer" |
| `progress_bar` | 进度条积木块，value/max/color 字段可编辑 |
| `countdown` | 倒计时积木块，duration/bind 字段可编辑 |
| `tooltip` | 悬停提示字段，附加在父控件节点上；自定义内容时显示 slot 占位符 |
| `badge` | 角标字段，附加在父控件节点右上角预览位置 |
| `avatar` | 头像积木块，src/size/border 字段可编辑，预览圆形裁剪效果 |
| `divider` / `spacer` | 布局辅助积木块；`spacer grow` 标注"占满剩余空间" |
| `video`（UI 层） | 视频控件积木块，与脚本层 `play video` 节点视觉区分 |
| `tab_bar` | 标签页积木块，tabs 列表可编辑，bind 变量字段可见 |
| `pagination` | 分页积木块，total 字段可编辑 |
| `dialog` | 模态容器积木块；标注"视觉容器，执行流由脚本层 modal 控制" |
| `drawer` | 抽屉积木块，side/bind 字段可编辑 |
| `accordion` | 折叠面板积木块，title 字段可编辑，展开/折叠状态可预览 |
| `card` | 卡片容器积木块，具名 slot 占位符显示 |
| `popover` | 浮层积木块，trigger/anchor 字段可编辑，锚定关系以箭头标注 |
| `grid` | 网格布局积木块，columns/gap 字段可编辑，编辑器内预览列数 |

---

### `screen`（Pygame 顶层容器）

`screen` 定义一个完整的 UI 画面，职责是**布局**——把控件组合成完整的 UI 画面。`gui` 负责控件的定义和复用，`screen` 负责把这些控件组织到一起。

绝对定位完全可用，同时扩展了语义化相对定位能力。

```apy
screen hud:
    pin top_right:
        text "00:30"
    pin bottom center:
        dialogue_box
```

#### `use`：screen 嵌套 screen

`use` 用于在一个 `screen` 内嵌入另一个 `screen`，支持传参：

```apy
screen common_header(title):
    text title:
        anchor top_center
        font_size 24

screen pause_menu:
    use common_header(title="暂停")
    button "继续" on_click: jump resume_game
    button "存档" on_click: jump save_menu
```

**`use` 只用于 screen 嵌套 screen。** `gui` 控件用直接调用语法，不用 `use`：

```apy
screen hud:
    health_bar(80, 100)             # gui 控件直接调用，不需要 use
    use common_header(title="HUD")  # screen 嵌套，必须 use
```

有插槽填充时必须写 `use`；无插槽填充时 `use` 可省略（直接调用同名 screen）：

```apy
# 有插槽，必须写 use
use base_dialog(title="设置"):
    slot body:
        toggle "音效"
    slot footer:
        button "确认" on_click: Return()

# 无插槽，use 可省略
pause_menu()    # 等价于 use pause_menu()
```

---

### `gui`（Pygame 精细控件）

`gui` 定义可复用的控件组件，职责是**控件的封装与复用**。默认在 `ui` 层，可通过 `layer=` 指定其他层。层级决定持久性——跟层走，不跟控件走。

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
    layer effect        # 跟随场景，scene 切换时清除

gui hud:                # 默认 ui 层，持久
    ...
```

**`screen` 和 `gui` 可以互相嵌套：**

```apy
screen hud:
    health_bar(80, 100)             # 在 screen 内调用 gui 控件
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

**`extends` 继承边界：属性层继承，结构层不继承。**

`extends` 继承分两层，规则明确：

- **属性层**（总是继承）：所有标量属性（`background`、`size`、`padding` 等），子类声明覆盖父类。
- **结构层**（不继承）：子控件树结构不继承。需要在父类结构内扩展时，父类通过 `slot` 声明扩展点，子类填充。

```apy
gui base_button(label, width=120):
    size (width, 40)
    background #444444
    slot prefix              # 扩展点：前置内容（图标等）
    text label:
        anchor center
    slot suffix              # 扩展点：后置内容

gui danger_button(label) extends base_button:
    background #cc2222       # 覆盖属性，结构不变，直接复用父类树

gui icon_button(label, icon) extends base_button:
    slot prefix:             # 填充父类扩展点
        default:
            image icon (size=(20, 20))
```

结构差异大的控件直接新写，不强行继承。`slot` 作为扩展点能覆盖绝大多数"想在父类结构里塞点东西"的场景；真正结构不同时，强行继承只会增加维护成本。

#### 简写语法

以下简写在不产生歧义的前提下减少代码量，完整写法始终有效。

**`style` 块：集中声明样式**

把样式属性从控件结构里剥离，状态样式在 `style` 块内以状态名作为缩进键，省略 `when` 关键字：

```apy
# 完整写法（始终有效）
gui option_button(label):
    size (200, 40)
    background #444444
    when hovered:
        background #555555
    when selected:
        background #226622
    text label

# style 块简写
gui option_button(label):
    style:
        size (200, 40)
        background #444444
        hovered:  background #555555
        selected: background #226622
    text label
```

**单行事件处理**

`on_click` 等事件处理器只有单个表达式时，允许内联到控件声明行：

```apy
# 完整写法
button "确认":
    on_click:
        $ save_game()

# 单行简写
button "确认" on_click: save_game()

# 多行退回完整块（规则与 .apy 其他地方一致：冒号后有内容单行，冒号后为空展开块）
button "确认":
    on_click:
        $ save_game()
        $ emit "saved"
```

**属性内联括号**

叶子控件（无子结构）的属性可以内联到声明行，用具名参数语法，与 `.apy` 指令语法完全一致：

```apy
# 完整写法
text label:
    font_size 18
    color #ffffff
    anchor center

# 内联简写（仅限叶子控件，有子结构时必须展开块）
text label (font_size=18, color=#ffffff, anchor=center)
```

**`vstack` / `hstack` 方向简写**

```apy
# 完整写法
stack vertical gap=8:
    button "A"
    button "B"

# 简写
vstack gap=8:
    button "A"
    button "B"

hstack gap=4:
    icon "a.png"
    text "标签"
```

`vstack` / `hstack` 是 `stack vertical` / `stack horizontal` 的别名，无额外语义。

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

### 插槽系统（`slot`）

插槽允许 `screen` 和 `gui` 声明内容空位，由调用方填入，实现模板骨架与内容的分离。

#### 具名插槽

`screen` 和 `gui` 均支持声明多个具名 `slot`，调用方用 `slot 名称:` 块填充：

```apy
# 定义带具名插槽的骨架 screen
screen base_dialog(title):
    background "ui/panel_bg.png"
    padding (20, 20)
    stack vertical gap=12:
        text title:
            font_size 20
        divider
        slot body           # 主内容区
        slot footer         # 底部按钮区（可选）

# 填充插槽（有插槽填充时必须写 use）
use base_dialog(title="道具栏"):
    slot body:
        inventory_grid(items)
    slot footer:
        button "关闭" on_click: hide screen inventory
```

**未填充的 `slot` 直接不渲染**，不报错——让可选区域零成本省略。

#### 插槽默认内容

`slot` 支持声明默认内容，调用方不填时自动使用：

```apy
screen base_dialog(title):
    stack vertical:
        slot header:
            default:
                text title (font_size=18)
        slot body
        slot footer:
            default:
                button "确认" on_click: Return()
```

#### `gui` 的具名插槽

`gui` 控件同样支持具名插槽，调用时用块语法填充：

```apy
gui card(title):
    panel:
        text title (font_size=18)
        divider
        slot content
        slot badge          # 右上角徽章区（可选）

# 调用时填充
card(title="今日任务"):
    slot content:
        text "完成剧情A"
        text "解锁CG"
    slot badge:
        icon "new_badge.png"
```

`slot children` 作为匿名插槽的语法糖保留，等价于声明一个名为 `children` 的具名插槽：

```apy
gui simple_card(title):
    panel:
        text title
        slot children       # 匿名插槽，调用方直接在块内写子控件

# 使用时
simple_card("任务"):
    text "完成剧情A"
    button "查看详情"
```

#### 插槽边界规则

- **`slot` 不支持嵌套透传**：不允许将外层 `slot` 透传给内层 `screen` 或 `gui`，需要多层组合时用 `extends` 或拆成独立定义。规则简单，插槽来源可追踪。
- **`slot` 只能声明在 `screen` 和 `gui` 定义块内**，不允许出现在 `label` 或普通 `.apy` 脚本流程中。
- 同一控件内 `slot` 名称不可重复，引擎启动时检查并报错。

---

### 扩展特性

以下特性是对 Ren'Py screen 系统的设计补全，覆盖 Ren'Py 的核心缺失：

#### 1. 控件局部状态

控件内部状态不污染全局 `store`：

```apy
gui collapsible(title):
    state expanded = False

    button title on_click: expanded = not expanded

    if expanded:
        vstack:
            slot children
```

**`state` 生命周期规则：**

1. 每个控件**实例**拥有独立的 `state` 副本——同一 `gui` 实例化两次，两份 `state` 互不影响
2. 控件销毁（所在 `screen` 关闭）时 `state` 丢弃，不持久化
3. `state` 不写入 `store`，两者完全隔离——需要持久化时开发者显式用 `store`

**实例标识（`key`）：**

引擎用 `(父容器路径, key)` 作为控件实例的稳定标识，决定 `state` 在跨帧、列表重排时能否正确保持。

静态列表（顺序固定）不写 `key` 时，引擎按声明顺序分配隐式 index（`key=0`、`key=1`……），行为可预测。动态列表内控件增减或重排时，隐式 index 会导致 `state` 错位，必须显式指定 `key`：

```apy
# 静态列表，顺序固定，隐式 index 安全
vstack:
    collapsible("章节一"):
        text "内容一"
    collapsible("章节二"):
        text "内容二"

# 动态列表，必须显式 key
for item in quest_list:
    collapsible(item.title, key=item.id):   # key 必须是稳定标识符
        text item.description
```

`key` 类型只允许 `str` 或 `int`，解析期检查。debug 模式下对动态列表内无 `key` 的控件输出警告。

**`state` 销毁时机完整规则：**

| 情况 | 行为 |
|------|------|
| screen 关闭 | 所有 state 丢弃 |
| 动态列表重排，key 匹配 | state 保留 |
| 动态列表重排，key 消失 | state 丢弃 |
| 动态列表重排，无 key | 按隐式 index 对齐，可能错位，debug 模式输出警告 |

#### 2. 控件间事件系统

控件之间通过 `emit` / `on_event` 通信，与顶层 `on enter` / `on key` 钩子命名不冲突。

**默认冒泡**：`emit` 向父容器冒泡，适合父子控件间通信：

```apy
gui tab_bar(tabs):
    state active = tabs[0]

    hstack:
        for tab in tabs:
            button tab on_click: emit("tab_changed", tab)

gui tab_content:
    on_event "tab_changed": (tab):
        show_content(tab)
```

**具名频道**：需要跨控件树通信时，使用具名频道，不通过 `store` 中转：

```apy
# 发送到具名频道
button "切换主题" on_click: emit channel="global" "theme_changed" "dark"

# 任意位置订阅（不依赖树结构）
gui theme_preview:
    on_event channel="global" "theme_changed": (theme):
        $ apply_theme(theme)
```

不写 `channel` 时默认冒泡行为不变；写 `channel` 时广播到所有订阅该频道的控件，不受树结构限制。

**停止冒泡：**

`on_event` 块末尾用 `-> stop` 阻止事件继续向上冒泡；不写或写 `-> propagate` 则继续冒泡（默认行为）：

```apy
on_event "tab_changed": (tab):
    show_content(tab)
    -> stop         # 停止冒泡，外层容器不再收到此事件

on_event "tab_changed": (tab):
    log(tab)
    -> propagate    # 继续冒泡（默认，不写等价于此）
```

`-> stop` 只对无 `channel` 的冒泡事件有意义。具名频道事件不冒泡，写 `-> stop` 时引擎启动输出警告（无效操作）。

**事件作用域规则**：
- 无 `channel`：向父容器冒泡，不广播到全局；`-> stop` 可阻断
- 有 `channel`：广播到所有订阅该频道的 `on_event`，不冒泡；`-> stop` 无效

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

根据状态动态改变样式，不需要 if/else 切换整个控件。

**预定义状态关键字**（`selected`、`hovered`、`focused`、`disabled`、`pressed`）直接用状态名：

```apy
gui option_button(label):
    style:
        background #444444
        hovered:  background #555555
        selected: background #226622
        focused:  border (2, #aaaaff)
        disabled: background #222222
```

**`state` 变量绑定**：`when` 支持绑定 `state` 局部变量的简单比较（`==`、`!=`、`>`、`<`），GUI 完整可解析：

```apy
gui affection_bar(value):
    state level = "low"

    python:
        level = "high" if value >= 80 else "mid" if value >= 40 else "low"

    fill:
        width value / 100 * 200
        when level == "high": color #ff8800
        when level == "mid":  color #aaaaff
        when level == "low":  color #444444
```

**`bind`：`selected` 状态自动绑定**

把控件的 `selected` 状态绑定到一个 `store` 或 `state` 变量，无需手动维护：

```apy
gui toggle_button(label, key):
    bind store[key]                 # selected 状态自动跟随 store[key] 的布尔值
    style:
        background #444444
        selected: background #226622
    button label on_click: store[key] = not store[key]
```

`when` 允许的表达式限制：预定义状态关键字 **或** `state` / `store` 变量的简单比较。不允许任意 Python 表达式，保证 GUI 编辑器完整可解析。复杂条件退回 `if` 块或 Python。

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

### 样式系统

Axn-Plus 的样式系统分为四个层级，职责不重叠，可以叠加使用：

| 层级 | 关键字 | 解决什么 | 粒度 |
|------|--------|----------|------|
| 设计 token | `theme` | 全局色彩/字体/间距，统一视觉语言 | 项目级 |
| 样式片段 | `mixin` | 跨控件复用的样式块，手动 apply | 片段级 |
| 全局具名样式 | `style` | 参与自动推导，零配置拾取 | 控件类级 |
| 控件继承 | `extends` | 同类控件的结构+属性整体继承 | 控件级 |

#### 第一层：`theme`（设计 token）

全局设计 token，一处定义，全局生效：

```apy
theme default:
    # 颜色
    color.primary    #ff8800
    color.surface    #2a2a2a
    color.danger     #cc2222
    color.text       #ffffff
    color.muted      #888888

    # 字体
    font.default     "fonts/default.ttf"
    font.heading     "fonts/bold.ttf"
    font.size.base   16
    font.size.small  12
    font.size.large  24

    # 间距
    spacing.sm  8
    spacing.md  16
    spacing.lg  24

    # 圆角
    radius.sm   4
    radius.md   8
```

控件内用 `$token` 引用：

```apy
gui base_button(label):
    style:
        background $color.surface
        border_radius $radius.md
    text label (color=$color.text, font=$font.default)
```

**多主题**：

```apy
theme default:
    color.primary #ff8800
    color.surface #2a2a2a

theme dark:
    color.primary #ffaa00
    color.surface #1a1a1a
```

运行时切换：

```apy
$ engine.set_theme("dark")
```

#### 第二层：`mixin`（样式片段）

可复用的样式块，通过 `apply` 手动引入，不参与自动推导：

```apy
mixin interactive:
    hovered:  background #555555
    pressed:  background #333333
    disabled: background #1a1a1a

mixin danger_style:
    background $color.danger
    hovered: background #ff3333
    border (1, #ff0000)
```

**参数化 `mixin`**：支持函数签名风格，提升复用灵活度：

```apy
mixin colored_border(color, width=2):
    border (width, color)
    hovered: border (width, color)

gui special_button(label) extends base_button:
    apply colored_border($color.primary)
    apply colored_border(#ff0000, width=3)
```

**`apply` 覆盖规则**：控件自身声明的属性优先级高于 `mixin`，多个 `mixin` 时后写的优先级高：

```apy
gui special_button(label) extends base_button:
    apply danger_style
    apply large_style       # 冲突属性取 large_style
    background #333333      # 自身声明，最终优先
```

#### 第三层：`style`（全局具名样式）

借鉴 Ren'Py `style` 系统的自动推导能力。`style` 与 `mixin` 底层相同，区别在于：

- `style` 参与自动推导，`mixin` 不参与
- 两者都可以手动 `apply`
- 语义上：`style` 是"这个控件类应该长什么样"，`mixin` 是"把这段样式混入进来"

**自动推导命名规则**：

| 命名格式 | 推导对象 |
|----------|----------|
| `控件名` | 控件基础样式 |
| `控件名_状态` | 控件指定状态样式 |
| `控件名_子元素` | 控件内指定子元素样式 |
| `控件名_子元素_状态` | 子元素的指定状态样式 |

```apy
# 定义全局 style，按命名约定自动推导
style button:
    background $color.surface
    border_radius $radius.md
    size (160, 44)

style button_hovered:
    background #555555

style button_text:
    color $color.text
    font_size $font.size.base
    anchor center

style button_text_hovered:
    color $color.primary

# 控件定义时，匹配命名约定的 style 自动生效，无需手动 apply
gui button(label):
    text label
    # button、button_hovered、button_text、button_text_hovered 全部自动拾取
```

**自动推导规则**：只拾取自身名字匹配的 `style`，不沿 `extends` 继承链向上查找。需要父类样式时显式 `apply`。

#### 优先级链

```
自身声明
  > 手动 apply mixin（后 > 前）
  > 自动推导 style
  > extends 父类自身声明
  > extends 父类 apply
  > theme token 默认值
```

冲突时规则唯一，无歧义。引擎启动时对已知冲突输出警告。

#### 完整示例

```apy
theme default:
    color.primary  #ff8800
    color.surface  #2a2a2a
    color.danger   #cc2222
    color.text     #ffffff
    radius.md      8

# 全局 style，参与自动推导
style button:
    background $color.surface
    border_radius $radius.md
    size (160, 44)

style button_hovered:
    background #555555

style button_text:
    color $color.text
    font_size 16
    anchor center

# mixin，用于手动 apply
mixin interactive:
    hovered:  background #555555
    pressed:  background #333333
    disabled: background #1a1a1a

mixin danger_style:
    background $color.danger
    hovered: background #ff3333

# 控件定义
gui base_button(label):
    apply interactive           # 手动 apply，补充 pressed/disabled 状态
    text label
    # style button、button_text 自动推导生效

gui confirm_button(label) extends base_button:
    style:
        background $color.primary   # 只覆盖背景色，其余继承

gui delete_button(label) extends base_button:
    apply danger_style              # 覆盖 interactive 的 hovered
```

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

**内置控件不分 `textbutton` / `imagebutton`**：内容类型不混入控件类型。`button` 只负责交互语义，内容通过第一个位置参数（字符串 `label`）或 `slot children` 填充。`label` 只接受字符串字面量，图片必须走 `slot children`，规则唯一无歧义。引擎标准库提供 `text_button` / `image_button` 作为便利封装，但它们是复合控件，不是独立控件类型。

**`button` 不预设视觉**：`button` 本身不携带任何默认背景或颜色。样式完全由 `style button` 全局推导或调用点 `style=` 参数决定。不定义 `style button` 时控件自然是素的，无需任何 flag。

**`style=` 只接受 `mixin`**：调用点样式覆盖只接受 `mixin`，不接受 `style`。`style` 是全局推导用的，`mixin` 是手动 apply 用的，职责不混，边界明确。

**`typewriter` 与对话文本共享 `TextRenderer`**：脚本层对话行和 UI 层 `typewriter` 控件底层共享同一套 `TextRenderer` 核心模块，两个入口行为完全一致。富文本标签、语音同步、`nowait` 等特性在两个入口同步生效，不维护两套实现。

**`dialog` vs `modal`**：脚本层 `modal` 动词负责执行流控制（阻塞、焦点接管、返回值写入）；UI 层视觉容器控件命名为 `dialog`，只负责视觉呈现和布局。两者职责不重叠，命名不冲突。`modal show "x.apy::MyDialog"` 调用的是 `gui MyDialog` 内含 `dialog` 容器的控件定义。

**`grid` 内置**：`grid` 是背包、图鉴、CG 画廊等高频场景的必需控件，不依赖 `vstack` + `hstack` 嵌套模拟。支持 `columns`、`gap`、`row_gap` 参数，与 `scroll` 配合处理超出场景。动态列表内控件必须显式指定 `key=`，规则与其他动态列表一致。

**`dropdown` / `radio_group` / `checkbox_group` 内置**：Ren'Py 原生缺失这些控件，需要手搓。Axn-Plus 直接内置，通过 `bind=` 绑定 `store` 变量，`options=` 接受静态列表或 `store` 变量。

**后端绑定**：后端在项目初始化时选定，之后固定。Pygame 项目使用 `screen` + `gui`，Qt 项目使用 `window`，不混用。

**`screen` 与 `gui` 的职责**：`screen` 负责布局——把控件组合成完整的 UI 画面；`gui` 负责控件的封装与复用。`use` 只用于 screen 嵌套 screen；`gui` 控件用直接调用语法。有插槽填充时必须写 `use`，无插槽填充时 `use` 可省略。

**持久性跟层走**：`gui` 控件放在哪一层，就遵守那一层的持久性规则。`clear` 不清除 `ui` 层（持久），但会清除 `effect` / `sprite` 层上的 `gui` 控件。需要持久但不在 `ui` 层时，显式声明 `persistent`。

**Python 逃逸的 surface 作用域**：`gui` 块内 Python 块拿到的是控件局部 surface，坐标从 `(0, 0)` 开始，引擎负责合成到正确位置。保证控件封装性，不暴露全局 surface。

**布局关键字语义化**：关键字描述意图（`pin`、`stack`、`grow`、`split`、`vstack`、`hstack`），不描述实现，不照搬前端术语。绝对定位（Ren'Py 风格）永远可用作退路。

**插槽系统**：`screen` 和 `gui` 均支持具名 `slot` 声明。未填充的 `slot` 直接不渲染，不报错。`slot` 支持 `default:` 块声明默认内容。禁止 `slot` 嵌套透传，需要多层组合时用 `extends` 或拆成独立 screen。`slot children` 作为匿名插槽语法糖保留，多插槽场景用具名 `slot`。

**简写语法**：`style` 块集中声明样式（`when` 可省略，状态名直接作为缩进键）；单行事件处理（`on_click` 只有一个表达式时可内联）；属性内联括号（叶子控件用具名参数内联）；`vstack` / `hstack` 是 `stack vertical` / `stack horizontal` 的别名。完整写法始终有效，简写是语法糖。

**样式系统四层优先级**：`自身声明 > 手动 apply mixin（后 > 前）> 自动推导 style > extends 父类自身声明 > extends 父类 apply > theme token 默认值`。`style` 参与自动推导（按 `控件名`、`控件名_状态`、`控件名_子元素`、`控件名_子元素_状态` 命名约定），`mixin` 不参与自动推导。自动推导只拾取自身名字匹配的 `style`，不沿 `extends` 继承链向上查找。`mixin` 支持参数化（函数签名风格）。冲突时规则唯一，引擎启动时对已知冲突输出警告。

**`style` 自动推导不沿继承链向上查找**：子控件不自动继承父控件的 `style`（如 `base_button_hovered` 对 `my_button` 不生效），避免"改父类 style 隐式影响所有子类"的耦合问题。需要继承父类 style 时，显式声明 `inherit_styles`：

```apy
gui my_button(label) extends base_button:
    inherit_styles          # 显式 opt-in：向上查找父类 style
    background #cc2222
# base_button_hovered 现在对 my_button 生效
# my_button_hovered 仍可覆盖（优先级不变）
```

不写 `inherit_styles` 时，编辑器在样式面板对未匹配的状态 style 输出提示（灰色标注"未定义，如需 hover 效果请定义 style my_button_hovered 或声明 inherit_styles"）。

**具名频道事件**：`emit` 不写 `channel` 时向父容器冒泡；写 `channel` 时广播到所有订阅该频道的 `on_event`，不受控件树结构限制，不冒泡。两种模式不混用，规则无歧义。

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
| `use` screen 嵌套 | 编辑器显示嵌套 screen 引用节点，参数列表可编辑 |
| `use` 省略（无插槽） | 编辑器等价处理为 `use`，无歧义 |
| `slot` 具名插槽声明 | 骨架编辑器中显示具名插槽占位符，可拖入控件 |
| `slot default:` | 占位符显示默认内容预览，调用方填充后覆盖 |
| 未填充的可选 `slot` | 编辑器以虚线框显示，提示"可选，未填充" |
| `style` 块简写 | 编辑器优先生成简写形式，完整 `when` 写法仍可解析 |
| 单行 `on_click` 内联 | 编辑器优先生成单行形式 |
| 属性内联括号 | 编辑器对叶子控件优先生成内联形式 |
| `vstack` / `hstack` | 编辑器生成简写形式，与 `stack vertical/horizontal` 等价处理 |
| `bind` | 控件节点样式面板显示绑定变量名，`selected` 状态标注"自动绑定" |
| `when state变量` 条件样式 | 样式编辑器内的条件分支，变量来源标注 |
| `theme` 块 | 独立主题编辑面板，token 列表完整可解析，色值显示色块预览 |
| `$token` 引用 | 属性字段显示 token 名+当前值，点击跳转 theme 定义 |
| 多主题切换 | 编辑器顶部主题选择器，实时预览切换效果 |
| `mixin` 声明 | 样式库面板独立区域，与 `style` 分开展示 |
| 参数化 `mixin` | 样式库面板显示参数列表，调用处显示实参值 |
| `apply` | 控件节点上显示已应用的 mixin 列表，可拖拽排序调整优先级 |
| `style` 全局具名样式 | 样式库面板独立区域，与 `mixin` 分开展示 |
| 自动推导生效的 `style` | 控件节点样式面板显示"自动推导来源：style 名"，灰色标注 |
| `emit channel=` 具名频道 | 编辑器以跨树箭头表示频道事件流向，与冒泡事件箭头视觉区分 |
| `-> stop` / `-> propagate` | 事件处理器节点上的冒泡控制标记；`-> stop` 以红色终止符显示，`-> propagate` 灰色（默认不显示） |
| `key=` 实例标识 | 动态列表内控件节点显示 key 字段；无 key 的动态列表控件以黄色警告标注 |
| `inherit_styles` | 控件节点样式面板显示"继承父类 style"标记，向上查找的 style 以蓝色来源标注区分自身推导 |
| `slot` 扩展点（父类声明） | 控件定义视图中以虚线框显示扩展点位置，子类填充后显示实际内容预览 |
| `dialog` 容器 | 模态框视觉容器积木块；标注"视觉容器，执行流由脚本层 modal 控制"，与脚本层 `modal show` 节点以连线关联 |
| `modal show` + `dialog` 组合 | 脚本区 `modal show` 节点显示目标控件路径，点击跳转对应 `gui` 定义；编辑器标注两者职责分工 |
| `button (style=mixin)` | 调用点样式字段显示 mixin 名，点击跳转 mixin 定义；传入 `style` 名时编辑器报错提示改用 `mixin` |
| `grid` | 网格布局积木块，columns/gap 字段可编辑，编辑器内按列数预览网格结构 |
| `typewriter` | 打字机积木块；标注"底层共享 TextRenderer"；speed/on_complete 字段可编辑 |

---

## 引擎架构

### 后端抽象与线程模型

引擎核心（`.apy` 运行时、`store`、脚本执行、游戏逻辑）与渲染后端完全解耦。核心不知道自己运行在 Pygame 还是 Qt 下，后端切换不影响任何游戏逻辑代码。

#### Pygame 后端

Pygame 后端占据主线程，游戏主循环直接在主线程内运行：

```
主线程
└── Pygame 事件循环（60fps）
    ├── engine.tick()       # 推进 .apy 执行
    ├── widget_tree.layout() # 布局计算
    └── widget_tree.draw()   # 渲染提交
```

无线程复杂性，结构最简单。

#### Qt 后端：线程模型（选型 B）

Qt 要求其对象的创建和操作必须在 Qt 主线程内完成。引擎核心在独立的游戏线程里运行，两者通过 Qt 信号槽桥接——`emit` 跨线程调用是 Qt 官方支持的线程安全机制，Qt 内部将其转为事件队列投递到目标线程的事件循环，不需要手动加锁。

```
Qt 主线程（Qt 事件循环 + 所有 Qt 对象的生命周期）
    ↕ Qt 信号槽（线程安全，Qt 内部队列化）
游戏线程（引擎核心 + .apy 运行时 + store）
```

**硬性约束：游戏线程绝对不能直接操作任何 Qt 对象。** 所有跨线程通信必须经过信号桥。

#### Qt 信号桥实现

**增量 UICommand 模型**：信号桥不传递全量 UI 状态快照，而是传递本帧产生的变更指令列表（`list[UICommand]`）。

**UICommand 类型安全**：`UICommand` 是 frozen dataclass，保证自身不可变，但 `value: Any` 字段如果传入可变对象（如 `list`），游戏线程继续修改该对象时 Qt 主线程会读到被修改后的数据，产生竞态。通过 `__post_init__` 白名单检查解决：

```python
_SAFE_TYPES = (int, float, str, bool, bytes, type(None), tuple)

@dataclass(frozen=True)
class UICommand:
    kind:   str
    target: str
    value:  Any

    def __post_init__(self):
        if __debug__:           # release 模式 __debug__ 为 False，自动跳过，零开销
            _assert_safe(self.value)

def _assert_safe(obj: Any, depth: int = 0):
    if depth > 8:
        return
    if isinstance(obj, _SAFE_TYPES):
        if isinstance(obj, tuple):
            for item in obj:
                _assert_safe(item, depth + 1)
    else:
        raise AxnInternalError(
            f"UICommand.value contains unsafe type: {type(obj).__name__}. "
            f"Only immutable primitives and tuples are allowed."
        )
```

debug 模式下任何把可变对象塞进 `UICommand` 的地方立即报 `AxnInternalError`，不等到竞态出现时才发现。原因：60fps 下哪怕只有一个字在打字机效果里逐字更新，全量 `to_dict()` 每帧都会序列化整棵控件树，开销不必要；全量 dict 跨线程传递还需要保证无共享可变引用，否则游戏线程继续 tick 时可能修改已发出的数据。`UICommand` 是不可变 dataclass，天然不共享引用，无需额外保护。

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Any
from PySide6.QtCore import QObject, Signal, Slot

# UI 变更指令：不可变 dataclass，跨线程传递安全
@dataclass(frozen=True)
class UICommand:
    kind: str       # "set_text" | "set_visible" | "set_style" | "scene_change" | "play_audio" | ...
    target: str     # 控件 id 或资源路径
    value: Any      # 新值，必须是基础类型或 frozen dataclass，不允许可变容器

class QtBridge(QObject):
    """唯一的跨线程通信通道。游戏线程持有引用，只调用 emit。"""

    # 游戏线程 → Qt 主线程（增量指令列表，每帧只发变更部分）
    ui_commands = Signal(list)      # list[UICommand]

    # Qt 主线程 → 游戏线程（用户输入）
    user_input  = Signal(str, object)  # (事件类型, 数据)

class GameThread(threading.Thread):
    def __init__(self, bridge: QtBridge):
        super().__init__(daemon=True)
        self.bridge = bridge
        self.engine = AxnEngine()

    def run(self):
        self.engine.load("game/script.apy")
        while self.engine.running:
            commands = self.engine.tick()   # 返回本帧产生的 UICommand 列表
            if commands:
                self.bridge.ui_commands.emit(commands)
            time.sleep(1 / 60)

class GameWindow(QMainWindow):
    def __init__(self, bridge: QtBridge):
        super().__init__()
        self.bridge = bridge
        bridge.ui_commands.connect(self._on_ui_commands)

    @Slot(list)
    def _on_ui_commands(self, commands: list[UICommand]):
        # 此处在 Qt 主线程，按指令逐条更新对应控件，不重建整棵树
        for cmd in commands:
            self._apply_command(cmd)

    def _forward_input(self, event_type: str, data=None):
        self.bridge.user_input.emit(event_type, data)

def main():
    app = QApplication(sys.argv)
    bridge = QtBridge()
    window = GameWindow(bridge)

    game = GameThread(bridge)
    game.start()

    window.show()
    app.exec()
```

#### `.apy` UI 描述 → Pygame 控件树

`.apy` 的 `gui` / `screen` 声明在解析阶段编译为 AST，运行时由控件工厂递归实例化为控件树。

**控件基类：**

```python
class Widget:
    def __init__(self):
        self.rect   = pygame.Rect(0, 0, 0, 0)
        self.children: list[Widget] = []
        self.style  = ResolvedStyle()
        self._dirty = True

    def layout(self, constraint: Constraint) -> Size:
        """自顶向下传入可用空间约束，自底向上返回实际占用尺寸。单次 pass 完成。"""
        raise NotImplementedError

    def draw(self, surface: pygame.Surface, offset: tuple[int, int]):
        raise NotImplementedError

    def handle_event(self, event: pygame.Event) -> bool:
        """返回 True 表示事件已消费，停止向上冒泡。"""
        for child in reversed(self.children):
            if child.handle_event(event):
                return True
        return False
```

布局采用 **constraint-based layout**：父控件将可用空间作为约束向下传递，子控件在约束范围内决定自身尺寸并向上返回，单次遍历完成整棵树的布局。比绝对定位灵活，比完整 CSS box model 轻量，适合游戏 UI 的使用场景。

**AST → 控件树翻译：**

```python
def build_widget(node: ASTNode, ctx: BuildContext) -> Widget:
    match node:
        case VStackNode(gap=g, children=ch):
            stack = VStackWidget(gap=g)
            stack.children = [build_widget(c, ctx) for c in ch]
            return stack

        case ButtonNode(label=l, on_click=handler):
            btn = ButtonWidget(label=l)
            btn.on_click = ctx.resolve_handler(handler)
            return btn

        case TextNode(content=c, style=s):
            return TextWidget(content=c, style=ctx.resolve_style(s))

        case PythonEscapeNode(source=src):
            # canvas / 自定义绘制，包装为代码节点
            return PythonCanvasWidget(source=src, ctx=ctx)

        case _:
            raise AxnBuildError(f"Unknown UI node: {type(node).__name__}")
```

样式在 `build_widget` 阶段一次性解析并注入（`theme token` → `style` 推导 → `mixin apply` → 自身声明），不在每帧运行时重复计算。

### 关键设计决策（引擎架构）

**Qt 后端选型 B（独立游戏线程 + 信号桥）**：引擎核心不依赖 `QTimer` 或任何 Qt 概念，游戏逻辑与 Qt 事件循环完全隔离。跨线程通信全部经过 `QtBridge` 的信号槽，Qt 内部保证线程安全，不需要手动锁。相比选项 A（独立进程）避免了 IPC 的延迟和数据同步复杂性；相比选项 C（游戏循环挂 `QTimer`）保持了引擎核心对后端的无感知。

**信号桥采用增量 UICommand 模型**：`engine.tick()` 返回本帧产生的 `list[UICommand]`，只发变更部分，不传递全量 UI 状态快照。理由：全量快照在打字机效果等高频更新场景下每帧都会序列化整棵控件树，开销不必要；全量 dict 还需保证跨线程无共享可变引用。`UICommand` 是 frozen dataclass，天然不可变，跨线程传递安全，Qt 主线程按指令逐条更新对应控件，不重建整棵树。

**游戏线程绝对不操作 Qt 对象**：这是 Qt 线程模型的硬性要求，也是信号桥设计的核心约束。所有需要触达 Qt 控件的操作必须通过 `bridge.signal.emit()` 发出，由 Qt 主线程的槽函数执行实际操作。

**Pygame 控件树采用 constraint-based layout**：单次遍历完成布局，脏标记控制重绘范围，避免每帧全量重算。不照搬 DOM/CSS 的完整 box model，只实现引擎 UI 实际需要的布局能力（`vstack`、`hstack`、`pin`、`grid`、`grow`、`split`）。

**样式在构建阶段解析，不在运行时重算**：`theme token` 展开、`style` 自动推导、`mixin apply`、优先级合并全部在控件树实例化时完成，运行时控件持有已解析的 `ResolvedStyle`，不重复查找样式链。状态样式（`hovered`、`selected` 等）例外——这部分在控件 `draw` 时按当前状态选取对应的预解析样式块。

---

## 项目结构

### 用户项目目录

引擎强制要求两个文件存在，其余完全自由：

```
Project_name/
   flow.apy               # 入口文件（强制）
   options_window.apy     # 引擎配置 + 预构建 UI（强制）
```

推荐结构（`axn init` 生成的默认骨架）：

```
Project_name/
   flow.apy
   options_window.apy
   main/
      scripts/
         script.apy
      image/
         gui/             # 代码 UI 所需图片资源（可为空）
      audio/
      video/
      font/
```

`main/` 目录不是强制要求。小项目完全可以只用 `flow.apy` + `options_window.apy` 实现整个项目。

#### `flow.apy`

入口文件，推荐用法是只做顶层流程调度，用 `call` 把各章节分发到独立文件：

```apy
label start:
    call chapter1.apy::prologue
    call chapter2.apy::chapter_two
    ...
```

引擎启动时扫描所有 `.apy` 文件，label 全局可见，支持两种 `call` 方式：

```apy
call chapter_one              # 全局符号表查找
call chapter2.apy::chapter_one  # 显式路径引用（包含 :: 时触发）
```

两种方式可混用，显式路径永远优先。label 命名冲突由开发者自行管理，引擎启动时报错提示。显式路径引用的文件或 label 不存在时，同样在启动时报错，不等到运行时。

#### `options_window.apy`

引擎配置（文件底部）+ 用 Pygame 代码绘制的预构建 UI（标题画面、菜单、对话框等）。UI 部分不依赖图片资源，完全由代码实现。

#### 资源引用规则

有 `main/` 目录时，按指令类型自动在对应子目录查找，文件名不需要扩展名：

```apy
show home          # 在 main/image/ 下查找 home.*
play music rain    # 在 main/audio/ 下查找 rain.*
```

无 `main/` 目录时，需要提供相对于项目根目录的完整路径：

```apy
show "path/to/home.png"
play music "path/to/rain.ogg"
```

**支持的文件格式：**

| 类型 | 格式 |
|------|------|
| 图片 | png, jpg, webp |
| 音频 | ogg, mp3, wav |
| 视频 | mp4, webm |
| 字体 | ttf, otf |

不在支持列表内的格式，或同目录下存在同名不同扩展名的文件，必须带完整扩展名：

```apy
play audio "rain.avi"       # 不支持的格式，需要扩展名（并引入对应编解码器）
show "home.jpg"             # 同目录下同时有 home.png 和 home.jpg 时消歧义
```

同目录下存在同名不同扩展名文件且未指定扩展名时，引擎启动时报错，不自动选择。

#### `show` 类型推断

`show` 后跟裸名字时，引擎查符号表推断类型，不需要子命令：

```apy
show home          # 符号表：图片文件 → 场景背景
show eileen        # 符号表：define char → 立绘
show hud           # 符号表：gui 定义 → UI 控件
show MyEffect()    # 符号表：自定义可显示类 → 自定义对象
```

自定义可显示类（继承 `AnimatedSprite` 的 Python 类）必须在 `.apy` 文件中显式声明才能进入符号表：

```apy
import MyEffect from "effects/my_effect.py"
import LivePortrait from "characters/live_portrait.py"
```

符号表里同一名字对应多种类型时，引擎启动时报错，要求重命名消歧义。

### 引擎目录结构

```
axn_plus/
   __init__.py
   engine.py                 # AxnEngine 主类，对外 API 入口
   core/
      __init__.py
      runner.py              # .apy 执行运行时
      store.py               # Store / persistent 实现
      scheduler.py           # 并行轨道、wait for、动画调度
      checkpoint.py          # 存档 / 读档逻辑
      hot_reload.py          # HotReloader，文件监听 + 分级重载
   parser/
      __init__.py
      lexer.py
      ast_nodes.py           # 所有 AST node dataclass，两层 parser 共享
      error.py               # AxnParseError 等
      full_parser.py         # 三遍扫描，引擎启动用，正确性优先
      incremental_parser.py  # 第一遍 + 局部重解析，LSP 用，容错优先，允许返回不完整 AST
   backends/
      __init__.py
      base.py                # AbstractBackend 接口
      pygame/
         __init__.py
         backend.py
         widget_tree.py      # constraint-based layout
         renderer.py
      qt/
         __init__.py
         backend.py
         bridge.py           # QtBridge 信号桥
         window.py
   ui/
      __init__.py
      style.py               # theme / mixin / style 解析与优先级合并
      slot.py                # 插槽系统
      widgets/
         __init__.py
         primitives.py       # text / image / rect / canvas
         interactive.py      # button / toggle / slider 等
         layout.py           # vstack / hstack / grid / pin 等
   asset/
      __init__.py
      loader.py              # 资源加载、缓存
      audio.py               # play / pause / stop / 多通道管理
      video.py
   apy/
      __init__.py
      stdlib.py              # 引擎内置指令实现
      transition.py          # 内置过渡效果
      transform.py           # keyframe 动画系统
      saveable.py            # @saveable / Saveable 基类
   cli/
      __init__.py
      init.py                # axn init，生成项目骨架
      build.py               # axn build，打包发布
      run.py                 # axn run，开发期启动
```

`parser/` 独立，供 Axn-Editor 的 LSP 插件直接复用，不与运行时耦合。LSP 插件复用 `parser/incremental_parser.py`，不使用全量三遍扫描器，保证实时补全响应速度。两层实现共享 `lexer.py` 和 `ast_nodes.py`。`core/` 中无任何 pygame 或 Qt import，后端通过 `backends/base.py` 的抽象接口交互。`cli/` 提供 `axn init` / `axn run` / `axn build` 三个子命令。

---

## 多通道音频

### 通道模型

引擎内置四个固定通道，同时支持用户自定义通道：

| 通道 | 对应指令 | 典型用途 |
|------|---------|---------|
| `music` | `play music` | 背景音乐 |
| `sound` | `play sound` | 音效 |
| `voice` | `play voice` | 语音 |
| `ambient` | `play ambient` | 环境音 |

四个内置通道名为保留关键字，不允许自定义通道使用相同名称，引擎启动时检查并报错。

`play music` 等价于 `play audio (channel="music")`，内置通道是语法糖。

### 默认行为：串行队列

同一通道内默认串行，新的 `play` 加入队列，等前一个结束再播：

```apy
play music "bgm/morning.ogg"
play music "bgm/tension.ogg"   # 排队，morning 结束后播
```

队列长度默认上限为 16，超出时输出警告。可在 `options_window.apy` 中配置。

### 并行：使用自定义通道

不同通道之间天然并行，通过自定义通道实现同类音频的并行播放：

```apy
channel create bg_layer         # 推荐先声明
channel create ambient_layer

play music "bgm/morning.ogg" (channel="bg_layer")
play music "bgm/rain.ogg" (channel="ambient_layer")   # 与 bg_layer 同时播放
```

也可以不预先声明直接在 `play` 里创建，但推荐先声明使依赖关系可见。

### `stop` 行为

```apy
stop music              # 停止当前播放，队列保留
stop music (clear)      # 停止当前播放并清空队列
```

推荐显式使用 `stop` 而不是依赖新 `play` 打断，语义更清晰。

### 存档与读档

| 通道 | 读档行为 |
|------|---------|
| `music` | 静默恢复，保留播放进度、队列、音量状态 |
| `ambient` | 静默恢复，保留播放进度、队列、音量状态 |
| `sound` | 读档时清空，不恢复 |
| `voice` | 读档时清空，不恢复 |
| 自定义通道 | 默认静默恢复，可在 `options_window.apy` 中覆盖 |

`sound` 和 `voice` 不恢复的原因：短音效和语音通常与具体动作绑定，读档后对应动作已经过去，恢复没有意义。

静默恢复保存的内容：当前播放文件名、播放进度（时间戳）、队列里剩余的待播项、通道的音量和淡入淡出状态

---

## 编译器与运行时系统

### 编译目标：字节码

`.apy` 文件经三遍扫描生成 AST 后，编译为自定义字节码执行。选择字节码而非直接解释 AST 或生成 Python 代码的理由：

- 存档机制依赖"精确执行位置"，字节码的 PC（程序计数器）天然满足，AST 解释实现起来很别扭
- `.apy` 的 Python 块已有 `compile()` + `exec()` 设计，字节码方案与之完美兼容
- 可缓存字节码，对未修改文件跳过重新编译，降低重启开销

---

### 错误处理系统

#### 错误分类

```
AxnError (基类)
├── AxnParseError        # Lexer / Parser 阶段
├── AxnCompileError      # 编译器阶段（AST → 字节码）
├── AxnRuntimeError      # VM 执行阶段
│   ├── AxnNameError     # 未声明变量引用
│   ├── AxnTypeError     # 类型不匹配（flag 注解检查，debug 模式专属）
│   ├── AxnJumpError     # jump/call 目标不存在
│   └── AxnSaveError     # 存档序列化失败
├── AxnAssetError        # 资源加载失败
│   └── AxnVoiceError    # voice 短路径推断失败
└── AxnInternalError     # 引擎自身 bug，直接暴露 Python traceback
```

#### 错误信息格式

**脚本作者看到的（AxnParseError / AxnRuntimeError 等）：**

```
AxnParseError: Unclosed bracket in '$' line
  → scene.apy, line 12

  10 |
  11 | eileen: "你好。"
  12 | $ x = (
            ^
  13 |     1 + 2

Hint: Multi-line Python belongs in a 'python:' block.
```

格式规则：
- 错误类型 + 一句话描述
- 文件名 + 行号
- 上下文窗口（前 2 行，错误行，后 1 行）
- `^` 指向具体列（能定位到列时）
- `Hint:` 给出修复建议（能给的时候给）

边界情况：
- 文件开头不足 2 行时，只显示实际存在的行
- 文件末尾没有后 1 行时，显示 `# EOF` 占位

**引擎内部错误（AxnInternalError）：**

```
AxnInternalError: Unexpected state in scheduler.py
This is a bug in the Axn-Plus engine, not your script.
Please report at: https://github.com/axn-plus/axn-plus/issues

--- Internal Traceback ---
Traceback (most recent call last):
  File "axn_plus/core/scheduler.py", line 84, in tick
    ...
```

明确告诉作者"这不是你的问题"，然后才暴露 traceback。

**多位置错误（label 冲突、循环引用）：**

```
AxnCompileError: Duplicate label 'morning_scene'
  → scene.apy, line 5
  → chapter2.apy, line 103

Hint: Labels are globally visible. Rename one to resolve the conflict.
```

```
AxnParseError: Circular inheritance detected
  eileen_adult → eileen_teen → eileen_adult

  → characters.apy, line 8   (define char eileen_adult extends eileen_teen)
  → characters.apy, line 15  (define char eileen_teen extends eileen_adult)
```

**警告格式：**

不中断执行，格式为 `AxnWarning: [模块名] 描述 → 位置`：

```
AxnWarning: [scheduler] 'wait for all' has no finite transforms to wait for.
  All transforms are 'repeat forever'. Did you mean to use 'wait'?
  → eileen_enter animation block, scene.apy, line 3
```

#### Hint 策略

不是每个错误都有 Hint，以下情况必须给：

| 错误场景 | Hint 内容 |
|----------|-----------|
| `$` 行括号未闭合（旧行为，现已改为警告） | 改用 `python:` 块 |
| `with store` 内出现下标访问 | 先在 `python:` 块算好再赋值 |
| `menu as` 内出现 `jump` | `menu as` 不允许跳转，改用 `menu` |
| `say` 传入静态角色名 | 改用 `角色:` 语法 |
| `define char narrator` | `narrator` 是保留关键字 |
| label 命名冲突 | 列出所有冲突位置 |
| 循环 `include` | 打印完整引用链 |
| 循环继承 | 打印继承链 |

#### 错误处理矩阵

```
错误类型              开发模式（引擎运行）        发布包
─────────────────────────────────────────────────────────────
AxnParseError        终端完整报错 + 停止          编译阶段拦截，不进包
AxnCompileError      终端完整报错 + 停止          编译阶段拦截，不进包
AxnWarning           终端显示 + 继续运行          完全静默
AxnTypeError         终端完整报错 + 继续          完全静默
assert               执行 + 报错                  剥离
AxnNameError         终端完整报错 + 停止          错误界面 + crash.log
AxnJumpError         终端完整报错 + 停止          错误界面 + crash.log
AxnSaveError         终端完整报错                 错误界面 + crash.log
AxnInternalError     完整 Python traceback        错误界面 + crash.log
AxnAssetError        终端完整报错 + 停止          静默 + crash.log
  └── 图片/立绘      同上                         不渲染 + crash.log
  └── 音视频         同上                         跳过 + crash.log
  └── AxnVoiceError  同上                         静默无声 + crash.log
```

发布包中保留的错误统一走两层处理：
1. 写入 `crash.log`（开发者事后可查）
2. 显示对玩家友好的错误界面（引擎提供默认样式，开发者可在 `options_window.apy` 中自定义）

#### `options_window.apy` 错误相关配置

```apy
engine:
    strict = false                      # true 时 Warning 升级为报错
    release_asset_missing = "silent"    # silent / placeholder / error
    error_report_url = ""               # 自动上报地址，空则不上报
    locale = "zh"                       # 错误信息语言跟随此设置
    ignore_multiline_dollar = false     # true 时静默 $ 多行续行警告
```

#### `$` 行括号续行

从"解析期报错"改为"运行期警告"，开发者选择 ignore 后自行承担后果：

```apy
$ x = (
    1 + 2
)
```

```
AxnWarning: [parser] '$' line contains multi-line expression.
  Bracket continuation is allowed but not recommended.
  Behavior may be undefined in some contexts.
  → scene.apy, line 12

Hint: Use a 'python:' block for multi-line expressions.
  Add 'ignore_multiline_dollar = true' in options_window.apy to suppress this warning.
```

---

### Lexer

#### 设计约束

**`tolerant` 模式（LSP 专用）：**

Lexer 支持 `tolerant` 参数，全量 parser 默认关闭，LSP 增量 parser 开启：

```python
class Lexer:
    def __init__(self, source: str, filename: str, tolerant: bool = False): ...
```

`tolerant=True` 时，遇到非法字符不抛出 `AxnParseError`，改为插入 `ERROR` token 并跳过继续扫描。增量 parser 遇到 `ERROR` token 时跳到下一个安全点（下一个 `NEWLINE` + `DEDENT` 归零）继续解析，返回部分 AST。这保证用户正在输入到一半的代码不会导致整个文件的补全失效。

两层 parser 共享 `lexer.py` 和 `ast_nodes.py`，通过 `tolerant` 参数区分行为，不维护两套 Lexer 实现。

**缩进规则：**
- 同一文件内必须使用统一的缩进单位（空格数量或 Tab），不允许混用
- 第一次遇到缩进时记录该文件的缩进单位，后续不一致时报错
- 不限制缩进数量（2、3、4、5 均可），但同一文件必须统一
- Tab 与空格不允许在同一文件内混用
- Axn-Editor 默认使用 3 空格缩进

```
AxnParseError: Inconsistent indentation
  This file uses 3-space indentation (detected at line 2).
  → scene.apy, line 8, col 1
Hint: Use the same indentation unit throughout the file.
```

**字符串引号：**
- 只允许双引号 `"`
- Python 块内不受此限制，单双引号随意
- Axn-Editor 自动补全只处理双引号

**颜色字面量：**
- `#rrggbb` 和 `#rrggbbaa` 在 Lexer 层直接识别为 `COLOR` token，不当作注释

**容错模式（`tolerant`）：**
- Lexer 支持 `tolerant: bool` 构造参数，默认 `False`
- `tolerant=False`：遇到非法字符或未闭合结构立即抛 `AxnParseError`，用于引擎启动的全量三遍扫描，正确性优先
- `tolerant=True`：遇到错误跳过当前字符，插入 `ERROR` token 后继续，用于 LSP 增量解析器，容错优先
- 增量解析器收到含 `ERROR` token 的流后，跳到下一个安全点（下一个 `NEWLINE` + `DEDENT` 归零）继续解析，返回部分 AST
- 目的：用户在编辑器内输入到一半时，LSP 仍能为剩余部分提供补全和语法高亮，不因单行残缺导致整个文件失效
- 两层实现共享同一个 `Lexer` 类，`tolerant` 参数决定错误处理策略，不维护两套实现

#### Token 类型

```python
from enum import Enum, auto

class TokenType(Enum):
    # 字面量
    STRING          = auto()    # "文字"
    NUMBER          = auto()    # 42, 3.14
    BOOL            = auto()    # True, False
    NONE            = auto()    # None
    COLOR           = auto()    # #ff8800

    # 标识符与关键字
    IDENTIFIER      = auto()

    # .apy 引擎关键字
    DEFINE          = auto()
    CHAR            = auto()
    EXTENDS         = auto()
    LABEL           = auto()
    JUMP            = auto()
    CALL            = auto()
    RETURN          = auto()
    MENU            = auto()
    CHOICE          = auto()
    IF              = auto()
    ELIF            = auto()
    ELSE            = auto()
    UNLESS          = auto()
    MATCH           = auto()
    SHOW            = auto()
    HIDE            = auto()
    SCENE           = auto()
    CLEAR           = auto()
    PLAY            = auto()
    STOP            = auto()
    PAUSE           = auto()
    RESUME          = auto()
    WAIT            = auto()
    CAMERA          = auto()
    LAYER           = auto()
    EXPRESSION      = auto()
    SAY             = auto()
    NARRATE         = auto()
    WITH            = auto()
    ANIMATION       = auto()
    TRANSFORM       = auto()
    PARALLEL        = auto()
    TRACK           = auto()
    TRANSLATE       = auto()
    INCLUDE         = auto()
    IMPORT          = auto()
    TEMPLATE        = auto()
    SCREEN          = auto()
    GUI             = auto()
    WINDOW          = auto()
    SLOT            = auto()
    STATE           = auto()
    STYLE           = auto()
    MIXIN           = auto()
    THEME           = auto()
    APPLY           = auto()
    EMIT            = auto()
    ON              = auto()
    FLAG            = auto()
    CONST           = auto()
    SET             = auto()
    CHECKPOINT      = auto()
    ASSERT          = auto()
    MODAL           = auto()
    INPUT           = auto()
    PYTHON          = auto()    # python: 块关键字
    NARRATOR        = auto()    # 保留关键字
    STA             = auto()
    DYN             = auto()
    AS              = auto()
    FOR             = auto()
    FROM            = auto()
    IN              = auto()

    # 行首特殊符号
    DOLLAR          = auto()    # $
    AT              = auto()    # @（旁白）
    ARROW           = auto()    # ->

    # 结构符号
    COLON           = auto()    # :
    COMMA           = auto()    # ,
    LPAREN          = auto()    # (
    RPAREN          = auto()    # )
    LBRACKET        = auto()    # [
    RBRACKET        = auto()    # ]
    EQUALS          = auto()    # =
    PLUS_EQUALS     = auto()    # +=
    DOUBLE_COLON    = auto()    # ::（跨文件引用）

    # 布局
    NEWLINE         = auto()
    INDENT          = auto()
    DEDENT          = auto()
    EOF             = auto()

    # 注释（保留用于 round-trip）
    COMMENT         = auto()

    # Python 块内容（整体作为字符串保留，不解析内部）
    PYTHON_BLOCK    = auto()
```

#### Token 数据结构

```python
@dataclass
class Token:
    type:  TokenType
    value: Any      # 原始值；STRING 已去引号，NUMBER 已转型
    raw:   str      # 原始文本，用于 round-trip
    file:  str
    line:  int
    col:   int
```

---

### Parser（三遍扫描）

#### 第一遍：名字收集

只收集所有 `define` 的名字，不解析字段内容和 `extends` 关系。目标是建立全局名字集合，使第二遍能识别跨文件引用。成本极低，只需识别行首的 `define` 关键字。

#### 第二遍：继承图与符号表

扫描所有 `define extends` 关系，建立继承有向图，拓扑排序确定解析顺序。检测循环继承并在此阶段报错。同时建立全局符号表（名字 → 类型），供第三遍使用。

#### 第三遍：完整解析

按拓扑顺序完整解析所有文件，展开继承字段，构建完整 AST。行首遇到已知角色名走对话路径，否则走指令路径。label 冲突检查在此阶段完成。`sta label` 的静态检查也在第三遍完成——遇到 `$` 块或 `python:` 块时立即报错，不留到编译器阶段。

#### 增量缓存

对未修改文件跳过第三遍，直接使用缓存 AST。使用 **mtime + hash 双重校验**：

- mtime 未变：直接认为有效，跳过 hash 计算
- mtime 变了：计算 SHA-256 hash，hash 也变了才视为需要重新解析
- 缓存损坏（pickle 读取失败）：视为无效，重新解析

缓存文件存放在项目的 `.axncache/` 目录，构建发布包时不打入包内。

#### 注释归属规则

与文档 `.apy` 脚本格式章节中的规则一致：

- **行内注释**（`代码  # 注释`）：Lexer 阶段归属同行节点
- **行间注释**（单独占行）：Parser 阶段按缩进层级归属父节点，空行切断与下方节点的联系
- 顶层行间注释（无父节点）作为独立注释节点存在
- 所有注释以 `COMMENT` token 保留，不丢弃，保证 round-trip fidelity

#### AST 节点基类

所有节点携带位置信息和注释归属：

```python
@dataclass
class ASTNode:
    file:     str
    line:     int
    col:      int
    comments: list[CommentNode] = field(default_factory=list)

@dataclass
class CommentNode(ASTNode):
    content: str
    inline:  bool   # True = 行内注释，False = 行间注释
```

#### 三遍扫描组装入口

```python
class Parser:
    def __init__(self, files: dict[str, str]):
        # files: filename → source 字符串
        self.files  = files
        self.lexed: dict[str, list[Token]] = {}

    def parse_all(self) -> dict[str, FileNode]:
        # Lex 所有文件
        for filename, source in self.files.items():
            lexer = Lexer(source, filename)
            self.lexed[filename] = lexer.tokenize()

        # 第一遍
        first       = FirstPass(self.lexed)
        known_names = first.run()

        # 第二遍
        second                    = SecondPass(self.lexed, known_names)
        parse_order, symbol_table = second.run()

        # 第三遍（含增量缓存）
        third = ThirdPass(self.lexed, symbol_table, parse_order)
        return third.run()
```

#### 关键设计决策（Parser）

**`sta label` 静态检查在第三遍完成**：遇到 `$` 块或 `python:` 块时立即报错，不留到编译器阶段，保证"sta 声明即保证"的语义在最早阶段兑现。

**第三遍遇到第一个错误即停止**：不尝试继续收集后续错误，避免级联错误产生误导性的噪音。

**`define` 只允许出现在文件顶层**：保证第一、二遍扫描无需理解嵌套结构，各遍成本极低。

**label 冲突在第三遍统一检查**：`label_table` 跨文件共享，第三遍每解析一个 label 就写入并检查冲突，确保任意文件顺序下均能检测到。

---

### 编译器（AST → 字节码）

#### 编译目标结构

- **CompiledLabel**：单个 label 的编译产物，包含指令列表、常量池、参数表
- **CompiledChunk**：单个 `.apy` 文件的编译产物，包含该文件所有 label、animation、on_hook
- **CompiledProject**：所有文件的编译产物，VM 直接使用；包含全局 label 索引、define 表、flag 注册表、const 表

核心数据结构骨架：

```python
@dataclass
class CompiledLabel:
    name:         str            # label 名，热重载匹配用，不可省略
    filename:     str            # 来源文件，错误定位 + 热重载查询用
    instructions: list[Instruction]
    constants:    list[Any]      # 常量池，可哈希对象自动去重
    params:       list[str]      # label 参数名列表

@dataclass
class Instruction:
    opcode:   OpCode
    operand:  Any                # 常量池索引或直接值
    line:     int                # 源码行号
    filename: str                # 源码文件名
```

**`CompiledLabel.name` 是必填字段**：热重载时通过名字匹配正在执行的 CallFrame，找到后替换引用并重置 PC。匿名 parallel track 的命名规则为 `__parallel_{parent_label}_{track_index}__`，有命名的 track 用命名作为后缀，保证名字全局唯一。

**`CompiledProject.label_index` 设计为可变 dict**：热重载就是在运行时替换这个索引里的 `CompiledLabel` 对象，不能是 frozen 结构。

#### 指令集

字节码指令分以下几类：

| 类别 | 指令举例 |
|------|---------|
| 控制流 | `JUMP` `JUMP_IF` `JUMP_IF_NOT` `CALL` `RETURN` `HALT` |
| Python 块 | `EXEC_PYTHON` `PUSH_EXPR` |
| 对话与旁白 | `DIALOGUE` `NARRATOR` `WAIT_CLICK` |
| 显示控制 | `SHOW` `HIDE` `SCENE` `CLEAR` `EXPRESSION_CMD` |
| 音视频 | `PLAY_AUDIO` `STOP_AUDIO` `PAUSE_AUDIO` `RESUME_AUDIO` `PLAY_VIDEO` `STOP_VIDEO` |
| 镜头 | `CAMERA` |
| 等待 | `WAIT_DURATION` `WAIT_FOR` |
| 菜单 | `MENU` `CHOICE` |
| 存档 | `CHECKPOINT` |
| 层管理 | `LAYER_MANAGE` |
| 输入控制 | `INPUT_DISABLE` `INPUT_ENABLE` |
| 模态框 | `MODAL_SHOW` `MODAL_HIDE` |
| 并行 | `PARALLEL_BEGIN` `PARALLEL_END` |
| 存取 | `LOAD_CONST` `STORE_VAR` `LOAD_VAR` `WITH_STORE` |
| 动画 | `CALL_ANIMATION` |
| 调试 | `ASSERT`（release 模式不生成） `DEBUG_BREAK` |

每条指令携带：opcode、operand（常量池索引或直接值）、源码行号、源码文件名。行号和文件名用于运行时错误定位。

#### 常量池

所有复杂数据（字符串、命令对象、Python code object）统一放常量池，指令只存索引。可哈希对象自动去重，不可哈希对象（code object）直接追加。

#### 关键设计决策（编译器）

**跨文件跳转 operand 格式用 `(文件名, label名)` 元组**：视觉小说项目规模不会有性能瓶颈；调试时 operand 直接可读；展平方案在增量重编译时需要重新计算所有地址，维护成本高。VM 运行时通过全局 label 索引定位目标，天然支持跨文件跳转。

**跳转地址延迟回填**：编译单个文件时跨文件 label 地址尚不完整，所有文件编译完后统一验证回填。跳转目标不存在时在此阶段报 `AxnCompileError`，不等到运行时。

**Python 块编译为 code object**：`compile()` 在编译阶段执行，运行时只做 `exec()`。语法错误在编译阶段暴露，文件名和行号偏移传入 `compile()`，保证 traceback 正确指向 `.apy` 源文件。

**parallel 块编译为多个独立 CompiledLabel**：每个 track 生成一个匿名 `CompiledLabel`，命名规则为 `__parallel_{parent_label}_{track_index}__`，有命名的 track 用命名作为后缀。`PARALLEL_BEGIN` 指令的 operand 存这些匿名 label 名的列表，交给 Scheduler 并发管理。

**release 模式差异**：`assert` 节点不生成指令；Python `compile()` 使用 `optimize=2`；`AxnTypeError` 类型检查指令不生成。

**`sta label` 已在 Parser 阶段验证**：编译器不重复检查，直接信任 `is_static` 标志。

---

### VM

#### 执行模型

- **tick 驱动**：VM 暴露 `tick()` 方法，每帧调用一次。每次 tick 执行指令直到遇到等待点（用户点击、计时器、动画、并行轨道），返回本帧产生的 `UICommand` 列表
- **调用栈**：每个 label 调用对应一个 CallFrame，持有对 `CompiledLabel` 的间接引用（不拷贝指令列表），包含 PC、局部变量表。label 参数绑定后写入 locals，执行时优先查 locals 再查 store。间接引用是热重载的基础——替换 `CompiledLabel` 后 frame 自动使用新指令，无需重建调用栈
- **指令分发**：使用分发表（`dict[OpCode, Callable]`）而非 match/case，可维护性更好

#### 等待状态

| WaitKind | 触发 | 清除 |
|----------|------|------|
| `CLICK` | `DIALOGUE` / `WAIT_CLICK` 指令 | 用户点击，由后端调用 `vm.on_click()` |
| `DURATION` | `WAIT_DURATION` 指令 | 每帧递减，归零自动清除 |
| `ANIMATION` | `WAIT_FOR` 指令 | 目标动画完成 |
| `PARALLEL` | `PARALLEL_BEGIN` 指令 | Scheduler 判断轨道完成条件满足 |

#### Store

- debug 模式下对有类型注解的 flag 变量通过 `__setitem__` 钩子做即时类型检查，错误信息包含声明位置
- release 模式下退化为普通 dict，零开销
- const 写入只读层，尝试赋值时抛 `AxnRuntimeError`
- `with store` 原子块：执行前对涉及 key 做浅拷贝快照，异常时自动回滚

#### Scheduler

负责并行轨道执行和 transform 注册表管理：

- 每个 track 对应一个独立 CallFrame，Scheduler 每帧独立步进每个 track
- **tick 顺序**：普通 track 按声明顺序先 tick，interactive track 永远最后 tick；顺序确定性有保证，开发者可依赖
- interactive track 的对话行独占用户输入，其他 track 继续运行
- transform 注册表以显示对象 id 为 key，`hide` 对象时自动停止所有附属 transform
- `wait for all` 只等前台动画（`repeat 1` / `repeat N`），后台动画（`repeat forever`）不参与
- 暴露 `active_label_files() -> set[str]` 接口供热重载器查询当前正在执行的文件集合
- 暴露 `has_active_parallel() -> bool` 接口供热重载器判断是否有 parallel 块正在执行

#### 关键设计决策（VM）

**Python 块执行环境**：store 作为 `globals`，frame.locals 作为 `locals`。变量写入 store，跨 label、跨 jump 天然持久化。引擎内置符号通过 `__builtins__` 注入为只读层，用户代码可见但不可覆盖。

**transform 归属对象不归属 track**：track 结束不停止其内 `show` 触发的 transform，`hide` 对象时才停止所有附属 transform。此规则在 parallel 场景下与单轨道场景完全一致。

**parallel 完成判断**：`wait=all` 等所有 track 完成；`wait=any` 等最先完成的 track；`wait=none` 立即推进，track 在后台继续运行直到自然结束。`wait=any` 与 interactive track 共存时编译器阶段已报错，VM 不需要处理此情况。

**错误边界**：Python 块内的异常包装为 `AxnRuntimeError` 上抛，携带源码位置。已格式化的 `AxnError` 直接上抛不二次包装。引擎内部异常抛 `AxnInternalError`，暴露完整 Python traceback。

**CallFrame 持有间接引用**：CallFrame 不内联指令列表，通过 `label_ref: CompiledLabel` 持有对象引用，每帧通过引用取指令。热重载时只需替换全局 label 索引中的 `CompiledLabel` 对象并重置相关 frame 的 PC，不需要重建调用栈。Scheduler 内的 track frame 同样遵守此规则。

**Scheduler tick 顺序确定性**：普通 track 按声明顺序先 tick，interactive track 永远最后 tick。此顺序作为规范写入文档，开发者可依赖，引擎不允许在不同版本间改变此顺序。

**热重载分级处理**：热重载请求在帧末批量处理（防抖合并）。parallel 块执行期间，不影响当前执行的文件立即处理，影响当前执行文件的请求推迟到 parallel 结束后自动处理。编辑器显示待处理计数，parallel 结束后消失。

---

### 核心数据结构骨架

以下结构为所有模块的共同锚点，设计阶段已确认，实现时不应偏离：

```python
class Store(dict):
    _debug:         bool
    _type_registry: dict[str, type]  # flag 注解 → 类型，debug 模式下 __setitem__ 检查

@dataclass
class CompiledLabel:
    name:         str           # 热重载时按名字匹配
    filename:     str
    instructions: list[Instruction]
    constants:    list[Any]
    params:       list[str]

@dataclass
class CallFrame:
    label_ref: CompiledLabel    # 间接引用，不拷贝指令列表
    pc:        int
    locals:    dict

@dataclass
class Track:
    name:        str | None
    frame:       CallFrame
    interactive: bool
    done:        bool

class Scheduler:
    active_tracks:   list[Track]        # 普通 track 按声明顺序，interactive 永远最后
    transform_table: dict[int, list]    # 显示对象 id → transform 列表

    def tick(dt: float) -> list[UICommand]: ...
    def active_label_files() -> set[str]: ...   # 热重载查询接口
    def has_active_parallel() -> bool: ...      # 热重载查询接口

class HotReloader:
    _pending_reloads: list[str]         # 待处理文件路径，帧末批量处理

    def enqueue(filepath: str): ...     # 文件监听回调，只入队
    def flush(): ...                    # 帧末调用，分级处理
```

---

### 热重载系统

热重载是运行时注入能力的核心价值，从 MVP 阶段即需支持，不是事后插入的功能。CallFrame 的间接引用设计正是为此服务。

#### 热重载边界

| 内容 | 支持 | 说明 |
|------|------|------|
| `.apy` 脚本文件 | ✅ | label 级重编译，PC 重置到 label 入口 |
| 资源文件（图片/音频） | ✅ | 下次引用时重新加载 |
| Python 模块 | ✅ | `importlib.reload()` |
| 原生库（.so / .dll） | ✅ | dlclose + 重新 dlopen，调用方需确保无残留引用 |
| Store 结构 / flag 声明 | ❌ | 变量注册表变更影响存档兼容性，需重启 |
| `define` 角色定义 | ❌ | 符号表基础，运行时替换会导致状态混乱，需重启 |
| 引擎核心（engine.py） | ❌ | 需重启 |

#### 文件监听

使用 `watchdog` 库，不自己实现。防抖合并是必须的——编辑器保存时通常触发多次 `on_modified`：

```python
class AxnFileWatcher(FileSystemEventHandler):
    def on_modified(self, event):
        if event.src_path.endswith('.apy'):
            self._pending.add(event.src_path)   # 去重入队，帧末统一处理

    def flush():
        """帧末调用，批量处理变更"""
        ...
```

#### .apy 热重载流程

1. 文件变更入队（`enqueue`）
2. 帧末 `flush`，检查是否有 parallel 块正在执行
3. 不影响当前执行的文件：立即走增量 parser 重解析 → 重编译变更 label → 替换 `label_index` → 找到相关 CallFrame 替换 `label_ref` + 重置 PC
4. 影响当前执行文件的请求：推迟到 parallel 结束后自动处理
5. 编辑器显示"N 个文件变更等待热重载"计数，处理完毕后消失

**PC 重置到 label 入口**是最保守也最安全的策略。视觉小说场景下开发者通常就是想从头看改过的那段，重置是期望行为，不尝试保持执行位置。

#### 原生库热重载

原生库热重载是放弃 iOS/Web 的根本原因，也是与 Ren'Py 最大的差异点。

```python
class NativeLibReloader:
    _loaded: dict[str, ctypes.CDLL]

    def reload(self, lib_path: str) -> ctypes.CDLL:
        if lib_path in self._loaded:
            self._unload(lib_path)      # 先卸载旧库
        new_lib = ctypes.CDLL(lib_path)
        self._loaded[lib_path] = new_lib
        return new_lib
```

**`dlclose` 不保证立即卸载**：Linux 下如果有其他代码持有旧库的符号引用，`dlclose` 只是减引用计数，不会真正卸载。开发者必须确保旧库的所有函数调用结束后再 reload。此约束在文档和错误信息中明确说明。

#### 关键设计决策（热重载）

**热重载从 MVP 即支持**：这要求 `CallFrame` 从第一天起就持有间接引用而非内联指令列表，`CompiledProject.label_index` 从第一天起就是可变 dict，`Scheduler` 从第一天起就暴露 `active_label_files()` 和 `has_active_parallel()` 接口。事后插入会导致大规模重构。

**文件监听使用 `watchdog`**：不自己实现，防抖合并到帧末批量处理。

**parallel 期间分级处理**：不影响当前执行的文件立即 reload，影响当前执行文件的请求推迟，不阻塞整个热重载系统。

**Store / define / 引擎核心不支持热重载**：边界必须明确告知开发者，不能让其猜测。尝试热重载这些内容时引擎输出明确错误，提示需要重启。

---

## 不是什么

- 不是 Ren'Py 的分支或 fork
- 不是面向零编程基础用户的工具（引擎本身）
- 不追求最大化跨平台覆盖
