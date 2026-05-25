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

**作用域模型**：引擎维护一个全局 `store` dict，贯穿整个游戏生命周期。所有 Python 块的 `exec()` 调用共享同一个内部 exec globals dict（`_exec_globals`），变量天然跨 label、跨 jump 持久化。`Store` 作为该 dict 的代理层，对外暴露游戏变量的读写接口；序列化时过滤所有 dunder key（`__builtins__`、`__name__` 等 `exec()` 自动写入的内部符号），保证存档不被污染。引擎内置符号（`show`、`jump` 等 API）通过 `__builtins__` 注入为只读层，用户代码可见但不可覆盖。

**`store` 是全局单例，跨文件共享。** 在任意 `.apy` 文件中通过 `$` 或 `python:` 块写入的变量，在所有其他文件中可直接读取，无需 `import`。`store` 没有文件级命名空间，开发者需自行管理变量命名以避免冲突。

### 解析器模型

**三遍扫描**：解析器分三遍处理所有 `.apy` 文件，均在引擎启动时一次性完成，不在运行时重复。

- **第一遍**：轻量扫描，只收集所有 `define` 的名字，不解析字段内容和 `extends` 关系。目标是建立全局名字集合，使第二遍能识别跨文件引用。
- **第二遍**：扫描所有 `define extends` 关系，建立继承有向图，拓扑排序确定解析顺序。检测循环继承并在此阶段报错（`AxnParseError: circular inheritance`）。同时建立全局符号表（角色名 → 类型），供第三遍使用。
- **第三遍**：按拓扑顺序完整解析所有文件，展开继承字段，构建完整 AST。行首遇到已知角色名走对话路径，否则走指令路径。label 冲突检查也在此阶段完成。

`define` 只允许出现在文件顶层，保证第一、二遍扫描无需理解嵌套结构，各遍成本极低。第一、二遍合计开销相对于第三遍可忽略不计；瓶颈始终在第三遍的 AST 构建。对未修改的文件可跳过第三遍，直接使用缓存的 AST，进一步降低重启开销。

**为什么需要三遍而不是两遍**：引入 `define extends` 后，跨文件的继承依赖关系在解析前无法确定顺序——父类定义必须先于子类解析，但依赖关系本身要解析才能知道。三遍扫描将"收集名字"、"建立依赖图"、"完整解析"三个阶段显式分离，避免了解析顺序的不确定性。Ren'Py 通过不支持 `define` 继承来回避这个问题，Axn-Plus 选择支持继承，对应的代价是多一遍轻量扫描。

**`$` 行括号续行**：`$` 后的内容在遇到换行符时理论上应当终止，但实际上括号不平衡时引擎不阻止运行，而是在解析时输出警告并继续处理——括号续行被当作多行表达式接受，行为在某些上下文中可能未定义。不推荐此写法，推荐改用 `python:` 块。可在 `options_window.apy` 中设置 `ignore_multiline_dollar = true` 静默此警告。

```
AxnWarning: [parser] '$' line contains multi-line expression.
  Bracket continuation is allowed but not recommended.
  Behavior may be undefined in some contexts.
  → scene.apy, line 12

Hint: Use a 'python:' block for multi-line expressions.
  Add 'ignore_multiline_dollar = true' in options_window.apy to suppress this warning.
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
    # 归属 morning_scene（父节点），不归属下面的 autumn 对话行
    autumn: "早上好。"  # 归属此对话行（行内注释）
    sophia: "你好。"

# 归属 morning_scene（父节点），空行切断与下方 scene 的联系

scene bg_room
```

GUI 处理：节点移动时注释跟随；节点删除时若有关联注释则弹出确认；注释在编辑器中显示为节点上方的灰色文本框，可编辑。

### 基础语法

```apy
# 角色定义（静态声明，内联）
# define 默认推断类型为 char；显式声明可用 define char
define autumn:
    name "autumn"
    color #ff8800
    sprites "assets/autumn/"
    voice_prefix "vo/autumn/"
    default_expression "neutral"
    side_image "ui/autumn_side.png"
    font "fonts/handwriting.ttf"
    type_sound "sfx/type_autumn.ogg"
    dialogue_box "ui/autumn_box.apy::autumnBox"

define char autumn:   # 显式声明，等价于上方写法，意图更清晰

# 角色继承：子角色继承父角色所有字段，显式声明的字段覆盖父定义
define char autumn_adult extends autumn:
    sprites "assets/autumn_adult/"
    voice_prefix "vo/autumn_adult/"
    # 未声明字段全部继承：name、color、dialogue_box 等保持不变

# layers 模型下的继承：同名动态层内按 key 合并，未声明的状态继承父定义
define char autumn_casual extends autumn:
    layers:
        outfit:
            casual "outfit_casual_v2.png"   # 只覆盖此状态
            # school 继承父定义

# 继承规则：
# - 支持链式继承（A extends B extends C），但引擎启动时输出警告，可 ignore
# - 链式继承字段展开顺序为从根到叶，子类覆盖父类；建议保持单层继承以维持可读性
# - 继承只发生在编译期展开，运行时 autumn_adult 与 autumn 是完全独立的对象
# - show autumn_adult 和 show autumn 互不影响
# - 运行时修改 autumn 的层状态（表情、换装等），autumn_adult 完全不受影响，反之亦然
# - layers 模型下的 key 合并也发生在编译期，运行时两个角色的层状态互不共享

# 分层立绘：states 和 layers 二选一，不可混用
# 混用时引擎启动报错：
#   AxnParseError: Cannot use both 'states' and 'layers' in the same 'define char'.
#     → characters.apy, line 8 (define char autumn)
# states：整图切换模型
define char sophia:
    name "Sophia"
    states:
        neutral  "sophia_neutral.png"
        happy    "sophia_happy.png"
        sad      "sophia_sad.png"
    default_expression "neutral"

# layers：分层叠加模型
define char autumn:
    name "autumn"
    sprites "assets/autumn/"
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

# 响应式 triggers：角色可见时监听 store 变量，条件满足时自动触发动画
# 只在角色可见时监听，hide 后自动暂停，show 后恢复
define char autumn:
    name "autumn"
    layers:
        ...
    triggers:
        when store["hp_ratio"] < 0.2:
            transform breathe_heavy             # 替换当前 transform
        when store["relationship"]["autumn"] >= 80:
            transform += glow_soft              # 追加 transform，保留现有
        when store["day"] != store["prev_day"]:
            call animation day_change           # 触发 animation

# triggers 规则：
# - 每帧检查条件，false → true 时触发一次，不重复触发
# - 条件只允许 store 变量的简单比较（==、!=、>、<、>=、<=），不允许函数调用
# - 回滚后清空所有 trigger 的触发状态，等下次 false → true 才重新触发
# - 复杂条件退回 python: 块 + on key / on enter 手动处理

# expression 指令：无对话时切换表情（states 和 layers 模型均支持）
expression autumn happy                      # 走 expressions 映射（layers 模型）或整图切换（states 模型）
expression autumn (face=happy, brow=angry)   # 直接指定各层，绕过 expressions 映射（仅 layers 模型）
expression autumn (outfit=casual)            # 换装（仅 layers 模型）
expression autumn happy (transition=dissolve) # 带过渡效果

# 角色对话，表情作为行内修饰符；括号内支持裸关键字（布尔 flag）和具名参数
autumn: "你好。" (happy)
autumn: "今天天气不错。" (speed=0.5, voice="vo/001.ogg", nowait)
# layers 模型下可直接指定各层（绕过 expressions 映射）
autumn: "……" (face=happy, brow=angry)
autumn: "换装了。" (outfit=casual)
# 表情状态跟着角色对象走，不跟场景走：
# 对话修饰符修改角色的持久表情状态，与角色是否可见无关
# show 出场时使用角色当前表情状态，不重置为 default_expression

# voice 短路径：相对 voice_prefix 的短路径，引擎自动补全扩展名
# 等价于 voice="vo/autumn/001.ogg"（假设 voice_prefix = "vo/autumn/"）
autumn: "你好。" (voice="001")

# 旁白（单行两种等价写法，风格自选，同一项目内保持一致）
@ "阳光透过窗户照进来。"
narrator: "阳光透过窗户照进来。"     # 与角色行对齐，可读性更好
# 连续旁白使用 narrate: 块（见下文）

# 位置与可见性（show 不控制表情）
# 指令结构：动词 [子命令] [位置参数...] (具名参数...)
# show 位置参数顺序：角色 → 位置 → duration
show autumn left                                        # 最简写法
show autumn left 0.3                                    # 指定 duration
show autumn left 0.3 (enter=slidein_left)               # 补全具名参数
show autumn (layer=effect)                              # 显式指定层，默认为 sprite 层

# 动画句柄：推荐用 handle= 具名参数（推荐写法）
show autumn left 0.3 (enter=slidein_left, handle=anim_autumn)
# 保留 as 句柄写法，但引擎运行时输出警告，可 ignore
show autumn left 0.3 (enter=slidein_left) as anim_autumn  # AxnWarning: 建议改用 handle=

# 多实例：同一角色同时出现在多个位置，用 alias= 命名独立实例
# 不同实例的位置、表情、transform 状态完全独立，互不影响
show autumn left (alias=left_autumn)
show autumn right (alias=right_autumn)
show autumn (transform=ghost, alias=ghost_autumn)
hide left_autumn                    # 按 alias 名操作具体实例
expression left_autumn happy        # 按 alias 名修改表情
# 同样保留 as 写法但警告：show autumn left as left_autumn  ← AxnWarning

# hide 位置参数顺序：角色 → duration
hide autumn                                             # 立即隐藏
hide autumn 0.5                                         # 指定 duration
hide autumn 0.5 (exit=fadeout)                          # 补全具名参数

# 多角色并行：同行逗号分隔 = 并行执行，换行 = 串行执行
show autumn left 0.3 (enter=slidein, handle=anim_autumn), sophia right 1.0 (enter=slideout, handle=anim_sophia)

# 等待控制
wait                    # 等用户点击
wait 2.0                # 等 2 秒
wait for anim_autumn    # 等特定动画完成
wait for all            # 等所有动画完成（默认行为，显式写出意图更清晰）
wait for any            # 等最先完成的动画

# 场景切换
# scene 默认同时清空 sprite 层（高频用法零开销）
# scene 位置参数顺序：路径 → duration
scene bg_room                           # 切换背景，默认清空 sprite 层
scene bg_room 0.5                       # 指定 duration
scene bg_room 0.5 (with=fade)           # 补全具名参数
scene bg_room (keep)                    # 保留所有立绘
scene bg_room (keep=autumn)             # 只保留 autumn，其余清除
scene bg_room (keep=[autumn, sophia])   # 保留多个

# transition：全屏过渡，不切换任何内容（替代 Ren'Py 独立 with 语句的用途）
# 位置参数顺序：过渡名 → duration
transition fade 1.0             # 全屏渐黑再渐亮
transition fade_black 0.5       # 直接渐黑（不再亮）
transition fade_white 0.3       # 渐白
transition dissolve 0.5         # 全屏 dissolve
# 内置过渡名与 show/hide/scene 的 enter/exit/with 参数共享同一套过渡库

# clear：精确清除，无过渡，不受 scene 影响
clear                           # 清除 sprite 层所有元素
clear autumn                    # 清除指定角色
clear autumn sophia             # 清除多个
clear (layer=effect)            # 清除指定层所有元素
clear autumn (layer=effect)     # 清除指定层上的指定元素

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
# play music / ambient 默认 loop（视觉小说 BGM 和环境音绝大多数需要循环）
# play sound / voice 默认不循环（音效和语音通常单次播放）
# 需要单次播放时显式加 (once)，需要循环时显式加 (loop)——与各自默认相反时才需要声明
# play 位置参数顺序：路径 → volume → fadein → fadeout
play music "bgm/morning.ogg"                            # 全默认，自动循环
play music "bgm/morning.ogg" 0.8                        # 指定 volume，自动循环
play music "bgm/morning.ogg" 0.8 1.0                    # volume + fadein，自动循环
play music "bgm/ending.ogg" (once)                      # 片尾曲，只播一次
play music "bgm/ending.ogg" 0.8 1.0 (once)             # 带参数的单次播放
play sound "sfx/door.ogg" 0.6                           # 音效，默认单次
play sound "sfx/heartbeat.ogg" (loop)                   # 音效循环（显式声明）

# queue：将音频加入通道队列，当前播放结束后自动播放
# 位置参数顺序与 play 一致：路径 → volume → fadein → fadeout
# queue music / ambient 同样默认 loop
queue music "bgm/day.ogg"
queue music "bgm/night.ogg" 0.6
queue music "bgm/ending.ogg" (once)                     # 队列中的单次播放

# stop 位置参数顺序：fadeout
stop music                              # 立即停止
stop music 1.0                          # 指定 fadeout

# pause / resume（保留进度暂停，与 stop 语义不同）
# pause 位置参数顺序：fadeout
pause music                             # 立即暂停
pause music 0.3                         # 带 fadeout 的暂停（音量渐弱）
resume music                            # 立即恢复
resume music 0.3                        # 带 fadein 的恢复
pause video
resume video 0.3

# pause（游戏进程控制，区别于音视频的 pause/resume）
# 全局冻结：时间停止，所有 transform、timer、动画全部冻结
pause                               # 冻结所有，等点击推进
pause 3.0                           # 冻结，3 秒后自动推进，点击可提前
pause until flag_ready              # 冻结，条件满足后推进
pause (freeze_audio=False)          # 冻结画面和动画，音频继续

# pause transform：单个 transform 暂停，保留进度
show autumn (transform=breathe, handle=anim_autumn)
pause transform anim_autumn         # 暂停，保留当前帧进度
resume transform anim_autumn        # 从暂停帧继续

# freeze / unfreeze：单个控件冻结（不响应输入）
freeze hud_button                   # 单个控件冻结，自动应用 disabled 样式
unfreeze hud_button
freeze (layer=effect)               # 冻结整个层上的所有控件
unfreeze (layer=effect)

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
camera follow autumn                    # 跟随 autumn 的位置
camera follow autumn (lag=0.3)          # 带延迟跟随，更自然
camera follow none                      # 取消跟随

# 层管理
layer create effect (above=sprite)      # 创建层，指定位于 sprite 层之上
layer destroy effect                    # 销毁层
layer order sprite effect ui            # 重排层顺序（从下到上）

# say 动词（专用于说话者在运行时动态决定的场景）
# 静态说话者必须使用 角色: 或 @，用 say 传入静态角色名时解析期报错：
#   AxnParseError: 'say' with static character name 'autumn'.
#     Use 'autumn: "..."' syntax instead.
#     → scene.apy, line 8
$ speaker = get_current_speaker()
say speaker "动态说话者。"              # speaker 是 store 变量，运行时求值
say speaker "下一句。" (happy)          # 修饰符与对话行完全一致

# choice（动态菜单，程序化生成选项列表）
# menu 是静态声明语义（编译期确定），choice 专门处理动态场景
$ options = build_options(day, relationship)
choice options (timeout=10.0)

# 截图
screenshot                              # 保存到 options_window.apy 配置的默认路径
screenshot "screenshots/ch1_end.png"   # 指定路径
screenshot (include_ui=true)            # 包含 UI 层（默认 false）

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

# 对话框窗口控制
window show              # 显示对话框容器
window hide              # 隐藏对话框容器；当有对话行时，文字按 hide_behaviour 配置策略降级渲染
window hide (all)        # 隐藏整个对话系统（框 + 文字）；此形式通过 mode= 参数控制行为
window hide (all, mode=pause)   # 遇对话行挂起，window show 后继续显示（默认）
window hide (all, mode=skip)    # 对话行静默跳过，不计入历史，执行流不停
window auto              # 有对话时自动 show，无对话时自动 hide（推荐默认）

# window hide（无 all）的降级行为在 options_window.apy 中全局配置：
# engine:
#     window:
#         hide_behaviour = "overlay"   # 文字叠加在场景上（无背景框），默认
#         hide_behaviour = "drop"      # 丢弃当前对话内容，静默跳过
#         hide_behaviour = "error"     # 抛出 AxnWarning，可 ignore
#         hide_behaviour = "queue"     # 对话排队，window show 后继续
#
# hide_behaviour 配置只对 window hide（无 all）生效；
# window hide (all) 的行为由 mode= 参数直接控制，与 hide_behaviour 无关

# 鼠标光标控制
# 命令式：脚本流程中随时切换，立即生效
cursor "assets/cursor/sword.png"    # 切换为自定义图片光标
cursor default                       # 恢复全局默认光标
cursor none                          # 隐藏光标

# 变量式：通过 store 变量驱动，适合状态联动
$ engine.cursor = "assets/cursor/pointer.png"
$ engine.cursor = "default"
$ engine.cursor = "none"

# 样式绑定：控件级，鼠标进入控件时自动切换，离开时恢复
button "攻击" (cursor="assets/cursor/sword.png") on_click: jump battle
image "map/region_a.png" (cursor=hover, on_click: jump region_a)   # 引用 options_window 中定义的具名cursor

# 优先级（从高到低）：
# 变量 engine.cursor > cursor 指令 > 控件 cursor= 参数 > options_window.apy 全局默认

# pass（空语句，什么都不做，用于占位）
pass

# 单行 Python（单行内 Python 语法合法即可）
$ flag_met_autumn = True

# 多行 Python
python:
    for item in inventory:
        item.apply()

# while 循环（rollback= 必须显式声明，否则解析器报错）
# 缺少 rollback= 时：
#   AxnParseError: 'while' block requires explicit 'rollback=' parameter.
#     Rollback behavior is ambiguous for loops containing dialogue.
#     Recommended: (rollback=none) for most cases.
#     → scene.apy, line 8
while hp > 0 (rollback=none):
    autumn: "还没结束！"
    $ hp -= 10

# for 循环（rollback= 必须显式声明，理由相同）
for item in inventory (rollback=none):
    autumn: "我持有了{item.name}。"

# break / continue（仅在 while / for 块内有效）
# 出现在块外时：
#   AxnParseError: 'break' is only valid inside a 'while' or 'for' block.
#     → scene.apy, line 12
while True (rollback=none):
    $ hp -= 10
    if hp <= 0:
        break
    if hp > 80:
        continue
    autumn: "还能撑住。"

# 不需要对话行的循环推荐退回 python: 块
python:
    for item in inventory:
        item.apply()

# 标签（默认动态）；label 签名直接使用 Python 函数签名风格
label morning_scene:
    autumn: "早上好。" (smile)

label morning_scene(mood, weather="sunny"):
    autumn: "早上好。"
    return mood + "_done"   # return 后跟任意 Python 表达式

# 局部 label：. 前缀，只在当前文件内可见，不进全局符号表
# 适合模块内部跳转点，避免 loop_start / check_condition 等通用名污染全局命名空间
label chapter1_battle:
    call .intro
    jump .main_loop

label .intro:              # 局部 label，只在声明所在文件内可见
    autumn: "战斗开始！"
    return

label .main_loop:
    $ hp -= 10
    if hp <= 0:
        jump .ending
    jump .main_loop

label .ending:
    autumn: "结束了。"
    return

# 跨文件访问局部 label：使用显式路径语法
call chapter1.apy::.intro       # 显式路径自然扩展，与现有跨文件引用语法一致

# 局部 label 规则：
# - . 前缀 = 局部 label，Parser 第一遍扫描时识别，不写入全局符号表
# - 只在声明所在文件内可见，同文件内直接用 . 前缀名调用
# - 不同文件可以有同名局部 label（如 .intro），互不冲突
# - 跨文件访问必须用显式路径：文件名::.label名
# - label 冲突检查只针对全局 label，局部 label 不参与全局冲突检测
# - GUI 编辑器在脚本区以缩进或折叠形式展示局部 label，视觉上归属所在文件

# 条件（支持 elif 链）
if flag_met_autumn:
    autumn: "好久不见。"
elif flag_heard_of_autumn:
    autumn: "久仰大名。"
else:
    autumn: "初次见面。"

# unless：if not 的语法糖，用于卫语句场景
unless flag_met_autumn:
    jump prologue

# match：多路路由，匹配 store 变量的单一值
# 简单形式（GUI 可完整解析为多路分支节点）
match day:
    1       -> day1_scene
    2       -> day2_scene
    3, 4    -> midgame_scene
    _       -> ending

# match 复杂形式（含表达式或 guard，整块降级为代码节点）
match relationship["autumn"]:
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
    "询问详情" (id="ask", if=flag_met_autumn, disabled=flag_tired):
        jump route_c
    "隐藏选项" (id="secret", hidden=flag_secret):
        jump route_secret

# menu 内联跳转：选项只有一条 jump 时可用 -> 省略展开块
menu:
    "答应她" (if=flag_can_agree) -> route_a
    "拒绝"                       -> route_b
    "询问详情" (if=flag_met_autumn):   # 有额外逻辑时退回展开块
        $ log_choice("ask")
        jump route_c

# menu as：选完后继续执行流，选项 -> 右边是返回值表达式而非 label 名
menu as answer:
    "是"  -> "yes"
    "否"  -> "no"

autumn: "你选了 {answer}。"

# menu as 展开块：选项有前置逻辑时用 -> 显式 return
menu as answer:
    "答应她":
        $ log_choice("agree")
        -> "yes"        # 块内显式返回值
    "拒绝" -> "no"      # 简单场景仍用内联写法

# menu as 内不允许 jump，混用时解析器报错：
#   AxnParseError: 'jump' is not allowed inside 'menu as'.
#     'menu as' collects a return value; use 'menu' for flow branching instead.
#     → scene.apy, line 8
# GUI 处理：menu as 对应独立的"菜单返回值"节点，与跳转型菜单节点分开

# with char：连续对话锁定角色和默认修饰符
# 块内裸字符串自动归属当前角色；行级修饰符按槽位覆盖块级默认值（表情槽、具名参数槽、Flag 槽各自独立）
with autumn (happy):
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
jump prologue unless flag_met_autumn
return unless flag_can_continue

# 不支持 call ... as result if ...（条件不满足时返回值语义不明），退回 if 块处理
# GUI 处理：条件跳转节点显示为带条件标签的跳转箭头，视觉权重轻于完整 if 块

# call 不接返回值
call morning_scene("happy")

# call 用 as 接返回值（推荐写法）
call morning_scene("happy") as result

return
```

### `define image`（非角色分层图片对象）

`define char` 解决角色立绘的分层需求，`define image` 解决**非角色**可显示对象的分层需求——道具、背景元素、环境特效、装饰图等。不需要单开一个角色，也不携带任何角色专属字段（`voice_prefix`、`dialogue_box`、`side_image` 等）。

```apy
# 静态图片别名（最简形式，进符号表，show 可用裸名字）
define image bg_room = "assets/bg/room.png"

# states 模型：整图切换
define image weather_overlay:
    states:
        clear "fx/clear.png"
        rain  "fx/rain.png"
        snow  "fx/snow.png"
    default_expression "clear"

# layers 模型：分层叠加（核心用途）
define image lantern:
    layers:
        base    "props/lantern_base.png"
        glow:
            bright "props/lantern_glow_bright.png"
            dim    "props/lantern_glow_dim.png"
        flicker:
            active "props/lantern_flicker.png"
            none   "props/lantern_static.png"
    expressions:
        on:  (glow=bright, flicker=active)
        off: (glow=dim,    flicker=none)
    default_expression "off"

# triggers 同样支持（响应 store 变量）
define image alarm_light:
    layers:
        light:
            red  "fx/alarm_red.png"
            off  "fx/alarm_off.png"
    triggers:
        when store["alert_level"] >= 3:
            transform blink_red
```

使用方式与 `show` 完全一致，`expression`、`hide`、`alias=` 均适用：

```apy
show lantern right
expression lantern on
expression lantern (glow=bright, flicker=none)   # 直接指定各层
show lantern (alias=lantern2, transform=sway)    # 多实例
hide lantern 0.5 (exit=fadeout)
```

**`define image` 与 `define char` 的边界：**

| | `define char` | `define image` |
|---|---|---|
| 对话行 | ✅ | ❌ 解析期报错 |
| 语音相关字段 | ✅ | ❌ 不支持 |
| `layers` / `states` | ✅ | ✅ |
| `expressions` | ✅ | ✅ |
| `triggers` | ✅ | ✅ |
| `alias=` 多实例 | ✅ | ✅ |
| `define extends` | ✅ | ✅（只能继承同类型） |
| 跨类型继承 | ❌ 解析期报错 | ❌ 解析期报错 |
| GUI 符号表分类 | 角色 | 图片对象 |

`define image` 之间可以 `extends`，不能继承 `define char`，反之亦然。跨类型继承解析期报错：

```
AxnParseError: Cross-type inheritance is not allowed.
  'lantern_special' is 'define image' but extends 'autumn' which is 'define char'.
  → characters.apy, line 42
Hint: 'define image' can only extend other 'define image'. Use a separate 'define image' as base.
```

**静态别名与 `const` 的区别：**

```apy
define image bg_room = "assets/bg/room.png"   # 进符号表，show bg_room 用裸名字
const BG_ROOM = "assets/bg/room.png"          # 不进符号表，show $BG_ROOM 需要 $ 前缀
```

单行别名形式主要价值是路径变更时只改一处，并让资源名进入符号表统一管理。分层形式是 `define image` 的核心用途。

### 扩展语法

#### 文本标签系统

对话行内使用 `<tag>` 语法插入内联格式，与 `{expr}` 插值双轨并行，两者在 `TextRenderer` 层统一解析，互不冲突。

```apy
autumn: "这个字<b>很重要</b>，<color=#ff0000>注意</color>。"
autumn: "稍等……<w=1.5>好了。"           # 中途暂停 1.5 秒后继续
autumn: "我叫<nw>"                       # 说完立刻推进，不等点击
autumn: "<fast>直接显示完整文本。"        # 跳过打字机效果
autumn: "音量<alpha=0.5>渐弱</alpha>的字。"
autumn: "内联图标：<image=icon/heart.png>"
autumn: "你好，{player_name}！"          # 插值保持不变，无冲突
```

**内置标签列表：**

| 标签 | 功能 |
|------|------|
| `<b>` / `</b>` | 粗体 |
| `<i>` / `</i>` | 斜体 |
| `<color=#rrggbb>` / `</color>` | 颜色 |
| `<size=N>` / `</size>` | 字号（px） |
| `<alpha=N>` / `</alpha>` | 透明度（0.0–1.0） |
| `<w>` / `<w=N>` | 中途等待点击 / 等待 N 秒后继续 |
| `<p>` / `<p=N>` | 段落暂停（清屏后等待点击 / N 秒） |
| `<nw>` | 说完不等点击，直接推进执行流 |
| `<fast>` | 跳过打字机效果，直接显示完整文本 |
| `<image=path>` | 内联图片（表情图标等） |
| `<s>` / `</s>` | 删除线 |
| `<u>` / `</u>` | 下划线 |
| `<speed=N>` / `</speed>` | 句内打字机速度倍率（`<speed=2>` 为两倍速，`<speed=0.5>` 为半速；与行级 `speed=` 修饰符正交，两者相乘生效） |
| `<rb>` / `<rt>` / `</rb>` | Ruby 注音（`<rb>漢<rt>かん</rt>字<rt>じ</rt></rb>`） |

**`<w>` / `<nw>` 对 VM 执行模型的影响：**

一行对话可能产生多个等待点，`DIALOGUE` 指令内部维护状态机：

- `<w>` 产生中途 `WAIT_CLICK`，继续渲染后续文字
- `<nw>` 渲染完毕后不产生 `WAIT_CLICK`，直接推进
- 回滚边界：`<w>` 产生的中途等待点**不作为**回滚检查点，整行对话作为一个回滚单元

**不计入历史记录的标签：**

`<nw>` 行和 `<fast>` 行正常计入历史。若需要某行完全不计入：

```apy
autumn: "这句话不会进历史记录。" (no_history)
```

---

#### 回滚系统

采用分级策略，按 label 声明回滚行为，不做全量状态快照链：

```apy
label morning_scene (rollback=dialogue):    # 默认，只回滚对话显示状态
    autumn: "早上好。"

label choice_scene (rollback=checkpoint):   # 回到最近 checkpoint 的完整状态
    menu:
        "选项A" -> route_a
        "选项B" -> route_b

label cutscene (rollback=none):             # 完全禁止回滚
    play video "cutscene/intro.mp4"
```

| 策略 | 回滚内容 | Python 状态 | 开销 | 适用场景 |
|------|---------|------------|------|---------|
| `dialogue`（默认） | 当前对话行、角色表情、背景 | **不回滚** | 极低 | 普通对话段 |
| `checkpoint` | 最近存档点的完整状态 | 回滚到存档点 | 中 | 含重要选择的场景 |
| `none` | 禁止回滚 | — | 零 | 过场动画、不可撤销操作 |

`dialogue` 模式下 Python 状态变更不回滚，这是明确的设计取舍——大多数玩家回滚只是想重读上一句话。此行为在文档和编辑器中均有明确标注。

---

#### 自动推进与跳过模式（Auto / Skip Mode）

**Auto 模式**

自动推进：语音播完后（或无语音时等待指定时长），引擎自动推进对话，无需点击。

```apy
auto on          # 开启 Auto 模式
auto off         # 关闭 Auto 模式
auto toggle      # 切换
```

| 情况 | 行为 |
|------|------|
| 有语音 | 语音播完后等 `preferences.auto_delay` 秒再推进 |
| 无语音 | 等 `preferences.auto_forward_time` 秒后推进 |
| 遇到 `menu` / `choice` | 强制暂停，等用户选择 |
| 遇到 `pause (hard)` | 强制暂停 |
| 遇到 `input disable` 块 | Auto 被屏蔽，块结束后自动恢复 |
| `<nw>` 行 | 不等待，立即推进（与手动模式一致）|
| `<w>` 中途等待点 | 等满 `auto_forward_time` 后继续当前行（不推进到下一行）|
| `parallel` interactive track 内 | Auto 在 track 内正常生效 |

**`pause (hard)` 标记**：普通 `pause` 在 Auto/Skip 模式下被跳过；加 `(hard)` 后强制停止两种模式，等用户手动操作：

```apy
pause (hard)          # Auto/Skip 遇到此处强制停止
pause (hard) 3.0      # 手动模式等 3 秒；Auto/Skip 模式强制停止，不自动推进
```

**Skip 模式**

跳过模式：快速略过已读（或全部）对话。

```apy
skip on          # 开启 Skip 模式
skip off         # 关闭 Skip 模式
skip toggle      # 切换
```

Skip 策略通过 `preferences.skip_mode` 配置：

| 策略 | 行为 |
|------|------|
| `"seen"`（默认） | 只跳过已读对话，遇到未读对话停止 |
| `"all"` | 跳过全部对话，不区分已读/未读 |

**已读状态追踪**：引擎为每条对话行维护唯一标识（`文件名 + label名 + 行偏移` 的哈希），首次到达时写入 `persistent`，跨存档槽共享，只增不减。分支路径上的对话互不影响——只有玩家实际经过的行才标记为已读。

**Skip 停止条件：**

| 条件 | 是否停止 | 是否可配置 |
|------|---------|----------|
| `menu` / `choice` | 永远停止 | ❌ |
| 未读对话（`"seen"` 策略） | 停止 | — |
| `pause (hard)` | 永远停止 | ❌ |
| `modal show` | 停止 | ❌ |
| `input disable` 块 | 默认停止 | ✅ |
| `checkpoint` | 默认继续 | ✅ |

```apy
# options_window.apy
engine:
    skip:
        stop_at_checkpoint  = false   # true 时在 checkpoint 处停止跳过
        skip_transitions    = true    # 跳过时略过过渡动画
        skip_voice          = true    # 跳过时略过语音（不播放）
```

**Auto/Skip 与回滚**：Auto 模式下回滚正常工作；Skip 模式下回滚被禁用（方向相反，同时触发时 Skip 优先，回滚键临时停止 Skip 并回退一步）。

**Auto/Skip 快捷键**：在 `engine.keymap` 中配置（见定制按键映射章节），引擎默认 Auto 为 `a`，Skip（按住）为 `ctrl`。

**Round-Trip**：`auto on/off/toggle`、`skip on/off/toggle`、`pause (hard)` 对应脚本区独立积木块；编辑器工具栏显示 Auto/Skip 状态指示灯；`pause (hard)` 在积木块上以"强制停止"图标与普通 `pause` 视觉区分。

---

#### 对话历史（Backlog）

引擎层维护历史 buffer，UI 层由标准库模板提供。

```apy
# options_window.apy
engine:
    history:
        max_entries      = 200
        include_voice    = true     # 点击条目可重播语音
        include_narrator = true
        include_choices  = false    # 选项不计入历史（可选）
        persist          = false    # true = 存入 persistent 跨存档保留，false = 仅当次游玩
```

脚本层可显式标记不计入历史：

```apy
autumn: "这句话不会进历史记录。" (no_history)
```

---

#### NVL 模式

NVL（Novel）模式：对话文本在全屏或大区域内逐段**累积显示**，不像 ADV 模式每行替换上一行。适合大段叙事、纯文字向作品。

**角色声明**：在 `define char` 中设置 `mode "nvl"` 将该角色切换为 NVL 渲染路径：

```apy
define char narrator_nvl:
    name ""
    mode "nvl"                                        # 此角色走 NVL 渲染路径
    nvl_window "ui/nvl_box.apy::NVLBox"              # NVL 窗口模板（不填用引擎默认）
```

**脚本层控制**：

```apy
# nvl: 块 —— 块内所有对话累积显示（块内任意角色均走 NVL 路径，无论其 mode 声明）
nvl:
    autumn: "第一段话。"
    sophia: "第二段话，紧接在上面。"
    @ "旁白也可以参与。"
    autumn: "继续累积。"

nvl clear          # 清除 NVL 区域内的所有文本（不退出 NVL 模式）
nvl hide           # 退出 NVL 模式，对话框恢复 ADV 布局
```

**混用 ADV 和 NVL**：`nvl:` 块内走 NVL，块外走 ADV，同一 label 内可交替：

```apy
autumn: "ADV 对话，正常替换显示。"

nvl:
    autumn: "进入 NVL，开始累积。"
    @ "旁白段落。"
    sophia: "第三段。"

nvl clear
autumn: "NVL 清屏后回到 ADV。"
```

**修饰符**：NVL 块内对话行修饰符与 ADV 完全一致，支持表情、语音、`no_history`、`speed` 等。

**回滚**：整个 `nvl:` 块作为回滚粒度——回滚时清屏并重显整块内容。

**存档**：`nvl:` 块内不允许 `checkpoint`（编译期警告），存档须在块外触发：

```
AxnWarning: [nvl] 'checkpoint' inside an 'nvl:' block.
  NVL blocks cannot be safely checkpointed mid-block.
  Move the 'checkpoint' to before or after the 'nvl:' block.
  → scene.apy, line 18 (inside nvl: block starting at line 12)
```

**`options_window.apy` NVL 配置**：

```apy
engine:
    nvl:
        max_entries    = 10       # NVL 区域最多显示条目数，超出时自动滚动
        line_spacing   = 8
        clear_on_scene = true     # scene 切换时自动 nvl clear
```

**Round-Trip**：`nvl:` 块对应脚本区 NVL 段落积木块；`nvl clear` / `nvl hide` 对应独立指令节点；`mode "nvl"` 在角色定义积木块中显示为渲染模式字段（下拉：`"adv"` / `"nvl"`）。

---

#### 气泡式台词（Speech Bubbles）

对话框以气泡形式附着在立绘旁边，位置随立绘屏幕坐标实时更新，适合多角色并排说话的演出场景。

**角色声明**：

```apy
define char autumn:
    name "autumn"
    dialogue_mode "bubble"     # "box"（默认，固定位置对话框）/ "bubble"（气泡跟随立绘）
    bubble:
        anchor    "top"                    # 气泡附着方向：top / bottom / left / right
        offset    (0, -20)                 # 相对锚点的像素偏移
        tail      "ui/bubble_tail.png"     # 气泡尾巴图片（可选）
        max_width 300                      # 最大宽度（px），超出自动换行
        padding   (12, 8)
        window    "ui/bubble.apy::Bubble"  # 自定义气泡模板（不填用引擎默认）
```

**运行时行为**：

- 气泡位置每帧根据立绘当前屏幕坐标重新计算
- 立绘不可见（`hide` 后）时，气泡退化为 `fallback_mode` 指定的行为，引擎输出警告
- `together` 块内多角色气泡同时显示，各自独立定位；重叠检测不由引擎处理，开发者自行用 `offset` 调整
- `camera move` / `camera follow` 期间气泡随立绘移动，无需额外处理

**`options_window.apy` 气泡配置**：

```apy
engine:
    bubble:
        fallback_mode = "box"    # 立绘不可见时退化为 box（默认）/ "hide"（不显示）
        z_order       = 50       # 气泡在 UI 层的 z_order
```

**`dialogue_mode` 混用**：同一 label 内允许 `"box"` 和 `"bubble"` 角色同时说话，引擎分别走各自渲染路径，无冲突。

**Round-Trip**：`dialogue_mode "bubble"` 在角色定义积木块中显示为模式下拉；`bubble:` 子块展示为气泡配置面板；编辑器预览中气泡位置以相对立绘的虚线框标注。

---

#### 动态指令（`$` 前缀）

`$` 前缀标记运行时从 store 求值，规则与 `say speaker` 一致——**`$` 前缀 = 运行时求值**，适用于 `show` / `hide` / `call` / `jump`：

```apy
$ sprite = get_current_sprite()
$ target = compute_next_label()

show $sprite center 0.3 (enter=fadein)    # 动态 show
hide $sprite 0.5                          # 动态 hide
call $target("happy")                     # 动态 call
jump $target                              # 动态 jump
```

静态版本保持不变，无歧义：

```apy
show autumn center      # 静态，编译期确定
jump morning_scene      # 静态，编译期确定
```

解析规则：动词后第一个 token 为 `$identifier` 时走动态路径，否则走静态路径。GUI 编辑器对动态节点显示变量名字段，作为代码节点处理。

`$` 前缀只接受单一 store 变量名，不接受任意表达式——需要复杂计算时先用 `$` 块算好再引用：

```apy
python:
    sprite = choose_sprite(day, relationship)
show $sprite center
```

---

#### `startup` 块（初始化优先级控制）

控制初始化代码的执行阶段，替代 Ren'Py 的 `init N:` 数字优先级：

```apy
startup:                    # 默认阶段
    python:
        # 启动阶段可执行任意初始化逻辑
        import logging
        logging.basicConfig(level=logging.DEBUG)

startup (before):           # 最早执行，适合库初始化、依赖注入
    python:
        def early_setup():
            pass

startup (after):            # 最晚执行，适合依赖其他模块的初始化
    python:
        register_extensions()
```

三个阶段执行顺序：`before` → 默认 → `after`。同阶段内按文件扫描顺序执行。`startup` 块只允许出现在文件顶层，不允许嵌套在 `label` 或其他块内。

---

#### 叙事表达

**对话插值**

对话字符串内用 `{expr}` 插入 `store` 变量或简单表达式，引擎在渲染时求值：

```apy
autumn: "你好，{player_name}。今天是第 {day} 天。"
autumn: "好感度：{relationship['autumn']}/100"
```

插值表达式限制为单一表达式（变量、属性访问、下标、简单运算）。禁止函数调用和赋值——需要复杂计算时先用 `$` 块算好再引用。推断失败时抛出 `AxnInterpolationError`，指明文件位置和表达式内容：

```
AxnInterpolationError: Invalid interpolation expression: 'get_name()'
  Function calls are not allowed in dialogue interpolation.
  → scene.apy, line 42
Hint: Pre-compute the value with '$ name = get_name()' then use '{name}' in the dialogue.
```

**条件文本（inline conditional）**

```apy
autumn: "我们[已经|还没]见过面。" (if=flag_met)
```

`[A|B]` 语法：`if` 条件为真取 A，否则取 B。省略 B 时条件为假则显示空字符串：

```apy
autumn: "你[（有点憔悴）]看起来不错。" (if=flag_tired)
```

`[A|B]` 仅支持静态字符串片段，不支持嵌套。需要复杂条件分支时用 `if` 块。

**多语言（translate）**

```apy
translate zh:
    autumn: "你好。"
    @ "阳光透过窗户照进来。"

translate en:
    autumn: "Hello."
    @ "Sunlight streams through the window."
```

`translate` 块内只允许对话行和旁白，不允许引擎指令或 Python 块。引擎根据运行时语言设置选择对应块；当前语言无对应翻译时回退到第一个 `translate` 块并输出警告。同一 label 内的 `translate` 块必须紧跟原始对话行之后，解析器在启动时检查完整性。

**翻译工具链**

`axn` CLI 提供翻译字符串提取命令，将项目中所有 `translate` 块的内容导出为模板文件，供翻译人员填充：

```
# 提取所有待翻译字符串，生成翻译模板
axn extract-strings --lang en --output strings/en.apy

# 已有翻译模板时更新（新增字符串追加，已翻译内容保留）
axn extract-strings --lang en --output strings/en.apy --update

# 检查翻译完整性（列出缺失和多余条目）
axn check-strings --lang en --input strings/en.apy
```

生成的模板文件为标准 `.apy` 格式，翻译人员直接编辑：

```apy
# strings/en.apy — 由 axn extract-strings 自动生成，手动编辑翻译内容

translate en:
    # scene.apy::morning_scene, line 42
    autumn: "Good morning."

translate en:
    # scene.apy::morning_scene, line 43
    @ "Sunlight streams through the window."
```

每个条目包含来源注释（文件、label、行号），方便翻译人员定位上下文。`--update` 时引擎按来源注释匹配已有翻译，源码行号变化时给出警告但保留翻译内容，源码文本变化时标记为需要重新翻译。

`axn build` 时对每种语言的翻译完整性做静态检查，缺失条目以警告（非错误）列出，不阻止构建。

---

#### 动画与演出控制

**具名动画序列（animation block）**

```apy
animation autumn_enter:
    show autumn right 0.0
    camera move 1.1 0.5
    wait 0.3
    show autumn center 0.4 (enter=slidein)

animation autumn_exit:
    hide autumn 0.4 (exit=slideout)
    camera reset 0.3
```

调用方式与 `call` 一致：

```apy
call animation autumn_enter
call animation autumn_enter (handle=anim)   # 用 handle= 接句柄，配合 wait for 使用
call animation autumn_enter as anim         # 兼容写法，运行时警告可 ignore
```

`handle=anim` 得到的是 `AnimationHandle` 对象，只暴露以下接口：

```python
anim.done        # bool，动画是否完成
anim.cancel()    # 立即停止动画
```

`wait for anim` 是引擎层语法糖，底层轮询 `anim.done`。不允许在 Python 块里直接操作 `AnimationHandle` 对象——此限制保证 GUI 能完整解析 `wait for` 的依赖关系。

`animation` 块内只允许引擎指令（`show`、`hide`、`camera`、`play`、`wait` 等），不允许 Python 块、对话行、`jump`、`menu`。目的是保证 GUI 能完整解析，也防止演出片段携带业务逻辑。违反时解析期报错：

```
AxnParseError: 'python:' block is not allowed inside an 'animation' block.
  Move business logic to a 'label', then call the animation from there.
  → animations.apy, line 12 (inside 'animation autumn_enter')

AxnParseError: 'jump' is not allowed inside an 'animation' block.
  'animation' blocks are for visual sequencing only.
  → animations.apy, line 8 (inside 'animation boss_enter')

AxnParseError: Dialogue line is not allowed inside an 'animation' block.
  Use a 'label' with an interactive track for mixed dialogue and animation.
  → animations.apy, line 6 (autumn: "..." inside 'animation boss_enter')
```

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
call animation char_enter(autumn, "left")
call animation char_enter(autumn, "left") (handle=anim)
call animation char_enter(autumn)          # 使用默认参数
```

参数类型限制：只允许角色名、位置关键字、数值、字符串字面量，不允许传入 Python 表达式，也不允许 `$` 前缀的动态变量——`animation` 块内不允许 Python，参数必须在编译期完全确定，保证 GUI 能完整解析调用点。调用点传入 `$` 前缀参数时解析期报错，需要动态参数时退回 `label` + Python 块处理。

**`animation` 条件分支**

`animation` 块内允许基于参数变量的简单条件分支，不允许引用 store 变量，保证 GUI 完整可解析：

```apy
animation char_enter(char, mood="neutral"):
    show char right 0.0
    if mood == "happy":
        show char (transform=bounce_in)
    elif mood == "sad":
        show char (transform=fade_in_slow)
    else:
        show char (transform=fade_in)
    wait for all
```

条件只允许参数变量的简单比较（`==`、`!=`、`>`、`<`），不允许 store 变量或函数调用。需要响应 store 状态时退回 `label` + Python 块处理。GUI 将条件分支解析为多路分支节点。

**`animation` 的 `yield` 暂停点**

`yield` 在 animation 执行流中插入一个命名暂停点，animation 暂停在此等待外部 `resume`，调用方的执行流继续向下运行：

```apy
animation boss_enter:
    show boss center 0.0 (transform=slam_down)
    wait for all
    yield "boss_landed"          # animation 暂停，调用方继续执行
    camera shake 10 0.5
    play sound "sfx/roar.ogg"
```

```apy
# 调用方
call animation boss_enter (handle=anim_boss)
# animation 暂停在 yield，调用方继续向下执行
autumn: "它来了！"
sophia: "快跑！"
resume animation anim_boss      # 手动恢复，camera shake + roar 在这里触发
```

**`yield` 规则：**

- `yield` 只暂停 animation 自身，调用方的 track / label 继续执行，互不阻塞
- `resume animation <handle>` 从 yield 点继续，与调用方执行流完全独立
- `hide` 对象时若有未 `resume` 的 animation，输出警告，animation 随对象一起丢弃：

```
AxnWarning: [animation] 'boss_enter' has unresumed yield point 'boss_landed'.
  Animation will be discarded on hide.
  Consider calling 'resume animation anim_boss' before hiding.
  → scene.apy, line 28
```

**`animation loop` 块**

`animation` 块内支持 `loop` 循环结构，用于重复演出直到条件满足：

```apy
animation battle_idle:
    loop until store["battle_end"]:   # 条件只允许 store 变量简单比较
        camera shake 3 0.3
        wait 1.5
    camera reset 0.5
```

无条件循环用 `loop`（不加 `until`），需要外部打断时配合 `yield` 使用：

```apy
animation ambient_loop:
    loop:
        play sound "sfx/wind.ogg"
        wait 3.0
        yield "loop_tick"           # 外部可选择性 resume 或 cancel
```

**`loop` 与存档的交互：**

| loop 类型 | 存档行为 |
|-----------|---------|
| `loop`（无条件） | 禁止存档，手动存档直接拒绝并提示 |
| `loop until` | 挂起存档，条件满足 loop 结束后自动执行 |

```
AxnWarning: Save rejected: an unconditional 'animation loop' is running.
  Unconditional loops cannot be safely checkpointed.
  → battle_idle animation, scene.apy, line 8
```

**`show` 不阻塞执行流**

`show autumn center (transform=shake_x)` 之后立即推进到下一行，transform 在后台运行，对话行不等 transform 完成。需要等待时显式使用 `handle=` + `wait for`：

```apy
show autumn center (transform=shake_x, handle=anim_shake)
wait for anim_shake
autumn: "你好。"    # shake_x 完成后才显示对话
```

不等待时 transform 跑着，对话同时出现，两者互不阻塞。`handle=` 在单行 `show` 上与并行写法上均有效。旧 `as` 写法保留但输出警告，可 ignore。

**前台动画与后台动画**

`transform` 按 `repeat` 类型自动区分前台/后台身份，不需要用户额外声明：

- **前台动画**：`repeat 1` / `repeat N`，参与 `wait for all` 的等待判断
- **后台动画**：`repeat forever` / `repeat forever pingpong`，不参与 `wait for all`，持续运行直到对象被 `hide` 或显式 `transform=none`

```apy
animation autumn_enter:
    show autumn center (transform=[complex_enter, breathe])
    # complex_enter: repeat 1       → 前台，参与等待
    # breathe:       repeat forever → 后台，不参与等待
    wait for all    # 等 complex_enter 完成即推进，breathe 继续跑
```

边界情况：`wait for all` 时若所有 transform 均为后台动画（无前台动画），立即满足，debug 模式输出警告：

```
AxnWarning: 'wait for all' has no finite transforms to wait for.
  All transforms are 'repeat forever'. Did you mean to use 'wait'?
  Line 3, autumn_enter animation block.
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
        show autumn left 0.5
        autumn: "……"                 # 正常等待点击
        sophia: "是啊。"
    track bgm:                      # 非交互轨道，只允许引擎指令
        play music "bgm/tense.ogg" 0.8 1.0
        camera shake 5 0.3
```

同一 `parallel` 块只允许一个 interactive track，多个时解析器报错：

```
AxnParseError: Multiple 'interactive' tracks in the same 'parallel' block.
  Only one track can handle user input at a time.
  → scene.apy, line 15 (second 'track (interactive)' in 'parallel' block starting at line 8)
Hint: Merge the dialogue lines into one interactive track.
```

`parallel` 块内的所有 `track` 并行执行，默认等所有 `track` 完成后推进（等价于 `wait for all`）。需要提前推进时：

```apy
parallel (wait=any):
    track a:
        ...
    track b:
        ...
```

`wait=any` 与 interactive track 共存时，解析器直接报错，不允许此组合：

```
AxnParseError: 'wait=any' is incompatible with 'track (interactive)'.
  An interactive track waits for user input, making 'wait=any' unpredictable.
  Use 'wait=none' with explicit 'wait for <track>' for fine-grained control.
  → scene.apy, line 8 (parallel (wait=any) containing interactive track)
```

理由：interactive track 正在等用户点击时，"最先完成的 track"触发推进，点击事件时机不可预测，会产生误触或输入状态污染。需要提前推进的场景，改用 `wait=none` + 手动 `wait for` 精细控制。

`track` 可命名后用 `wait for` 精细控制：

```apy
parallel (wait=none):           # 不自动等待，手动控制
    track dialogue (interactive):
        autumn: "第一句"
        autumn: "第二句"
    track bgm:
        play music "bgm/tense.ogg" 0.8 1.0
    track scene (handle=anim_scene):
        show autumn left 0.5
        camera shake 5 0.3

wait for anim_scene             # 等 scene track 完成后推进
wait for dialogue               # 等 interactive track 完成后推进
```

**`wait=none` + interactive track 的输入路由规则**：`wait for <track>` 等待期间，interactive track 的输入路由规则与 `parallel` 块内完全一致——interactive track 独占用户输入，点击推进的是 track 内部当前等待的对话行，不影响外部 `wait for` 的等待状态。`wait for` 只观察 track 的完成信号（`track.done`），不接管输入。用户点击推进对话 → interactive track 内部状态前进 → track 执行完毕 → `wait for dialogue` 自然满足，执行流继续。外部 `wait for` 是纯被动观察者，不与输入系统交互。

`parallel` 块在 GUI 脚本区中表现为时间轴视图，每个 `track` 对应一条轨道；interactive track 以特殊标记区分。

**立绘状态（sprite states）**

同一角色的不同表情、服装变体通过 `states` 声明管理。引擎默认按 `{角色名}_{state}.png` 命名约定自动扫描 `sprites` 目录，也可以显式声明覆盖：

```apy
define char autumn:
    sprites "assets/autumn/"       # 自动扫描，按命名约定构建状态表
    states:                        # 显式声明，覆盖自动扫描结果
        neutral    "assets/autumn/neutral.png"
        happy      "assets/autumn/happy.png"
        sad        "assets/autumn/sad.png"
        happy_alt  "assets/autumn/casual_happy.png"   # 服装变体
```

状态切换通过对话修饰符触发，是瞬时换帧，不产生过渡动画：

```apy
autumn: "早上好！" (happy)     # 切换到 happy state
autumn: "……"     (sad)        # 切换到 sad state
```

需要带过渡的状态切换时，在修饰符内指定 `transition`：

```apy
autumn: "……" (sad, transition=dissolve)
```

**过渡（transition）**

过渡作用于出入场和场景切换，使用已有的具名参数语法触发：

```apy
show autumn left 0.3 (enter=fade)
hide autumn 0.5 (exit=dissolve)
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
show autumn center (enter=SlideFromTop(0.3))   # 直接传实例
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

**自定义文本标签**

通过 `startup` 块注册自定义标签，与引擎整体风格一致，不引入新语法：

```apy
startup:
    python:
        from axn_plus.apy.text_renderer import TextRenderer, ShakeEffect

        # 注册自定义标签
        TextRenderer.register_tag("shake", lambda content, args: ShakeEffect(content))
        TextRenderer.register_tag("glow", lambda content, args: GlowEffect(content, color=args.get("color", "#ffffff")))
```

使用时与内置标签语法完全一致：

```apy
autumn: "这个字<shake>在颤抖</shake>。"
autumn: "发光的<glow color=#ff8800>文字</glow>。"
```

`TextRenderer.register_tag(name, handler)` 中 `handler` 接收 `(content: list, args: dict)` 并返回渲染效果对象，继承 `TextEffect` 抽象类。自定义标签与内置标签优先级相同，同名时自定义标签覆盖内置标签并输出警告。

**`then` 链式串行**

`then` 在 `transform` 块内声明串行后续动画，替代 `compose=sequence`：

```apy
transform enter_then_breathe:
    then complex_enter       # 先播 complex_enter（repeat 1）
    then breathe             # 完成后接 breathe（repeat forever）
```

`then` 只在 `repeat 1` / `repeat N` 的 transform 之后有意义。若 `then` 前的 transform 为 `repeat forever`，解析期报错：

```
AxnParseError: 'then' after 'repeat forever' transform will never execute.
  'then' requires a finite transform (repeat 1 or repeat N) before it.
  → characters.apy, line 8
```

**`on_complete` 回调**

transform 完成后自动触发的后续操作，只允许有限操作保证 GUI 可解析：

```apy
transform entrance_flash:
    keyframe 0.0: alpha 0.0
    keyframe 0.3: alpha 1.0
    repeat 1
    on_complete: transform += idle_float    # 完成后自动追加后台动画
```

`on_complete` 只允许以下操作：
- `transform =` / `transform +=` / `transform = none`
- `emit "event_name"`

不允许对话行、jump、Python 块，保证 GUI 完整可解析。

`on_complete` 用于 `repeat forever` 时解析期报错：

```
AxnParseError: 'on_complete' has no effect on 'repeat forever' transform.
  'on_complete' requires a finite transform (repeat 1 or repeat N).
  → characters.apy, line 15
```

**`mode`（绝对 / 相对模式）**

`transform` 默认以绝对值定义 keyframe 属性。`mode relative` 时所有属性值相对于对象当前状态叠加：

```apy
transform nudge_right:
    keyframe 0.0: x_offset +0
    keyframe 0.2: x_offset +20
    keyframe 0.4: x_offset +0
    repeat 1
    mode relative       # 相对当前位置，可安全叠加在任何位置的角色上
```

| `mode` 值 | 行为 |
|-----------|------|
| `absolute`（默认） | keyframe 属性值为绝对值 |
| `relative` | keyframe 属性值相对于对象当前状态叠加 |

**应用 transform：**

```apy
show autumn center (transform=shake_x)                              # 单个
show autumn center (transform=[breathe, shake_x])                   # 多个并行叠加
show autumn center (transform=shake_x(duration=0.2))               # 覆盖 duration
```

`compose=sequence` 保留但警告，推荐改用 `then` 链式写法：

```
AxnWarning: [transform] 'compose=sequence' is deprecated. Use 'then' chain instead.
  → scene.apy, line 12
```

**叠加模型（`compose`）：**

| `compose` 值 | 行为 |
|---|---|
| `parallel`（默认） | 所有 transform 同时运行，属性冲突取列表最后一个，引擎启动时对已知冲突输出警告 |
| `sequence` | 已废弃，保留但警告，推荐改用 `then` |

**触发与停止模型：**

新的 `show autumn (transform=X)` 替换当前所有 transform，不叠加。需要追加时使用 `transform+=`：

```apy
show autumn (transform=breathe)       # 启动 breathe
show autumn (transform=shake_x)       # 停止 breathe，启动 shake_x
show autumn (transform+=shake_x)      # 在现有基础上追加 shake_x
show autumn (transform=none)          # 显式停止所有 transform
```

| 事件 | 行为 |
|---|---|
| `hide autumn` | 立即停止所有附属 transform |
| `show autumn`（无 transform 参数） | 保留当前 transform，不中断（移动位置不打断循环动画） |
| `show autumn (transform=X)` | 替换全部 transform |
| `show autumn (transform+=X)` | 追加，保留现有 transform |
| `show autumn (transform=none)` | 显式停止所有 transform |
| `repeat 1` 动画自然结束 | 停止该 transform，不影响其他 |

**复杂动画退到 Python：**

`transform` 块覆盖不了的场景（帧动画、骨骼、物理），直接用 Python 类，`show` 接受任何实现了 `AnimatedSprite` 接口的对象：

```python
class AnimatedSprite:
    def update(self, dt: float): ...          # 每帧调用，返回当前帧 surface
    def on_show(self): ...                    # show 时调用
    def on_hide(self): ...                    # hide 时调用
    def on_pause(self): ...                   # pause transform 时调用
    def on_resume(self): ...                  # resume transform 时调用
    def on_snapshot(self) -> dict: ...        # 存档时调用（与 @restorable 统一接口）
    def on_restore(self, data: dict): ...     # 读档时调用
```

```python
class LivePortrait(AnimatedSprite):
    def __init__(self, char):
        self.frames = char.load_frames()
        self.timer = 0.0

    def update(self, dt):
        self.timer += dt
        return self.frames[int(self.timer * 12) % len(self.frames)]

    def on_show(self):
        self.timer = 0.0

    def on_snapshot(self) -> dict:
        return {"timer": self.timer}

    def on_restore(self, data: dict):
        self.timer = data["timer"]
```

```apy
show LivePortrait(autumn) center
```

`AnimatedSprite` 的生命周期钩子与 `@restorable` 接口统一——`on_snapshot` / `on_restore` 即 `__snapshot__` / `__restore__` 的别名，两种写法均可，引擎内部统一处理。

**`animation` 与 `transform` 的边界：**

- `animation`：引擎指令序列，描述演出流程（谁在哪里、镜头怎么动、什么时候等待）
- `transform`：keyframe 数据，描述单个对象的属性变化

两者组合使用：

```apy
animation autumn_enter:
    show autumn right 0.0 (transform=complex_enter)
    camera move 1.1 0.5
    wait for all
```

`transform` 块在 GUI 脚本区中表现为时间轴 keyframe 编辑器，属性列表完整可解析。Python `AnimatedSprite` 子类作为代码节点处理。

---

#### 颜色矩阵与对象级着色器（Matrixcolor / Sprite Shader）

`color_matrix` 属性作用于单个显示对象（立绘、图片）的整体颜色，CPU 端合成，不需要 GPU。与 `transform` 属性正交，可以同时存在。

**内置颜色矩阵：**

```apy
show autumn (color_matrix=grayscale)                  # 去色
show autumn (color_matrix=SaturationMatrix(0.3))      # 降低饱和度
show autumn (color_matrix=TintMatrix(#8888ff, 0.4))   # 蓝调叠色（color, strength）
show autumn (color_matrix=BrightnessMatrix(-0.2))     # 降低亮度（-1.0–1.0）
show autumn (color_matrix=ContrastMatrix(1.5))        # 提高对比度
show autumn (color_matrix=InvertMatrix)               # 颜色反相
show autumn (color_matrix=none)                       # 清除
```

**与 `transform` 的结合：**

```apy
show autumn (transform=breathe, color_matrix=TintMatrix(#ff8888, 0.3))
# transform 和 color_matrix 独立管理，互不覆盖
```

**`color_matrix` 不属于 `transform` 属性**：它是独立的后处理步骤，不参与 keyframe 插值，不受 `transform` 的 `repeat` / `mode` 逻辑影响。需要动画化颜色变化时（如角色受击闪红）使用 `transition_matrix` 参数：

```apy
show autumn (color_matrix=TintMatrix(#ff0000, 0.8), transition_matrix=0.1)
# 0.1 秒内从当前 color_matrix 过渡到新值
```

**Layer 级颜色矩阵**：对整个层应用颜色变换，作用于层上所有元素的合成结果：

```apy
layer color_matrix sprite grayscale           # 整个 sprite 层去色
layer color_matrix sprite TintMatrix(#8888ff, 0.5)
layer color_matrix sprite none                # 清除
```

**自定义颜色矩阵**：继承 `ColorMatrix` 基类，实现 4×5 矩阵（RGBA + 偏移列）：

```python
class NightMatrix(ColorMatrix):
    def __init__(self, intensity=1.0):
        r = 0.3 * intensity
        b = 0.7 * intensity
        # 4x5 matrix: [r_row, g_row, b_row, a_row]
        self.matrix = [
            [1-r, 0, 0, 0, -0.1*intensity],
            [0, 1-0.1*intensity, 0, 0, 0],
            [0, 0, 1+b, 0, 0],
            [0, 0, 0, 1, 0],
        ]
```

```apy
show autumn (color_matrix=NightMatrix(0.8))
layer color_matrix bg NightMatrix(0.6)
```

**Layer 级 Transform**：对整个层应用 transform 或过渡，作用于层的合成 surface：

```apy
layer transform sprite (transform=fade_out) 0.5   # 整层渐出，duration 作为第二位置参数
layer transform sprite (transform=shake_x)        # 整层抖动
layer transform sprite (transform=breathe)        # 整层呼吸动画
layer transform sprite none                       # 清除层 transform
```

`layer transform` 的 transform 名称语义与 `show` 的 `(transform=X)` 完全一致，共享同一套 transform 定义。Layer transform 不参与 `wait for all`（始终视为后台动画）；需要等待时用 `(handle=layer_anim)` + `wait for`。

```apy
layer transform sprite (transform=fade_out, handle=layer_fade) 1.0
wait for layer_fade
scene bg_new_room
layer transform sprite none
```

**Round-Trip**：`color_matrix=` 在 `show` 积木块中显示为独立颜色矩阵字段，内置矩阵提供下拉选择，参数可编辑；`layer color_matrix` 和 `layer transform` 对应层管理面板中的独立字段。

---

#### 数据与逻辑

**`with store` 块（批量状态变更）**

```apy
with store:
    flag_met_autumn = True
    day += 1
    relationship = new_rel      # ✅ 整体替换顶层变量
    # relationship["autumn"] += 5  ← 解析期报错，改用 python: 块先算好再赋值
```

提供真正的原子语义：执行前对所有涉及的顶层 key 做快照，中途抛出异常时自动回滚，保证状态变更要么全部完成、要么完全不发生。

**原子性边界**：`with store` 只保证顶层 `store` 变量的原子性。块内只允许顶层变量的赋值语句（`x = ...`、`x += ...`），不允许下标访问（`dict["key"]`）、属性访问（`obj.attr`）、方法调用（`list.append(...)`）或任何流程控制。违反时解析期报错：

```
AxnParseError: 'with store' only allows top-level store assignments.
  Use a 'python:' block for nested mutations, then assign the result.
  12 | relationship["autumn"] += 5
```

需要批量修改 `dict` 子项时，先在 `python:` 块里算好新值，再用 `with store` 整体赋值：

```apy
python:
    new_rel = dict(relationship)
    new_rel["autumn"] += 5

with store:
    relationship = new_rel
    day += 1
```

由于块内只允许顶层赋值，涉及的 key 在编译期静态确定，快照成本极低。

**快照的浅拷贝语义**：快照保存的是 `store` 顶层 key 指向的对象引用，而非深拷贝。回滚时，引擎将这些 key 指回快照保存的旧引用——如果旧对象在块执行期间被外部代码（如通过 Python 直接持有引用并原地修改）改动，回滚无法恢复旧对象的内容。在纯 `.apy` 工作流下此场景不会发生。**回滚保证的语义是：`store` 顶层 key 指向执行前的对象，不保证该对象内容的深度一致性。** 需要深度一致性时，在 `python:` 块里手动构造新对象（如 `new_rel = dict(relationship)`），再通过 `with store` 整体替换顶层 key。

**编译期静态分析警告**：parser 第三遍扫描时，检查紧邻 `with store` 块之前的 `python:` 块（或 `$` 行）是否对 store 变量执行了下标赋值（`dict["key"] = ...`）、属性赋值（`obj.attr = ...`）或原地修改方法调用（`list.append(...)`、`dict.update(...)` 等）。检测到时在编译期输出警告：

```
AxnWarning: [store] In-place mutation of 'relationship' detected before 'with store' block.
  In-place mutations are not covered by 'with store' rollback.
  If an exception occurs inside the 'with store' block, 'relationship' cannot be restored.
  → scene.apy, line 22
Hint: Use 'python: new_rel = dict(relationship)' then assign via 'with store: relationship = new_rel'.
```

此检测覆盖最常见的踩坑路径，不做运行时检测（成本不值得，且不可靠）。

**`const` 声明**

```apy
const MAX_RELATIONSHIP = 100
const ROUTES = ["a", "b", "c"]
```

引擎启动时静态求值，写入只读层（与内置符号同层），用户代码不可覆盖。右值必须是字面量或字面量组合，不允许引用 `store` 变量或调用函数。尝试在运行时覆盖 `const` 时抛出 `AxnConstError`：

```
AxnConstError: Cannot assign to constant 'MAX_RELATIONSHIP'.
  Constants are read-only and cannot be modified at runtime.
  → scene.apy, line 8
```

**`flag` 声明块**

集中声明游戏状态变量，使引擎可在启动时建立完整变量列表，并自动纳入存档管理：

```apy
flag:
    met_autumn = False
    agreed = False
    can_refuse = True
```

`flag` 块只允许出现在文件顶层，不允许嵌套在 `label` 或其他块内。右值必须是字面量（`bool`、`int`、`str`、`None`），不允许表达式。引擎启动时静态扫描所有 `flag` 块，生成全局变量注册表；引用了未声明变量时，引擎输出警告但不阻止运行（兼容直接用 `$` 赋值的工作流）。

`flag` 块声明的变量直接写入 `store`，访问方式与普通 `store` 变量完全一致，无命名空间前缀。

有类型注解的变量在 debug 模式下触发即时类型检查：引擎通过 `Store.__setitem__` 钩子，在赋值时立即验证类型是否匹配，不匹配时抛出 `AxnTypeError` 并指明声明位置，而不是等到存档时才发现。release 模式下 `Store` 退化为普通 `dict`，零开销：

```
AxnTypeError: Type mismatch for 'day': expected int, got str.
  Declared at: scene.apy, line 3 (flag: day: int = 1)
  Assignment at: scene.apy, line 42
```

**`set` 指令（GUI 友好写法）**

专门用于修改 `flag` 块声明的变量，使 GUI 能建立声明与赋值之间的归属关系：

```apy
set met_autumn = True
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
    autumn: "新的一天。"
```

支持具名参数扩展：

```apy
checkpoint "第二章" (thumbnail=current, bgm_preview="bgm/morning.ogg")
```

GUI 脚本区对应存档点积木块，章节结构一眼可见。

**`assert` 调试断言**

开发期用于验证游戏状态，发行版自动剥离：

```apy
assert flag_met_autumn, "进入此路由前必须已见过 autumn"
assert relationship["autumn"] >= 0, f"好感度不能为负：{relationship['autumn']}"
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
| 存档前 | `on before_save` | 存档写入前，适合清理临时变量 |
| 读档后 | `on after_load` | 读档完成后、执行流恢复前，适合重建显示状态 |

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
# ui/autumn_box.apy
import "ui/base.apy"

autumnBox extends BaseBox:
    background "ui/box_autumn.png"
    name_color #ff8800
    font "fonts/handwriting.ttf"
```

`extends` 只继承属性，不支持方法覆盖（UI 控件不是 Python 类）。属性冲突时子类覆盖父类，无歧义。跨文件引用 `template` 需要显式 `import`，使依赖关系可见。

**`include`（脚本片段静态包含）**

```apy
include "common/prologue.apy"
```

编译期展开，等价于将目标文件内容内联到当前位置。与 `jump`/`call` 的区别：`include` 是编译期合并，不产生运行时跳转，也不创建独立的执行上下文。

适用场景：跨章节复用的开场白、结局模板、通用菜单片段等。引擎启动时检测循环 `include`，发现时抛出错误并打印完整引用链：

```
AxnParseError: Circular include detected.
  common/prologue.apy → common/shared.apy → common/prologue.apy

  → common/shared.apy, line 3 (include "common/prologue.apy")
Hint: Extract the shared content into a third file that neither imports the other.
```

---

**`expoint`（内容注入点）**

专为内容注入设计的受控接口，定位是脚本流程层的可选 call。调用时运行时查找有没有对应定义，有就执行，没有就静默跳过。推荐用于 MOD 支持、DLC 内容注入、主题替换、多结局变体等场景，不推荐用于引擎或开发者生成的 Python 内容。

三个操作可以独立存在，互不依赖：

```apy
# 创建（可选，语法糖，用于声明注入点、排查已定义的注入点列表）
expoint = "after_prologue"
expoint = "after_prologue" (type=dialogue)  # 类型约束：只允许注入对话行，违反时警告
expoint = "after_prologue" (type=script)    # 默认，允许任意脚本内容

# 定义（填充内容，可在任意 .apy 文件中定义，包括外部注入文件）
expoint after_prologue:
    autumn: "这是DLC注入的额外对话。"
    $ dlc_flag = True

# 覆盖定义（replace 语义，不触发多次定义警告）
expoint after_prologue (replace):
    autumn: "覆盖后的内容。"

# 调用（主流程中的注入点，未定义时静默跳过）
expoint after_prologue
```

**多次定义行为**：同一 `expoint` 被多个文件定义时，按加载顺序**追加执行**，引擎输出警告，可 ignore。需要覆盖语义时用 `(replace)`，此时不警告。

**硬性限制**：定义块内禁止 `jump`（会破坏"注入后继续主流程"的语义），允许 `call`。违反时解析期报错：

```
AxnParseError: 'jump' is not allowed inside an 'expoint' definition block.
  'expoint' is designed to inject content that returns to the main flow.
  Use 'call' to invoke sub-labels, or restructure with a regular 'label'.
  → dlc/chapter3.apy, line 18 (inside 'expoint after_prologue:')
```

多次定义时的警告：

```
AxnWarning: [expoint] 'after_prologue' is defined in multiple files.
  Execution order: dlc/base.apy → dlc/chapter3.apy
  Use '(replace)' to override instead of appending.
  → dlc/chapter3.apy, line 5
```

**与现有机制的区别**：

| | `label` + `call` | `include` | `expoint` |
|---|---|---|---|
| 未定义时 | `AxnJumpError` 报错 | 编译期报错 | 静默跳过 |
| 展开时机 | 运行时 | 编译期 | 运行时 |
| 多次定义 | 冲突报错 | — | 追加执行，警告 |
| 目标用户 | 开发者 | 开发者 | MOD作者、DLC、内容扩展 |

---

#### Round-Trip Fidelity 补充

扩展语法对 Round-Trip 边界的影响：

| 新增语法 | GUI 处理方式 |
|----------|-------------|
| `{expr}` 插值 | 对话积木块内内联编辑，表达式作为文本字段 |
| `[A\|B]` 条件文本 | 对话积木块内内联编辑，A/B 作为独立字段 |
| `translate` 块 | 对话积木块的多语言标签页 |
| `animation` block | 脚本区独立节点，内容可完整解析为子积木序列；条件分支显示为多路分支节点；`yield` 显示为暂停点标记；`loop` / `loop until` 显示为循环节点 |
| `parallel / track` | 脚本区时间轴视图 |
| `with store` 块 | 脚本区代码节点（与 `python:` 块同等处理）；块内只允许顶层赋值，违反时编辑器解析期报错 |
| `const` | 脚本区只读常量节点 |
| `template / extends` | 窗口区组件继承树 |
| `transform` block | 脚本区 keyframe 时间轴编辑器，属性列表完整可解析；`repeat pingpong`、逐段 easing、`mode relative`、`on_complete`、`then` 链式串行均可解析 |
| `transform+=` | 脚本区追加节点，与 `transform=` 节点同类 |
| `transform then` | 时间轴编辑器内串行节点，显示为顺序连接的 transform 块 |
| `transform on_complete` | 时间轴编辑器内完成回调字段，显示触发操作 |
| `transform mode relative` | 时间轴编辑器内模式标记，属性值显示为相对偏移 |
| `triggers` 块 | 角色定义积木块内的响应式触发器面板；条件字段可编辑；触发操作显示对应 transform / animation 节点 |
| `AnimatedSprite` Python 类 | 脚本区代码节点（与 `python:` 块同等处理）；生命周期钩子在节点属性面板中列出 |
| `animation yield` | 脚本区暂停点积木块，显示 yield 名称字段；`resume animation` 节点与之对应显示关联句柄 |
| `animation loop` | 脚本区循环积木块；`loop until` 显示条件字段；`loop`（无条件）标注"存档禁用" |
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
| `define extends` 角色继承 | 角色定义积木块显示继承关系；子角色字段列表中继承字段以灰色标注来源；链式继承超过两层时节点显示黄色警告标记，字段来源标注完整展开路径 |
| `define image`（静态别名） | 资源管理面板独立区域，与角色列表分开展示；显示文件路径和资源预览 |
| `define image`（分层） | 图片对象积木块，`layers`/`states`/`expressions`/`triggers` 编辑器与角色定义完全一致；符号表中以"图片对象"类型区分于角色 |
| `jump/call/return if/unless` 条件短路 | 脚本区带条件标签的跳转箭头节点，视觉权重轻于完整 `if` 块 |
| `track (interactive)` | 时间轴视图中以特殊标记区分交互轨道与普通轨道；普通轨道内出现对话行时编辑器报错 |
| `cursor` 指令 | 脚本区光标切换积木块；`default` / `none` 显示关键字标签，路径显示文件名 |
| `engine.cursor` 变量 | 脚本区代码节点；编辑器在变量面板标注"光标控制变量，优先级最高" |
| 控件 `cursor=` 参数 | 控件节点的光标字段；具名关键字显示下拉列表，路径显示文件名 |
| 文本标签 `<b>` / `<w>` / `<nw>` 等 | 对话积木块内富文本编辑器，标签高亮显示；`<w>` 显示为中途等待点标记；`<nw>` 显示为"无需等待"标记 |
| 文本标签 `<s>` / `<u>` | 对话积木块内富文本编辑器，删除线和下划线高亮显示 |
| 文本标签 `<speed=N>` | 对话积木块内富文本编辑器，显示速度倍率字段；与行级 `speed=` 修饰符视觉区分（后者作用于整行） |
| Ruby 注音 `<rb>` / `<rt>` | 对话积木块内注音编辑器，原文与注音分列显示 |
| `no_history` 修饰符 | 对话积木块上的"不计入历史"开关 |
| 局部 label（`.` 前缀） | 脚本区以缩进或折叠形式展示，与全局 label 视觉区分；标注"仅在当前文件可见"；跨文件引用时显示完整路径 `文件名::.label名` |
| `label (rollback=...)` | label 节点的回滚策略字段，下拉选择 `dialogue` / `checkpoint` / `none` |
| `window show/hide/auto` | 脚本区对话框控制积木块；`hide (all)` 显示 mode 字段 |
| `pause`（游戏进程） | 脚本区暂停积木块；显示冻结类型（全局 / transform / 控件） |
| `pause transform` / `resume transform` | 脚本区 transform 暂停/恢复节点，句柄名字段可编辑 |
| `freeze` / `unfreeze` | 脚本区控件冻结积木块，控件名或层名字段可编辑 |
| `startup (before/after)` | 脚本区初始化阶段节点，独立于流程图展示 |
| `notify` / `notify system` | 脚本区通知积木块；`system` 标注为系统级通知 |
| `while` / `for` 循环 | 脚本区循环积木块；`rollback=` 字段必填，显示回滚策略标签；`break`/`continue` 显示为循环内控制流节点 |
| `pass` | 脚本区空操作占位节点，灰色显示 |
| `queue music/sound/ambient` | 音频积木块的队列节点，与 `play` 节点同类，标注"队列追加" |
| `transition` 独立指令 | 脚本区全屏过渡积木块，过渡名下拉列表，duration 字段可编辑 |
| `screenshot` | 脚本区截图积木块，路径字段可编辑，`include_ui` 开关 |
| `show (handle=)` / `show (alias=)` | `handle=` 显示句柄名字段；`alias=` 显示实例名字段；使用旧 `as` 写法时以黄色警告标注"建议改用参数化写法" |
| `show (alias=)` 多实例 | 脚本区多实例节点，alias 名字段可编辑；hide/expression 按 alias 名操作时显示实例来源 |
| `on before_save` / `on after_load` | 编辑器独立事件钩子面板，与 `on enter`/`on key` 并列展示 |
| 自定义文本标签 | `startup` 块内代码节点；标注"通过 TextRenderer.register_tag 注册自定义标签"；使用时在对话积木块内高亮显示自定义标签名 |
| `expoint = "name"` 声明 | 注入点面板独立区域，列出所有已声明的注入点，显示类型约束 |
| `expoint name:` 定义 | 代码节点（内容任意，GUI 不解析归属）；面板标注"已有定义"绿色标记 |
| `expoint name (replace):` | 代码节点，标注"覆盖定义"；与普通定义节点视觉区分 |
| `expoint name` 调用 | 脚本区独立积木块，标注"可选，未定义时跳过"；已有定义时显示绿色指示，未定义时显示灰色 |
| 开发者工具 | 编辑器集成调试面板；发布包中完全剥离（灰色标注"release 不包含"） |
| 动态 `show $sprite` / `jump $target` | 脚本区动态指令节点，变量名字段可编辑，与静态版本视觉区分 |
| `scene (with=..., all)` / `scene (with=..., layer=[...])` | 场景切换积木块内过渡层级字段；`all` 显示为"全画面"标记，`layer=` 显示层名列表 |
| `TransitionLibrary.register` | startup 代码节点；编辑器过渡选择器中显示已注册的自定义过渡名，与内置过渡同列 |
| `draggable` | 控件节点上的拖拽配置面板；`data` / `preview` / `layer` 字段可编辑；`free=true` 标注"自由定位模式" |
| `droptarget` | 控件节点上的放置目标配置面板；`accept_type` 显示类型字段；`accept` lambda 降级代码节点；`drag_over` 状态在样式编辑器中与其他状态并列 |
| `moveable` | 控件节点上的拖移配置面板；`handle` / `bounds` / `snap` 字段可编辑；`persist` 显示绑定变量名，`persist_read` / `persist_write` 显示为独立开关 |
| `<shader=...>` 标签 | 对话积木块内富文本编辑器中高亮显示；内置着色器显示名称+参数字段；自定义着色器标注来源 |
| `style shader:` | 样式编辑器内着色器字段，支持多个效果堆叠，拖拽排序 |
| `TextShaderLibrary.register` | startup 代码节点；编辑器着色器选择器中显示已注册的自定义着色器名 |
| `together` 块 | 脚本区"同时对话"积木块，块内各角色对话行并排显示；`nowait` 在块内标注为警告（无效）；`line=` 字段显示为按行标签编辑器，`|` 分隔的各段可独立编辑；`inp=` 字段显示绑定变量名 |
| `chorus` 块 | 脚本区合唱积木块；角色名列表可编辑；对话内容字段单行；语音字段标注"各角色独立播放"；`line=` / `inp=` 字段处理与 `together` 一致 |
| `startup_sequence` | `options_window.apy` 专属面板，序列步骤以时间轴形式展示 |
| `splash`（图片/视频/序列） | 启动序列面板内的 splash 节点；三种形式视觉区分；`skippable` 显示为开关；在 `input disable` 块内时标注"跳过已禁用" |
| `warning` 块 | 启动序列面板内的警告节点；`once` 显示为开关并标注"已确认后跳过"；内部走完整控件编辑器 |
| `loading` 块 | 启动序列面板内的加载节点；`tips` 子块显示轮播文字列表；`engine.load_progress` 标注为引擎内置变量 |
| `auto on/off/toggle` | 脚本区 Auto 模式积木块，与 `skip on/off/toggle` 并列，模式状态指示灯 |
| `skip on/off/toggle` | 脚本区 Skip 模式积木块；`skip_mode` 字段显示"seen/all"标签 |
| `pause (hard)` | 脚本区暂停积木块上的"强制停止"开关；与普通 `pause` 视觉区分（红色边框） |
| `nvl:` 块 | 脚本区 NVL 段落积木块，块内对话行累积显示；`nvl clear` / `nvl hide` 对应独立指令节点 |
| `mode "nvl"` | 角色定义积木块中的渲染模式字段，下拉选择 `"adv"` / `"nvl"` |
| `dialogue_mode "bubble"` | 角色定义积木块中的对话框模式字段；`bubble:` 子块展示为气泡配置面板 |
| `color_matrix=` | `show` / `scene` 积木块中独立颜色矩阵字段；内置矩阵下拉选择，参数滑块可调；`none` 显示为"清除" |
| `layer color_matrix` | 层管理面板中的层级颜色矩阵字段，与层 transform 并列 |
| `layer transform` | 层管理面板中的层级 transform 字段；`handle=` 时显示句柄名；标注"始终后台动画" |
| `preload` 指令 | 脚本区预加载积木块；路径列表可编辑；`preload label` 显示目标 label 名 |
| `filter` 指令 | 脚本区音频滤波积木块；通道名下拉；效果列表可编辑；过渡时间字段可选；`none` 显示"清除效果" |
| `define achievement` | 成就定义积木块，字段完整可解析；`hidden=true` 标注"未解锁前隐藏" |
| `unlock achievement` | 脚本区解锁节点，成就名下拉选择（来自符号表） |
| `unlock scene` | 脚本区场景解锁节点，label 名字段可编辑 |
| `unlock music` | 脚本区音乐解锁节点，路径字段可编辑 |
| `preferences:` 块 | `options_window.apy` 专属偏好面板，内置项完整可解析，自定义项按类型渲染 |
| `$ preferences.xxx` | 脚本区偏好写入节点，字段名下拉选择，值字段可编辑 |
| `engine.variant()` | 脚本区条件节点，变体名下拉选择；编译期已知 false 的分支灰色标注"将在打包时剥离" |
| `engine.keymap` 配置 | `options_window.apy` 专属按键映射面板，按行为分组，键名可编辑；冲突标红 |
| `side_image auto` | 角色定义积木块头像配置面板中的"自动命名约定"标记 |
| `side_image:` 子块（带 states） | 头像配置面板内展示状态→图片映射，与角色 states 联动标注 |
| `slot side_image` | 对话框模板编辑器中的头像占位符，尺寸可调；旁白行时标注"空插槽" |
| 翻译工具链（`axn extract-strings`） | 编辑器翻译面板；显示语言覆盖率进度条；缺失条目高亮标注"需翻译" |
| `character callbacks:` 子块 | 角色定义积木块中的回调面板，各回调函数名字段可编辑 |
| `voice_tag` | 角色定义积木块中的语音分组字段，下拉选择已定义标签 |

### 静态与动态修饰符

| 修饰符 | 适用对象 | 含义 |
|--------|----------|------|
| 无修饰符 | 全部 | 默认行为（`define` 默认静态，`label` 默认动态） |
| `sta` | `label` | 强制声明为静态，表达作者意图 |
| `dyn` | `define`、`label` | 显式声明为动态，运行时求值 |

```apy
define autumn:        # 静态（默认，无需标记）
dyn define autumn:    # 动态，运行时求值

label morning_scene:      # 动态（默认）
sta label morning_scene:  # 强制静态，非默认行为，显式标记
```

`sta` 仅在 `label` 上有意义——`label` 默认动态，加 `sta` 表示这是有意为之的静态声明，代码审查时一眼可见。`define` 本身默认静态，`sta define` 无额外语义，不支持此写法（解析期报错）：

```
AxnParseError: 'sta' modifier is not valid for 'define'.
  'define' is already static by default. 'sta' is only meaningful for 'label'.
  → characters.apy, line 3
```

静态 label 与动态 label 的具体区别：

| | 动态 label（默认） | 静态 label（`sta`） |
|---|---|---|
| `dyn define` 角色引用 | 允许 | 编译期报错 |
| `$` / `python:` 块 | 允许 | 编译期报错 |
| GUI 解析 | 可能含代码节点 | 保证完整解析为积木块，无代码节点 |
| 引擎优化 | 无额外优化 | 编译期预处理，跳转目标缓存 |

`sta label` 内含动态代码时的报错示例：

```
AxnParseError: 'sta' label 'morning_scene' contains dynamic code.
  'sta' guarantees no dynamic nodes; '$' blocks and 'python:' blocks are not allowed.
  → scene.apy, line 12 ($ flag = True inside 'sta label morning_scene')
Hint: Remove 'sta' modifier to allow dynamic code, or move the logic outside this label.
```

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

**`show` 位置约束**：`show` 的位置参数只接受预定义关键字（`left`、`center`、`right` 等），数值坐标通过具名参数 `pos=` 传入，与 duration（数字类型）不产生歧义。duration 必须跟在位置关键字之后——数字直接跟在角色名后面（没有前置位置关键字）时，引擎运行时输出警告并降级处理为 duration，保持当前位置，可 ignore；需要指定 duration 但不改变位置时，推荐使用 `(duration=0.3)` 具名参数。连续两次 `show` 同一角色且位置相同、第二次没有 duration 时，引擎输出警告（可能是漏写），可 ignore。

```apy
show autumn left 0.3                 # 合法：关键字位置 + duration
show autumn (pos=(100, 200))         # 合法：数值坐标，具名参数
show autumn 0.3 (pos=(100, 200))     # 合法：duration + 数值坐标（duration 不依赖位置关键字，走具名 duration 槽）
show autumn (duration=0.3)           # 合法：保持当前位置，只指定 duration
show autumn 0.3                      # AxnWarning：数字直接跟在角色名后，位置关键字缺失
                                     # 降级处理：duration=0.3，保持当前位置
```

```
AxnWarning: [parser] Number directly follows character name in 'show' without position keyword.
  Treating as duration=0.3, keeping current position.
  → scene.apy, line 8

Hint: Use 'show autumn (duration=0.3)' to make intent explicit.
```

**子命令**用于同一动词下行为模式本质不同的场景（如 `camera move` / `camera shake` / `camera reset`，`play music` / `play sound` / `play video`）。子命令集合由引擎硬编码，不可由用户扩展，解析器行为完全可预测。判断标准：参数描述"怎么做"时用具名参数；改变"做什么"时拆为子命令。子命令不可省略。

| 指令 | 位置参数顺序 | 保留具名参数 |
|------|------------|------------|
| `show` | 角色 → 位置 → duration | `enter` `layer` `transform` `compose` |
| `hide` | 角色 → duration | `exit` |
| `scene` | 路径 → duration | `with` `keep` |
| `clear` | 角色列表（可选） | `layer` |
| `expression` | 角色 → 表情名（可选） | 动态层具名参数（`face` `brow` `outfit` 等）`transition` |
| `play music/ambient` | 路径 → volume → fadein → fadeout | `once`（默认 loop，加 `once` 单次播放） |
| `play sound/voice` | 路径 → volume → fadein → fadeout | `loop`（默认单次，加 `loop` 循环） |
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
| `cursor` | 路径或关键字（`default` / `none`） | — |
| `window show/hide/auto` | — | `all` `mode` `freeze_audio` |
| `pause`（游戏进程） | — | `freeze_audio` `until` |
| `pause transform` | 动画句柄名 | — |
| `resume transform` | 动画句柄名 | — |
| `freeze` | 控件名（可选） | `layer` |
| `unfreeze` | 控件名（可选） | `layer` |
| `queue music/sound/ambient` | 路径 → volume → fadein → fadeout | `loop` |
| `transition` | 过渡名 → duration | — |
| `screenshot` | 路径（可选） | `include_ui` |
| `expoint` （调用） | 注入点名 | — |
| `while` | — | `rollback`（必须显式声明） |
| `for` | 变量名 `in` 可迭代对象 | `rollback`（必须显式声明） |
| `pass` | — | — |
| `break` / `continue` | — | — |
| `on before_save` | — | — |
| `on after_load` | — | — |
| `notify` | 消息字符串 | `icon` `duration` `priority` |
| `notify system` | 消息字符串 | `subtitle` `icon` |
| `together` | — | — |
| `chorus` | 角色名序列 | — |
| `splash` | 路径或 `axn_logo` 或 `video 路径` 或 `sequence 路径` | `duration` `skippable` `fadein` `fadeout` `background` `fps` `audio` `pattern` `layer` |
| `warning` | — | `skippable` `once` `background` |
| `loading` | — | — |
| `draggable` | — | `data` `preview` `layer` `free` `on_drag` `on_release` |
| `droptarget` | — | `accept_type` `accept` `on_drop` |
| `moveable` | — | `handle` `bounds` `snap` `persist` `persist_read` `persist_write` |
| `auto` | `on` / `off` / `toggle` | — |
| `skip` | `on` / `off` / `toggle` | — |
| `nvl` | （块，或子命令 `clear` / `hide`） | — |
| `filter` | 通道名（`music` / `sound` 等） | 效果列表具名参数（`reverb` `lowpass` `pitch` 等），duration（过渡时间） |
| `preload` | 路径列表（可多个）或 `label 名` | — |
| `unlock achievement` | 成就 id | — |
| `unlock scene` | 显示名 | `label` `thumb` `category` |
| `unlock music` | 路径 | `title` `composer` `thumb` `loop` |
| `layer color_matrix` | 层名 → 矩阵名或 `none` | — |
| `layer transform` | 层名 | `transform` `handle` `none` |
| `channel create` | 通道名 | — |
| `channel destroy` | 通道名 | — |
| `after` | duration（秒） | — （块语法，块内只允许引擎指令） |
| `defer` | — | — （块语法，在 label 退出时执行） |
| `once` | `per_session` / `per_playthrough` / `ever` | — （块语法） |
| `unwind` | — | `to`（目标 label 名，可选） |
| `exit` | — | `confirm` `save` |
| `repeat`（块） | 重复次数 | — （块语法，块内只允许引擎指令） |
| `signal` | 信号名字符串 | 任意具名参数（附加数据） |
| `rollback fence` | — | — （块语法） |
| `snapshot` | 快照名字符串 | `keys`（可选，指定顶层变量列表） |

### 关键设计决策

**`show` 的 `handle=` / `alias=` 与旧 `as` / `at` 写法**：推荐参数化写法——`handle=` 获取动画句柄，`alias=` 命名多实例。旧 `as` / `at` 写法保留但引擎运行时输出 `AxnWarning`，可 ignore，符合兼容性容错原则。`with` 关键字已被 `with char` / `with store` 占用，用于 `show` 时解析期直接报错（真实歧义，非兼容问题）：

```
AxnParseError: 'with' is ambiguous in 'show' context. Use 'handle=' for animation handles.
  → scene.apy, line 8
```

`at` 只接位置关键字（`left` / `center` / `right` 等），不接 transform；transform 只走 `(transform=X)` 具名参数。`alias=` 命名的实例与原角色完全独立，位置、表情、transform 状态互不影响，`hide`/`expression` 按 alias 名操作。

**`show` 多实例语义**：同一角色可通过 `alias=` 同时存在多个独立实例，适用于回忆、幻觉、镜像等演出场景。每个实例独立维护自己的可见性、位置和表情状态，引擎内部以 `(角色名, alias名)` 作为唯一标识。未指定 `alias=` 时默认实例名为角色名本身。

**`while`/`for` 循环的回滚策略**：循环块内包含对话行时，回滚语义不明（回到圈首？回到上一行？），因此 `rollback=` 是必须显式声明的参数，缺失时解析器报错。绝大多数循环场景建议使用 `rollback=none`。不包含对话行的纯逻辑循环推荐退回 `python:` 块。`break`/`continue` 仅在 `while`/`for` 块内有效，出现在块外时解析期报错。

**`transition` 独立指令**：全屏过渡，不切换任何内容。替代 Ren'Py 独立 `with` 语句在"纯屏幕过渡"场景的用途，语义比 `with dissolve` 更明确（明确是全屏操作，不依附于任何 show/hide/scene）。过渡名与 `show`/`hide`/`scene` 的 `enter`/`exit`/`with` 参数共享同一套过渡库。

**`queue` 音频指令**：将音频加入通道串行队列，当前播放结束后自动播放。引擎内部队列机制已存在（上限16，可配置），`queue` 是对外暴露的脚本接口。参数结构与 `play` 完全一致。

**`expoint` 设计原则**：定位是脚本流程层的可选 call，专为内容注入设计的受控接口。三个操作（声明/定义/调用）完全独立，可单独存在。核心语义：调用时运行时查找定义，有则执行，无则静默跳过。多次定义追加执行并警告；`(replace)` 覆盖语义不警告。定义块内禁止 `jump`（破坏"注入后继续主流程"语义），允许 `call`。面向 MOD 作者、DLC、内容扩展，不推荐用于引擎或开发者生成的 Python 内容。

**开发者工具**：开发模式（`axn run`）内置，Shift+\` 呼出。包含控制台（执行 Python 表达式）、变量浏览器（实时查看/编辑 store）、label 跳转、截图（F12）。发布包完全剥离。后续集成进 Axn-Editor，命令行版本始终保留。光标控制提供三种层级，优先级从高到低依次为：`engine.cursor` 变量（最高，适合状态联动）、`cursor` 指令（命令式，适合演出流程中的即时切换）、控件 `cursor=` 具名参数（样式绑定，鼠标进入控件时自动切换，离开时恢复上层光标）、`options_window.apy` 全局默认（最低）。`cursor default` 和 `$ engine.cursor = "default"` 恢复到全局默认光标；`cursor none` / `$ engine.cursor = "none"` 隐藏光标。控件 `cursor=` 参数接受路径字符串或 `options_window.apy` 中定义的具名光标关键字（如 `hover`、`drag`）。光标切换不产生过渡动画，立即生效。对话修饰符修改角色的**持久表情状态**，与角色是否可见无关；`show` 出场时使用角色当前表情状态，不重置为 `default_expression`。`show` 指令仅控制位置和可见性，不影响表情状态。无对话时切换表情使用独立的 `expression` 指令，支持可选的 `transition` 参数。

**兼容性写法容错原则**：混杂写法或语义混乱的情况默认允许，引擎运行到对应节点时抛出报错提示，可 ignore 以继续进程。推荐已有的标准写法，但不阻止开发者使用非标准写法，由此产生的问题由开发者负责。此原则适用于所有非歧义、非引擎严重影响的兼容性问题。

**Python 块边界**：`$` 后允许任何在单行内 Python 语法合法的内容，包括三元表达式、多重赋值等（如 `$ x = 1 if flag else 2`、`$ a, b = b, a`）。需要换行的复合逻辑使用 `python:` 块，换行符即边界，无需记额外规则。

**角色定义**：内联在 `.apy` 文件中，不使用外部 JSON。`define` 默认推断类型，`define char` 为显式声明，语义等价但意图更清晰。支持 `dyn define` 实现运行时动态定义。

**`define image`（非角色分层图片对象）**：`define char` 解决角色立绘的分层需求，`define image` 解决非角色可显示对象的分层需求（道具、背景元素、环境特效等）。支持与 `define char` 相同的 `states`/`layers`/`expressions`/`triggers` 字段，但不携带任何角色专属字段（`voice_prefix`、`dialogue_box` 等），对话行指向 `define image` 对象时解析期报错。`define image` 之间可以 `extends`，不能跨类型继承 `define char`，反之亦然，跨类型继承解析期报错。单行别名形式（`define image bg_room = "path"`）让资源名进入符号表，`show bg_room` 可用裸名字；与 `const` 的区别是后者不进符号表，需要 `$` 前缀引用。

**`show` 层级**：默认操作 sprite 层，需要时通过 `(layer=effect)` 等显式指定。层级扩展需求驱动，不过早设计。

**并行与串行执行**：同行逗号分隔的指令并行执行，引擎默认等所有并行动画完成后推进。需要精细控制等待时机时，用 `handle=` 给动画命名，再用 `wait for` 显式控制。`wait for` 中 `for` 是介词而非子命令，`wait for all` / `wait for any` / `wait for <name>` 三种形式语义链完整。旧 `as` 写法保留但输出 `AxnWarning`，可 ignore。

**`scene` 默认清空 sprite 层**：`scene` 切换背景时默认同时清空 sprite 层（高频用法零开销）。需要保留立绘时显式使用 `(keep)` 或 `(keep=角色名)` 具名参数；`(keep=[autumn, sophia])` 支持保留多个。`scene` 只清非持久层，持久层（如 `ui`）完全不受影响。

**`clear` 指令**：精确清除，无过渡，定位是"批量/精确移除"而非"退场"。支持指定角色（`clear autumn`）、多个角色（`clear autumn sophia`）、指定层（`clear (layer=effect)`）、指定层上的指定元素（`clear autumn (layer=effect)`）。无参数时清除 sprite 层所有元素。`clear` 不支持过渡动画，需要过渡退场时使用 `hide`。`clear` 可以显式清除持久层，但需要明确指定 `layer=`，不会误伤持久层。

**`hide` 与 `clear` 的语义区别**：`hide autumn 0.5 (exit=fadeout)` 隐藏单个角色，支持过渡动画，强调"退场"；`clear autumn` 立即移除，无过渡，强调"清除"。需要过渡时用 `hide`，需要批量或精确无过渡移除时用 `clear`。

**`call` 返回值**：只支持 `as` 写法——`call label() as result`。`_return` 作为引擎内部实现细节，不对外暴露。

**`label` 签名**：直接使用 Python 函数签名风格，支持默认值、`*args`、`**kwargs`。`return` 后跟任意 Python 表达式。

**跨文件引用**：`jump` / `call` 不需要 `import`，引擎启动时自动扫描所有 `.apy` 文件，label 冲突在启动时报错。跨文件引用 `define`（如 UI 控件）需要显式 `import`，使依赖关系可见。label 命名冲突由引擎扫描和 VSCode 插件共同处理，语言层不强制约束。

**局部 label（`.` 前缀）**：`.` 前缀声明的 label 只在声明所在文件内可见，不写入全局符号表，不参与全局冲突检测。适合模块内部跳转点（`.intro`、`.main_loop`、`.ending` 等），避免通用名污染全局命名空间。不同文件可以有同名局部 label，互不冲突。同文件内直接用 `.前缀名` 调用；跨文件访问使用显式路径 `文件名::.label名`，与现有跨文件引用语法一致。Parser 第一遍扫描时识别 `.` 前缀，局部 label 不进第一遍收集的全局名字集合。GUI 编辑器以缩进或折叠形式展示局部 label，视觉上归属所在文件，与全局 label 视觉区分。

**UI 控件定义**：在独立的 `.apy` 文件中定义，通过 `文件路径::控件名` 语法引用，如 `"ui/autumn_box.apy::autumnBox"`。

**推断失败行为**：所有默认推断逻辑（层级推断、类型推断等）在推断失败时抛出明确错误，不静默走错分支。

**transform 与 animation 边界**：`transform` 描述单个对象的属性动画（keyframe 数据），`animation` 描述演出流程（指令序列）。两者职责不重叠，通过 `(transform=...)` 参数组合使用。`transform` 块内只允许有限的引擎关键字，不允许 Python，保证 GUI 可完整解析；超出表达能力的场景通过继承 `AnimatedSprite` 退到 Python。

**transform 时间单位**：keyframe 时间值为绝对秒数，与 `show`/`hide` 的 duration 单位一致。`duration` 参数可覆盖总时长，引擎等比缩放所有 keyframe 时间点，相对节奏不变。省略 `duration` 时取最后一个 keyframe 的时间值。

**transform repeat**：`repeat 1` / `repeat forever` / `repeat forever pingpong` / `repeat N pingpong`。`pingpong` 下奇数次正向、偶数次反向，N 指完整循环次数，不是单程次数。

**transform easing**：支持全局声明和逐 keyframe 声明（`keyframe T (easing=X):`），逐段优先级高于全局，全局未声明时默认 `linear`。

**transform 叠加冲突**：`compose=parallel`（默认）时属性冲突取列表最后一个，不做混合，引擎启动时输出警告。`compose=sequence` 已废弃，保留但运行时警告，推荐改用 `then` 链式写法。

**transform 触发与停止**：`show autumn (transform=X)` 替换全部现有 transform；`show autumn` 无 transform 参数时保留当前 transform（移动位置不打断循环动画）；`transform+=X` 追加；`transform=none` 显式停止所有；`hide` 时自动停止所有附属 transform。

**transform `then` 链式串行**：在 `transform` 块内用 `then` 声明串行后续动画，替代已废弃的 `compose=sequence`。`then` 之前的 transform 必须为有限动画（`repeat 1` / `repeat N`），否则解析期报错。

**transform `on_complete` 回调**：有限动画（`repeat 1` / `repeat N`）完成后自动触发，只允许 `transform =` / `transform +=` / `transform = none` / `emit`，不允许其他操作。用于 `repeat forever` 时解析期报错。

**transform `mode`（绝对/相对模式）**：`mode absolute`（默认）keyframe 属性值为绝对值；`mode relative` 时属性值相对对象当前状态叠加，可安全叠加在任意位置的角色上。

**transform 响应 store 变量**：keyframe 表达式中允许引用 store 变量做简单运算，但不推荐，运行时输出警告可 ignore。响应式动画的推荐方案是 `triggers` 块或 `AnimatedSprite`。

**`triggers` 响应式触发**：在 `define char` 内声明，只在角色可见时监听，`hide` 后自动暂停，`show` 后恢复。条件只允许 store 变量简单比较，每帧检查，false → true 时触发一次不重复。回滚后清空所有 trigger 触发状态，等下次 false → true 才重新触发。

**`animation` 条件分支**：`animation` 块内允许基于参数变量的简单条件分支（`if` / `elif` / `else`），不允许引用 store 变量，保证 GUI 完整可解析为多路分支节点。需要响应 store 状态时退回 `label` + Python 块处理。

**`animation yield` 暂停点**：`yield` 只暂停 animation 自身，调用方 track / label 继续执行。`resume animation <handle>` 从 yield 点继续。`hide` 对象时若有未 resume 的 animation，输出警告，animation 随对象丢弃。

**`animation loop` 块**：`loop`（无条件）执行期间禁止存档，手动存档直接拒绝；`loop until` 执行期间挂起存档，条件满足 loop 结束后自动执行。`animation loop` 不要求声明 `rollback=`（与 `while`/`for` 不同）——原因是 `animation` 块内不允许对话行，回滚语义无歧义，存档行为由 loop 类型自动决定，无需开发者显式声明。`while`/`for` 要求显式声明 `rollback=` 是因为块内可能含对话行，回滚语义不明，两者的不对称有意为之。

无条件 `loop` 在确认安全时（如外部有 `yield` + `cancel` 明确退出点），可通过 `(checkpoint=allow)` 显式解锁存档：

```apy
animation ambient_loop:
    loop (checkpoint=allow):    # 开发者声明：此 loop 有明确退出点，允许存档
        play sound "sfx/wind.ogg"
        wait 3.0
        yield "loop_tick"
```

`(checkpoint=allow)` 不改变 loop 的执行语义，只解除存档拦截。开发者需自行保证存档时 loop 状态可安全重建。

**`@restorable` 与 `@saveable` 的职责边界**：`@saveable` 处理对象作为 `store` 数据时的序列化；`@restorable` 处理对象作为显示对象时的读档重建。两者职责不重叠，**不推荐同时施加在同一个类上**，但不强制禁止。

同时声明两个装饰器时引擎输出警告，可 ignore：

```
AxnWarning: [save] @restorable and @saveable are both applied to 'LivePortrait'.
  @saveable handles store data serialization.
  @restorable handles display object reconstruction.
  Consider splitting into a data class (@saveable) and a display class (@restorable).
  If you know what you're doing, ignore this warning.
  → characters/live_portrait.py, line 8
```

**组合使用时的行为**：两套接口完全独立执行，互不干扰——`__save__` / `__load__`（或 pickle）走 `@saveable` 路径，`__snapshot__` / `__restore__` 走 `@restorable` 路径。如果同一个字段同时被两套接口引用，引擎不做任何干预，开发者自行保证一致性。

推荐的拆分方式（作为设计参考，不强制）：

```python
@saveable
class CharacterData:              # 纯游戏数据，存入 store
    def __init__(self):
        self.relationship = 50

@restorable
class CharacterDisplay(AnimatedSprite):   # 纯显示状态，不进 store
    def __init__(self, data: CharacterData):
        self.data = data
        self.expression = "neutral"
        self.timer = 0.0

    def __snapshot__(self):
        return {"expression": self.expression, "timer": self.timer}

    def __restore__(self, snapshot):
        self.expression = snapshot["expression"]
        self.timer = snapshot["timer"]
```

**keyframe 折叠规则**：冒号后有内容 = 单行（单属性）；冒号后为空 = 展开块（多属性）。和 Python 自身的单行/块语法直觉一致，解析器无歧义。

**transition 扩展**：内置过渡由引擎标准库提供，自定义过渡继承 `Transition` 抽象类，通过 `apply(surface, progress)` 接口实现，直接传实例到具名参数，不引入新语法。

**`elif` / `unless`**：`elif` 补全条件链，语义与 Python 完全一致。`unless` 是 `if not` 的语法糖，仅用于卫语句场景（块内通常只有一条 `jump` 或 `return`），不支持 `unless ... elif ...` 链式写法，避免语义混乱。

**`match` 简单形式与复杂形式**：`match <store变量>:` + `值 -> label` 为简单形式，GUI 完整解析为多路分支节点。含表达式或 guard 的复杂形式整块降级为代码节点，不做部分解析，规则明确无歧义。

**`menu ->` 内联跳转**：选项只有一条 `jump` 时用 `->` 省略展开块；有额外逻辑时退回展开块写法。同一 `menu` 内两种形式可混用，不要求统一。

**`with char` 块**：块内裸字符串自动归属当前角色，修饰符继承块级声明。行级修饰符按**槽位覆盖**块级默认值——修饰符分为表情槽、具名参数槽、Flag 槽三类，行级只覆盖显式指定的槽位，未指定的槽位继承块级默认值。适合连续独白场景，不适合多角色交叉对话。

```apy
with autumn (happy, speed=1.0):
    "第一句。"                    # 表情=happy, speed=1.0
    "第二句。" (sad)              # 表情=sad,   speed=1.0  ← 只覆盖表情槽
    "第三句。" (speed=0.5)        # 表情=happy, speed=0.5  ← 只覆盖 speed 槽
    "第四句。" (sad, speed=0.5)   # 表情=sad,   speed=0.5  ← 两个槽都覆盖
```

**`narrate` 块**：连续旁白的语法糖，替代重复的 `@`。块内裸字符串全部作为旁白处理，支持修饰符。GUI 对应旁白段落积木块。

**`narrator` 保留关键字**：`@` 和 `narrator:` 是单行旁白的两种等价写法，`narrate:` 块是连续旁白的语法糖（等价于多行 `@`），`with narrator:` 与 `narrate:` 块语义一致。`narrator` 是引擎保留关键字，不允许用户通过 `define` 覆盖；尝试 `define char narrator` 时引擎在启动时报错：

```
AxnParseError: 'narrator' is a reserved keyword and cannot be used as a character name.
  Use a different name for your character.
  → characters.apy, line 3
```

单行旁白写法（`@`、`narrator:`）风格自选，同一项目内保持一致即可；连续旁白推荐 `narrate:` 块。

**`voice` 短路径**：对话修饰符中 `voice="001"` 自动展开为 `voice_prefix + "001" + 扩展名`。扩展名推断规则：若 `define` 中声明了 `voice_ext` 字段则直接使用；未声明时引擎按 `.ogg` → `.mp3` → `.wav` 优先级扫描，找到第一个存在的文件即用，全部找不到时抛出 `AxnVoiceError`：

```
AxnVoiceError: Voice file not found for short path '001'.
  Searched: vo/autumn/001.ogg, vo/autumn/001.mp3, vo/autumn/001.wav
  Character 'autumn' voice_prefix: 'vo/autumn/'
  → scene.apy, line 42
Hint: Use a full path like (voice="vo/autumn/001.ogg") or declare 'voice_ext' in define.
```

完整路径写法永远有效，短路径是语法糖。引擎构建发布包时（`axn build`），对所有短路径的推断结果固化为一张查找表（`dict[str, str]`，短路径 → 完整路径），打包进包体。运行时 `asset/loader.py` 优先查表，零 I/O；开发期查表未命中时回退到实时扫描兜底。

DLC / `mount_archive` 场景下，后挂载的归档引入新语音文件时，查找表支持增量合并（主表 + 归档补丁表分层查找），不覆盖主表。

```apy
define autumn:
    voice_prefix "vo/autumn/"
    voice_ext ".ogg"        # 可选；显式指定跳过扫描，性能更好；不填时按优先级自动推断
```

**`flag` 声明块**：只允许顶层声明，右值只允许字面量。支持可选类型注解（`name: type = value`），不写则不检查，保持向后兼容。引用未声明变量时输出警告不报错，保持与 `$` 工作流的兼容性。`flag` 声明的变量直接写入 `store`，无命名空间前缀，访问方式与普通变量完全一致。

**`flag` 初始化时机**：引擎启动时（早于任何 `startup` 块）统一扫描所有 `flag` 块，建立全局变量注册表，但**不立即写入 `store`**。写入时机：

- **新游戏**：`start` label 执行前，用声明的默认值初始化 `store`
- **读档**：用存档值恢复 `store`；存档中没有的 `flag`（新版本新增）用声明的默认值补齐；存档中多余的变量（旧版本删除的 `flag`）静默丢弃并输出警告：

```
AxnWarning: [save] Store variable 'old_flag' exists in save but is not declared in any 'flag' block.
  It will be discarded. If this is intentional, ignore this warning.
  If not, check if a 'flag' declaration was accidentally removed.
```

此机制保证跨版本存档兼容性：新增 `flag` 不破坏旧存档，删除 `flag` 不导致读档失败。

类型注解的作用：存档时做类型验证（不匹配抛 `AxnSaveError`）；GUI 变量面板按类型渲染控件（`bool` → 开关，`int`/`float` → 数字输入框，`str` → 文本输入框，`list`/`dict` → 折叠代码节点）；VSCode 插件可做悬停类型提示和赋值类型检查。

**类型注解验证边界**：`dict` / `list` 类型注解只验证顶层类型（`isinstance` 检查），不验证内部结构。例如 `relationship: dict = {}` 只保证 `relationship` 是一个 `dict`，不保证其 key/value 的类型。需要结构验证时，继承 `Saveable` 并在 `__load__` 里手动校验，不引入额外语法。

```apy
flag:
    met_autumn: bool = False   # 有类型注解，存档时验证
    day: int = 1
    player_name: str = ""
    relationship: dict = {}
    agreed = False             # 无类型注解，不检查，向后兼容
```

**`set` 指令**：推荐写法，不强制。`$` 永远可用。`set` 修改未在 `flag` 块声明的变量时，引擎输出警告（与引用未声明变量一致）。`set` 的存在价值是让 GUI 能追踪变量归属，`$` 的存在价值是不限制 Python 能力，两者定位不重叠。

**`checkpoint`**：引擎指令层语法，GUI 完整解析为存档点积木块。`thumbnail=current` 表示截取当前帧作为存档缩略图，为引擎保留关键字，不暴露为 Python 值。存档时 call 栈被丢弃，读档后以 checkpoint 下一行作为新的顶层执行起点。`checkpoint` 出现在被 `call` 的子 label 里时引擎给出编译期警告，建议移至顶层 label 入口处。

**`checkpoint` 存档时机**：存档在 `checkpoint` 指令执行时触发，记录的恢复起点是 `checkpoint` 的**下一行**。读档后直接从该行开始执行，`checkpoint` 指令本身不重新执行，不触发二次存档。

`checkpoint` 之前已发生的副作用（外部 API 调用、不可逆操作等）读档后不会重新执行。需要在读档后重现的副作用，应放在 `checkpoint` 之后——读档后这些代码会正常执行。

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

**`on key` 与 `parallel` interactive track 的关系**：`on key` 是全局事件钩子，不受 interactive track 的输入独占影响。interactive track"独占用户输入"指的是点击推进对话的输入路由，不涉及全局按键绑定。因此 `on key "escape"` 打开暂停菜单等全局操作在 interactive track 等待期间始终有效，除非 `input disable` 显式介入。

**位置参数连续填充规则**：位置参数必须从第一个开始连续提供，跳过任何一个则之后全部改具名参数。不支持占位符语法（`_`）。规则全局统一，适用于所有指令，用户学一条规则即可推导所有指令行为。

**`show` 位置与坐标扩展**：`show` 的位置参数只接受预定义关键字（`left`、`center`、`right` 等），数值坐标通过具名参数 `pos=(x, y)` 传入。两者类型不同（关键字 vs 数字），解析器无歧义。duration 必须跟在位置关键字之后；数字直接跟在角色名后面（无前置位置关键字）时，引擎运行时输出警告并降级处理为 duration，保持当前位置，可 ignore。推荐改用具名参数 `(duration=0.3)` 使意图明确。连续两次 `show` 同一角色且位置相同、第二次没有 duration 时，引擎输出警告提示可能漏写，可 ignore。

**`pause` / `resume`**：独立动词，不是 `play` / `stop` 的子命令。语义区别：`stop` 停止并丢弃进度，`pause` 保留进度暂停，`resume` 从保留位置恢复。适用于 `music`、`sound`、`video`，子命令语法与 `play` / `stop` 对称。`pause` 接受 `fadeout` 位置参数（画面/音量渐暗），`resume` 接受 `fadein` 位置参数。

**`play music` / `play ambient` 默认 loop**：视觉小说 BGM 和环境音绝大多数需要循环，默认 loop，单次播放时显式加 `(once)`。`play sound` / `play voice` 默认单次，循环时显式加 `(loop)`。`queue music` / `queue ambient` 同样默认 loop，与 `play` 保持一致。

**`play video` 默认阻塞**：与 `play music`（默认非阻塞）相反，`play video` 默认阻塞执行流，播完后才推进。非阻塞时显式加 `(async)`。理由：视频大多数时候是过场动画，播完才推进是高频用法；背景循环视频是少数场景，需要显式声明意图。`(blocking)` 关键字保留但冗余，不推荐写。

**`say` 动词**：专用于说话者在运行时动态决定的场景。静态说话者必须使用 `角色:` 或 `@`，`say` 传入静态角色名（编译期可确定的标识符）时报错，不允许作为 `角色:` 的等价写法。此限制保证代码风格统一，消除"两种写法都能用"带来的歧义。修饰符与对话行修饰符完全一致。

**`choice` 动词**：`menu` 是静态声明语义，选项在编译期确定，GUI 完整解析为菜单节点。`choice` 专门处理动态场景，接受运行时生成的选项列表（`list[dict]`），整体作为代码节点处理，GUI 不尝试解析列表内容。两者定位不重叠，`choice` 不是 `menu` 的超集。

**`input disable` 两种形式**：对称写法（`input disable` / `input enable`）和块语法（`input disable: ...`）均支持，两者语义等价。块语法是推荐写法——引擎保证块结束后自动恢复，即使块内发生 `jump` 或异常也能正确还原输入状态，无需手动配对 `enable`。对称写法保留，适合跨 label 的长期禁用场景。`input disable` 支持细粒度 flag 列表（`skip`、`rollback`、`all`），无参数时等价于 `(all)`。

**`modal` 焦点模型**：`modal show` 激活时，引擎自动屏蔽底层场景输入（等价于 `input disable (all)`）、将焦点锁定到模态框内控件，模态框关闭时自动恢复，无需手动管理。`modal show ... as result` 阻塞执行流，等用户在模态框内做出选择后将结果写入 `result` 变量再推进。`modal` 与 `input disable` 的区别：`input disable` 用于演出期间屏蔽输入（无返回值、不转移焦点），`modal` 用于 UI 交互等待用户选择（有返回值、转移焦点）。

**`camera follow`**：`camera` 的新子命令，与 `move` / `shake` / `reset` 平级。`camera follow none` 取消跟随，恢复静止镜头。`lag` 参数控制跟随延迟（秒），值越大镜头越"懒"，0 为即时跟随。`camera follow` 与 `camera move` 可共存——`follow` 设定跟随目标，`move` 在此基础上叠加偏移。

**`layer` 管理**：`layer` 作为独立动词，子命令 `create` / `destroy` / `order`。`create` 的 `above` / `below` 具名参数指定新层相对于已有层的位置；`order` 接受层名序列（从下到上），一次性重排所有层顺序。层的创建和销毁在引擎启动时静态检查，运行时销毁非空层时输出警告。

**持久层**：`layer create` 支持 `persistent` flag，声明为持久层的层不受 `scene` 的默认清除影响。内置层中 `ui` 默认持久，`bg` / `sprite` / `effect` 默认非持久。`clear (layer=ui_hud)` 可以显式清除持久层，但 `clear`（无 `layer` 参数）只清非持久层，不误伤持久层。

**分层立绘**：角色立绘支持两种模型，二选一，同一 `define` 块内不可混用，混用时引擎启动报错：

```
AxnParseError: Cannot use both 'states' and 'layers' in the same 'define char'.
  → characters.apy, line 8 (define char autumn)
```

`states` 模型：整图切换，每个状态对应一张完整立绘图片。`layers` 模型：多图层叠加，静态层（单文件）不参与状态切换，动态层（有子状态列表）通过修饰符切换；`expressions` 映射将一个修饰符名映射到多个动态层的组合状态，`expressions` 映射必须覆盖所有动态层，漏写时引擎启动报错：

```
AxnParseError: 'expressions' mapping for 'autumn' is incomplete.
  Dynamic layer 'brow' has no entry in 'expressions'.
  All dynamic layers must be covered by at least one expression mapping.
  → characters.apy, line 22
```

图层叠加顺序：要么全部走声明顺序，要么全部写 `z_order`，不允许混用，混用时引擎启动报错：

```
AxnParseError: Mixed z_order in 'layers' for 'autumn'.
  Either all layers must have explicit 'z_order', or none should.
  Layer 'body' has z_order=1, layer 'hair' has no z_order.
  → characters.apy, line 8
```

**`expression` 指令**：无对话时切换表情的专用指令，`show` 不承担此职责。`states` 模型下 `expression autumn happy` 整图切换；`layers` 模型下走 `expressions` 映射。`layers` 模型支持直接指定各层（`expression autumn (face=happy, brow=angry)`）绕过映射，也支持换装（`expression autumn (outfit=casual)`）。可选 `transition` 具名参数控制过渡效果。两套模型下用户侧语法完全一致，差异由引擎内部按角色声明类型分派。

**`menu as` 返回值**：`menu as result` 选完后继续当前执行流，选项 `->` 右侧为返回值表达式而非 label 名。`menu as` 内不允许 `jump`，混用时解析器报错。需要前置逻辑时用展开块 + 显式 `->` 返回。GUI 对应独立的"菜单返回值"节点，与跳转型菜单节点分开，不混用。

**`define extends` 角色继承**：子角色继承父角色所有字段，显式声明的字段覆盖父定义。`layers` 模型下同名动态层内按 key 合并，未声明状态继承父定义。支持链式继承（A extends B extends C），但引擎启动时输出警告，可 ignore；字段展开顺序为从根到叶，子类覆盖父类。建议保持单层继承以维持可读性——链式超过两层后，字段来源在代码审查时难以追踪，GUI 编辑器对超过两层的链式继承在角色定义节点上显示黄色警告标记，并在字段列表中标注完整展开路径。`define char` 与 `define image` 之间不允许跨类型继承，解析期报错。继承只发生在编译期展开，运行时两个角色是完全独立的显示对象。

**条件跳转短路写法**：`jump`/`call`/`return` 行末可接 `if`/`unless` 条件，条件表达式为完整 Python 表达式。不支持 `call ... as result if ...`（条件不满足时返回值语义不明），退回 `if` 块处理。GUI 对应带条件标签的跳转箭头节点，视觉权重轻于完整 `if` 块，与 `unless` 卫语句设计意图一致。

**`parallel` 交互轨道模型**：`track (interactive)` 显式标记允许对话行的交互轨道，独占用户输入，每个 `parallel` 块只允许一个。普通轨道不允许对话行，遇到时解析器报错：

```
AxnParseError: Dialogue line is not allowed in a non-interactive track.
  Declare this track as 'track name (interactive)' to allow dialogue.
  Note: only one interactive track is allowed per 'parallel' block.
  → scene.apy, line 12 (autumn: "..." inside non-interactive track)
```

`wait=any` 与 interactive track 共存时解析器直接报错——interactive track 等待用户点击期间，`wait=any` 触发推进的时机不可预测，会产生输入状态污染，此组合没有合理的使用场景。需要提前推进时改用 `wait=none` + 手动 `wait for`。`wait=none` 下使用 `wait for <interactive_track>` 时，输入路由规则不变：interactive track 仍然独占用户输入，`wait for` 是纯被动观察者，只轮询 `track.done`，不接管输入；用户点击推进对话 → track 内部前进 → track 完成 → `wait for` 自然满足。此设计消除了对话行与并行执行之间的交互模型歧义。

**`Store` 作为 exec globals 的代理层**：`exec()` 的 globals 使用内部 `_exec_globals` dict，`Store` 作为其代理层对外暴露游戏变量的读写接口。`exec()` 会自动向 globals 写入 `__builtins__` 等 dunder key，`Store` 序列化时过滤所有 dunder key，保证存档不被污染。此设计同时避免了用户通过 `__builtins__["eval"]` 等路径绕过只读层的问题。

**`with store` 真正的原子语义**：只允许顶层 `store` 变量的赋值语句（`x = ...`、`x += ...`），不允许下标访问、属性访问、方法调用或任何流程控制。违反时解析期报错，不静默通过。原子性边界明确：快照和回滚只针对顶层 key，`dict`/`list` 子项的内部修改不在保证范围内——需要修改子项时，先在 `python:` 块里构造好新值，再用 `with store` 整体赋值。由于块内只允许顶层赋值，涉及的 key 在编译期静态确定，快照成本极低，"原子性"是有实现保证的语义，不是注释。快照保存的是对象引用而非深拷贝——回滚保证 `store` 顶层 key 指向执行前的对象，不保证该对象内容的深度一致性；在纯 `.apy` 工作流下此边界不可见，通过 Python 直接持有 `store` 变量引用并原地修改时需开发者自行注意。外部引用检测不做运行时实现（`sys.getrefcount()` 不可靠，`gc.get_referrers()` 有 GC 暂停开销），此边界通过文档说明清楚即可。

**`with store` 前置原地修改的静态分析警告**：parser 第三遍扫描检测紧邻 `with store` 块之前的 `python:` 块或 `$` 行是否对 store 变量执行了下标赋值、属性赋值或原地修改方法调用，检测到时编译期输出警告。不做运行时检测（成本不值得且不可靠），此警告覆盖最常见的踩坑路径。

**`flag` debug 模式类型检查**：有类型注解的变量在 debug 模式下通过 `Store.__setitem__` 钩子即时验证类型，错误信息包含声明位置。release 模式下 `Store` 退化为普通 `dict`，零开销。避免类型错误拖到存档时才暴露。

**`menu` 的 `default` 参数**：使用选项 `id` 而非选项文本，避免多语言环境下文本匹配失败。选项通过可选的 `(id="...")` 声明标识符；未声明 `id` 时以选项文本作为 fallback，引擎启动时输出警告提示多语言风险。

**`on key` 组合键**：字符串格式为 `"修饰键+key"`，修饰键小写，顺序固定为 `ctrl → shift → alt → key`，用 `+` 连接（如 `"ctrl+s"`、`"shift+f5"`）。解析器在启动时验证格式合法性。

**`camera reset` 清除 follow 状态**：`camera reset` 同时清除 follow 状态，恢复静止镜头。reset 后需要继续 follow 须显式重新声明，避免隐式状态残留。

**`animation` 参数化**：`animation` 块支持函数签名风格参数，和 `label` 保持一致。参数类型限制为角色名、位置关键字、数值、字符串字面量，不允许 Python 表达式，也不允许 `$` 前缀的动态变量——参数必须在编译期完全确定，保证 GUI 能完整解析调用点。调用点传入 `$` 前缀参数时解析期报错。需要动态参数时退回 `label` + Python 块处理。

**`show` 永不阻塞执行流**：`show` 指令（含 `transform` 参数）之后立即推进到下一行，transform 在后台运行。需要等待时显式使用 `handle=` + `wait for`。`handle=` 在单行 `show` 和并行写法上均有效。旧 `as` 写法保留但输出 `AxnWarning`，可 ignore。

**前台动画与后台动画**：`transform` 按 `repeat` 类型自动区分身份，不需要用户额外声明。`repeat 1` / `repeat N` 为前台动画，参与 `wait for all`；`repeat forever` / `repeat forever pingpong` 为后台动画，不参与 `wait for all`，持续运行直到对象被 `hide` 或 `transform=none`。`wait for all` 时若无前台动画立即满足，debug 模式输出警告。

**`transform` 归属对象不归属 track**：`scheduler.py` 维护全局 transform 注册表，key 为显示对象 id。`parallel` track 调度与 transform 调度完全独立，track 结束不停止其内 `show` 触发的 transform，`hide` 对象时才停止所有附属 transform。此规则在 `parallel` 场景下与单轨道场景完全一致。

**存档分层快照策略**：显示状态按"可序列化层"（直接快照）和"不可序列化层"（`AnimatedSprite`）分层处理，不做全量快照也不做脚本重放。`AnimatedSprite` 通过 `@restorable` 装饰器提供 `__snapshot__` / `__restore__` 扩展点，不加 `@restorable` 时以初始状态重建并输出警告。`@restorable` 与 `@saveable` 职责不重叠：前者处理显示对象重建，后者处理 `store` 游戏数据序列化。

**`transform` 读档恢复**：前台动画（`repeat 1` / `repeat N`）读档不恢复；后台动画（`repeat forever`）读档恢复但从头开始播，存档只保存 transform 名称列表，不保存进度；`AnimatedSprite` 走 `@restorable` 机制。

**`parallel` 块与存档的边界**：`checkpoint` 禁止出现在 `parallel` 块内（解析期报错）；`checkpoint` 须在所有 track 完全结束后执行，否则运行时跳过并警告；`parallel` 执行期间自动存档挂起，手动存档挂起并显示 UI 提示，两者均在块完全结束后触发。

**`on enter` 读档触发行为**：默认（`restore=auto`）读档后不重新触发，依赖状态快照恢复；`restore=always` 读档后重新触发，开发者自行保证幂等性；`restore=never` 读档后永不触发。`restore=always` 时 debug 模式输出幂等性提醒。

**文本标签系统**：对话行内使用 `<tag>` 语法，与 `{expr}` 插值双轨并行，`TextRenderer` 统一解析。`<w>` 在句中产生中途等待点，`DIALOGUE` 指令内部维护状态机处理多等待点；`<nw>` 说完不等点击直接推进。`<w>` 产生的中途等待点不作为回滚检查点，整行对话作为一个回滚单元。

**回滚系统分级策略**：`rollback=dialogue`（默认）只回滚对话显示状态，Python 状态变更不回滚；`rollback=checkpoint` 回到最近存档点的完整状态；`rollback=none` 完全禁止回滚。策略在 label 声明处指定，不做全量状态快照链，务实取舍开销与功能的平衡。

**对话历史 buffer**：引擎层维护，UI 层由标准库模板提供。`no_history` 修饰符标记不计入历史的对话行；`<nw>` 行和 `<fast>` 行正常计入历史。buffer 是否持久化到 `persistent` 由 `options_window.apy` 配置。

**动态指令 `$` 前缀**：`show $sprite` / `hide $sprite` / `call $target` / `jump $target` 统一用 `$` 前缀标记运行时求值，与 `say speaker` 的动态变量语义一致，一条规则覆盖全部动态指令。`$` 前缀只接受单一 store 变量名，不接受任意表达式，保证 GUI 可解析调用点。

**`startup` 块**：替代 Ren'Py 的 `init N:` 数字优先级，使用语义化阶段声明（`before` / 默认 / `after`）。三个阶段顺序固定，同阶段内按文件扫描顺序执行。只允许出现在文件顶层。

**`pause`（游戏进程）与 `wait` 的区别**：`wait` 是执行流等待点，游戏世界继续运行（动画、音频继续）；`pause` 是字面意义的冻结，时间停止，所有 transform、timer、动画全部冻结。`pause transform` 单独暂停指定动画句柄，保留进度，`resume transform` 从冻结帧继续。`freeze` / `unfreeze` 冻结控件的输入响应，自动应用 `disabled` 样式。全局 `pause` 和 `pause transform` 状态独立，全局 `resume` 不意外恢复单独暂停的 transform。

**`window` 控制**：`window show` / `window hide` / `window auto` 控制对话框容器可见性。`window auto` 是推荐默认工作流，引擎自动管理，彻底规避对话进行时窗口被隐藏的问题。`window hide` 只隐藏容器，文字按 `hide_behaviour` 策略降级；`window hide (all)` 隐藏整个对话系统，`mode=pause` 时遇对话行挂起等 `window show` 恢复，`mode=skip` 时静默跳过对话行且不计入历史。`mode=skip` 期间遇到 `menu` 时强制切换为 `pause` 模式并输出 `AxnWarning`，不允许静默跳过玩家选择权。对话框与输入系统正交，`window hide` 不影响 `input disable` 状态，反之亦然。

**`notify` 升格为核心指令**：从标准库扩展升格为核心脚本指令。`notify` 触发游戏内通知，`notify system` 触发平台级系统通知（游戏最小化时可见）。内置库模板提供默认 UI，开发者可替换。

**兼容性写法容错原则**：混杂写法或语义混乱的情况默认允许，引擎运行到对应节点时抛出报错提示，可 ignore 以继续进程。推荐已有的标准写法，但不阻止开发者使用非标准写法，由此产生的问题由开发者负责。此原则适用于所有非歧义、非引擎严重影响的兼容性问题。

**`show` 裸数字降级行为**：数字直接跟在角色名后面（无前置位置关键字）时，引擎运行时输出警告，降级处理为 duration，保持当前位置，可 ignore。推荐改用 `(duration=0.3)` 具名参数使意图明确。此行为属于兼容性容错，不是设计意图，不推荐依赖。

**`scene` 过渡层级作用域**：`scene` 的 `with` 参数默认只作用于背景层，立绘层独立不参与。`(all)` flag 让整个画面（含立绘）参与过渡，`(layer=[...])` 精确指定参与层。`transition` 独立指令始终作用于整个画面。

**并行过渡叠加规则**：对象级过渡（`show`/`hide` 的 `enter`/`exit`）可以多个同时运行互不干扰；全屏级过渡（`scene` 的 `with`，`transition` 指令）同一时间只允许一个，后触发的覆盖前一个并输出警告。

**自定义过渡注册**：`TransitionLibrary.register(name, cls)` 让自定义过渡进入过渡库，注册后与内置过渡等价，支持裸名字引用和参数化调用。传实例写法保留，适合一次性使用。

**拖放系统**：`draggable` 声明可拖拽控件，携带 `data`、`preview`、`layer` 参数；`droptarget` 声明放置目标，`accept_type` 简单过滤（GUI 可解析），`accept` lambda 复杂过滤（降级代码节点）；`drag_over` 作为新增状态关键字用于悬停视觉反馈。`free=true` 时走自由定位模式，不需要 `droptarget`。预览控件默认渲染在最顶层，可通过 `layer=` 自定义。`moveable` 专为窗口拖移设计，与 `draggable` 独立可共存；`persist` 默认隐式双向绑定，`persist_read` / `persist_write` 显式控制单向行为。

**`draggable` + `moveable` 共存时的消歧义规则**：

- `moveable (handle=区域)` 时：`handle=` 区域内的拖动触发移动，其余区域触发 `draggable`
- `moveable`（无 handle）时：整个控件区域优先触发移动，`draggable` 完全不响应——引擎启动时输出警告：

```
AxnWarning: [ui] 'draggable' and 'moveable' both declared without 'handle'.
  'moveable' takes priority over the entire widget area.
  'draggable' will never trigger.
  Consider adding 'moveable (handle=...)' to define a dedicated drag handle.
  → ui/inventory.apy, line 24
```

手势起始时引擎检查起点是否在 `handle=` 区域内，在则走 `moveable`，否则走 `draggable`。规则唯一，可预测。

**文本着色器**：`<shader=效果名(参数)>` 标签作为统一入口，支持内联和 `style` 系统集成；`TextShader` 基类提供 `apply_char`（字符级）和 `apply_block`（块级）两个入口；`is_static=True` 时引擎只计算一次结果缓存；`char_index` / `total_chars` 均相对于已显示部分，打字机效果中自然展开；`TextShaderLibrary.register` 注册自定义着色器。

**`together` 块**：多角色同时说话，共享一个等待点，点击打断所有语音；整体作为一个回滚单元；对话修饰符正常修改各自角色持久表情状态；`nowait` 在块内无效并警告；对话框布局策略在 `options_window.apy` 中配置。支持 `line=` 和 `inp=` 块级参数（见下）。

**`chorus` 块**：多角色合唱，显示在同一对话框，所有角色语音同时播放各走各自 `voice_prefix`；整体作为一个回滚单元；名字显示格式可配置。支持 `line=` 和 `inp=` 块级参数（见下）。

**`together` / `chorus` 的 `line=` 和 `inp=` 块级参数**：两个语法糖，适用于 `together` 和 `chorus` 块。`line=` 用 `|` 分隔，按顺序对应块内各行施加文本标签效果，单值无 `|` 时应用整块，多出的段静默忽略；同一段内逗号叠加多个标签。`inp=` 接受 `store` 变量（必须是 `dict`），key 为修饰符参数名，统一应用到块内所有行；行内显式声明的同名参数优先级更高。两者均为纯展开语法糖，不引入新渲染逻辑，可同时使用互不干扰。

**旁白在非交互轨道**：`@` / `narrator:` 不算对话行，允许出现在 `parallel` 的非交互轨道，不产生输入路由冲突。

**`startup_sequence`**：在 `options_window.apy` 声明，早于 `start` label 执行。`splash` 支持静态图片、视频、图像序列三种形式；`warning` 块平台合规用，`once=true`（默认）确认后写入 `persistent` 不再显示；`loading` 块显示期间后台加载资源；`input disable` 块在 `startup_sequence` 内完全禁用输入含跳过；开发模式 `skip_startup=true` 跳过整个序列；`show_axn_logo=false` 禁用引擎内置 logo。

**Auto/Skip 模式**：两者互斥，同时触发时 Skip 优先。Auto 遇 `menu`、`pause (hard)`、`modal show` 强制暂停；Skip 遇 `menu`、`pause (hard)`、`modal show` 永远停止，`"seen"` 策略下遇未读对话停止。Auto 期间回滚正常工作；Skip 期间回滚被禁用，回滚键临时打断 Skip 并回退一步。`pause (hard)` 是任何自动推进机制的硬停止点，不受兼容性容错原则影响——此行为属于"对引擎有严重影响"的情况。

**已读追踪**：引擎为每条对话行维护唯一标识（`文件名 + label名 + 行偏移` 哈希），写入 `persistent`，跨存档槽共享，只增不减。`$` 动态 `say speaker` 行以运行时角色名 + 文本哈希作为标识（每次文本变化都是新标识，设计取舍：动态文本无法静态追踪）。

**NVL 模式**：`nvl:` 块内文字累积，整块作为回滚单元。`nvl:` 块内禁止 `checkpoint`（编译期警告）。`nvl clear` 清屏不退出 NVL，`nvl hide` 退出 NVL。`nvl:` 块内所有角色走 NVL 渲染路径，无论其 `define char` 中的 `mode` 声明——块级语义优先于角色声明。

**Speech Bubbles**：气泡位置每帧跟随立绘屏幕坐标重新计算，不参与 `wait for` 等待系统（纯视觉追踪）。立绘不可见时按 `fallback_mode` 处理，不报错，开发者自行决定降级策略。`together` 块内多角色气泡各自独立定位，重叠检测不由引擎处理。

**`color_matrix` 独立于 `transform`**：`color_matrix` 是显示对象的后处理步骤，不参与 keyframe 插值，不受 `transform` 的 `repeat` / `mode` 影响。`transition_matrix` 参数提供过渡时间，内部实现为线性插值，不走 `transform` 系统的 easing。

**Layer transform 始终后台动画**：`layer transform` 不参与 `wait for all`，等价于 `repeat forever` 的后台动画。需要等待时必须用 `(handle=)` + `wait for`。理由：层是全局共享的，前台动画等待语义在层级操作上易产生意外阻塞。

**图片预加载静态分析范围**：编译器只分析静态可知的 `show` / `scene` / `expression` 资源路径（字符串字面量或 `define image` 引用），`$` 前缀动态引用标记为不可预测，不纳入预加载提示表。开发者可用 `preload` 指令手动补充动态场景的预加载。

**音频滤波器参数插值**：`filter music (reverb(room=0.6)) 1.0` 中的过渡时间是在当前参数值和目标参数值之间线性插值，不是渐入渐出。对不同滤波器类型之间的切换（如从 `reverb` 切到 `lowpass`），无法插值，直接切换并输出警告。

**成就 id 唯一性**：`define achievement` 的 id 在引擎启动时检查全局唯一性，冲突时报 `AxnCompileError`。`unlock achievement` 传入未定义的 id 时报 `AxnRuntimeError`（与 `AxnJumpError` 同级别处理）。

**`preferences` 钩子实现**：写入 `preferences.xxx` 通过 `__setattr__` 钩子触发对应系统副作用（音量调整、窗口模式切换等）。读取 `preferences.xxx` 直接返回存储值，无副作用。`preferences` 对象本身不可通过 `store` 访问，`store["preferences"]` 会触发 `AxnNameError`。

**场景回放 `store` 隔离**：回放期间创建 `store` 的浅拷贝副本作为执行环境，回放结束后丢弃，原始 `store` 不受影响。回放内的所有写入操作（`$` 赋值、`with store`）作用于副本，不影响原始状态。`persistent` 在回放期间**不隔离**（回放可以正常解锁成就等），开发者需自行确保回放内的 `unlock` 操作是幂等的。

**HTTP Fetch 域名白名单**：白名单只在发布包（`axn build`）中生效，开发模式无限制。绕过方式（如通过 `python:` 块直接调用 `httpx`）不受白名单约束——白名单是 `engine.fetch` API 层的限制，不是系统级网络限制。开发者通过 Python 直接发请求时自行负责合规性。

**平台变体在发布包中固化**：`axn build --platform android` 时，编译器将 `engine.variant("pc")` 等在目标平台上已知为 `false` 的分支标记为死代码并剥离。`engine.variant("touch")` 在 PC 触摸屏上可能为 `true`，不固化。

**自动化测试的 `store` 状态**：测试脚本的 `assert store:` 读取当前执行中的 `store`，与游戏进程共享同一 `store`，无隔离。测试间如需状态隔离，每个测试文件从 `start` label 重新开始。

---

#### 转场扩展（Transition Extension）

**过渡层级作用域**

`scene` 的 `with` 参数默认只作用于背景层，立绘层独立不参与：

```apy
scene bg_room 0.5 (with=fade)                        # 只有背景层 fade，立绘不动
scene bg_room 0.5 (with=fade, all)                   # 整个画面（含立绘）一起 fade
scene bg_room 0.5 (with=fade, layer=[bg, effect])    # 指定层参与过渡
```

`transition` 独立指令始终作用于整个画面，语义不变。

**并行过渡叠加规则**

分两类处理：

- **对象级过渡**（`show`/`hide` 的 `enter`/`exit`）：作用于单个对象的 surface，多个同时跑互不干扰，正常合成
- **全屏级过渡**（`scene` 的 `with`，`transition` 指令）：同一时间只允许一个运行，后触发的覆盖前一个，前一个立即中止并输出警告：

```
AxnWarning: [transition] Full-screen transition 'fade' interrupted by 'wipe'.
  Only one full-screen transition can run at a time.
  → scene.apy, line 12
```

**自定义过渡注册**

自定义过渡通过注册机制进入过渡库，注册后与内置过渡完全等价：

```python
# startup 块内注册
startup:
    python:
        from axn_plus.apy.transition import TransitionLibrary
        TransitionLibrary.register("slide_from_top", SlideFromTop)
        TransitionLibrary.register("glitch_in", GlitchTransition)
```

注册后支持裸名字引用和参数化调用：

```apy
show autumn (enter=slide_from_top)           # 裸名字，使用 __init__ 默认参数
show autumn (enter=slide_from_top(0.3))      # 带参数，等价于 SlideFromTop(0.3)
scene bg_room (with=glitch_in(speed=2.0))
```

传实例的写法（`enter=SlideFromTop(0.3)`）依然保留，两种方式并存：注册适合复用，传实例适合一次性使用。未注册的名字在引擎启动时报 `AxnAssetError`。

---

#### 拖放系统（Drag & Drop）

**核心模型：拖拽源 + 放置目标，事件驱动。**

**`draggable`：声明可拖拽控件**

```apy
gui inventory_item(item):
    draggable (
        data    = item,                      # 拖拽携带的数据
        preview = drag_preview(item),        # 跟随鼠标的预览控件
        layer   = effect                     # 预览控件渲染层，默认最顶层
    )
    image item.icon (size=(48, 48))

# 预览控件定义
gui drag_preview(item):
    image item.icon (size=(40, 40), alpha=0.7)
```

**`droptarget`：声明放置目标**

```apy
gui equipment_slot(slot_id):
    droptarget (
        accept_type = "equipment",           # 简单类型过滤，GUI 可解析
        on_drop     = dropped_on_slot
    ):
        if store["equipment"][slot_id]:
            image store["equipment"][slot_id].icon
        else:
            rect (size=(48, 48), color=#333333, border_radius=4)

# 复杂过滤条件退回 lambda，降级代码节点
gui any_slot:
    droptarget (
        accept  = lambda data: data.type == "equipment" and data.level <= store["max_level"],
        on_drop = dropped_on_slot
    ):
        ...
```

`on_drop` 处理器接收被拖拽的 `data` 和目标控件参数：

```apy
on_drop dropped_on_slot(data, slot_id):
    $ equip(store["equipment"], slot_id, data)
    emit "equipment_changed"
```

**`droptarget` 悬停视觉反馈**：复用条件样式系统，新增 `drag_over` 状态关键字：

```apy
gui equipment_slot(slot_id):
    droptarget (...):
        style:
            drag_over: border (2, #ff8800)    # 有效拖拽悬停时的高亮
```

不设置时使用默认样式（无视觉变化）。

**自由定位拖拽（拼图类）**

不走 `droptarget`，直接拿坐标：

```apy
gui puzzle_piece(piece):
    draggable (
        data       = piece,
        free       = true,              # 自由定位，不需要 droptarget
        on_drag    = piece_dragging,    # 拖拽中每帧触发，携带当前坐标
        on_release = piece_released     # 释放时触发，携带最终坐标
    )
    image piece.image
```

**`moveable`：窗口拖移**

专为"拖动控件本身改变位置"设计，与 `draggable` 独立，可共存：

```apy
gui floating_log:
    moveable                              # 最简写法，整个控件可拖移
    moveable (handle=title_bar)           # 只有 title_bar 区域可拖动
    moveable (
        bounds  = screen,                 # 移动范围限制：screen / parent / none
        snap    = (8, 8),                 # 吸附网格（可选）
        persist = store["log_pos"],       # 位置持久化到 store（可选）
        persist_read  = true,             # 初始化时从 store 读位置（默认 true）
        persist_write = true              # 销毁时写回 store（默认 true）
    )
    panel:
        hstack as title_bar:
            text "日志"
            spacer grow
            button "×" on_click: hide gui floating_log
        scroll vertical:
            slot children
```

`persist` 默认隐式双向绑定：初始化时读，销毁时写。`persist_read` / `persist_write` 用于显式控制单向行为。`persist` 但 store 里尚无值时，fallback 到声明位置。

`moveable` 与 `draggable` 共存：

```apy
gui inventory_item(item):
    moveable                    # 可移动自身位置
    draggable (data=item)       # 也可拖到 droptarget 触发事件
```

---

#### 文本着色器（Text Shader）

**设计定位**：在现有标签系统上增加复杂视觉效果（渐变、描边、发光、波浪等），CPU 端 surface 操作。Pygame 后端主推此方案；Qt 后端 GPU shader 作为可选扩展，不在核心设计范围内。

**`<shader>` 标签**

```apy
autumn: "这是<shader=gradient(#ff8800, #ffffff)>渐变文字</shader>。"
autumn: "这是<shader=outline(color=#000000, width=2)>描边文字</shader>。"
autumn: "这是<shader=glow(color=#ff8800, radius=4)>发光文字</shader>。"
autumn: "这是<shader=wave(amplitude=3, speed=2.0)>波浪文字</shader>。"
autumn: "这是<shader=shadow(color=#00000088, offset=(2,2))>阴影文字</shader>。"

# 组合多个效果
autumn: "这是<shader=[outline(#000000, 2), glow(#ff8800, 4)]>组合效果</shader>。"
```

**style 系统集成**

```apy
style glitch_text:
    shader: [
        gradient(#ff0000, #ffffff)
        wave(amplitude=2, speed=3.0)
    ]

style title_text:
    shader: outline(#000000, width=3)
    font_size 32

autumn: "重要台词" (style=glitch_text)
text "游戏标题" (style=title_text)
```

**内置着色器列表**

| 着色器 | 参数 | `is_static` |
|--------|------|-------------|
| `gradient(color1, color2)` | 渐变色 | `true` |
| `outline(color, width=2)` | 描边 | `true` |
| `shadow(color, offset=(2,2))` | 阴影 | `true` |
| `glow(color, radius=4)` | 发光 | `false` |
| `wave(amplitude=3, speed=2.0)` | 波浪形变 | `false` |

**自定义着色器**

继承 `TextShader`，与 `Transition` 扩展方式保持一致：

```python
class RainbowShader(TextShader):
    is_static = False       # 有 time 依赖，不缓存
    
    def __init__(self, speed=1.0):
        self.speed = speed

    def apply_char(self, surface, char_index, total_chars, time):
        # char_index / total_chars 均相对于已显示部分
        # 打字机播放中 total_chars 随时间增长，效果自然展开
        hue = (time * self.speed + char_index * 0.1) % 1.0
        color = hsv_to_rgb(hue, 1.0, 1.0)
        return colorize(surface, color)

    def apply_block(self, surface, time):
        # 块级处理入口，gradient 等需要整行宽度的效果在此实现
        return surface
```

`TextShader` 提供两个入口：`apply_char`（字符级）和 `apply_block`（块级），均有默认实现（直接返回 surface），着色器只需覆盖需要的那个。

**缓存机制**：`is_static=True` 时引擎只计算一次，结果缓存到控件销毁；`is_static=False` 时每帧更新。内置静态着色器：`outline`、`shadow`、`gradient`（无动画）。

注册后与内置着色器完全等价：

```apy
startup:
    python:
        TextShaderLibrary.register("rainbow", RainbowShader)

autumn: "彩虹<shader=rainbow(speed=2.0)>文字</shader>出现了。"
```

---

#### 多角色对话（Multi-character Dialogue）

**`together` 块：多角色同时说话**

```apy
together:
    autumn: "我们一起说！"
    sophia: "我们一起说！"

together:
    autumn: "我说我的。" (happy)
    sophia: "我说我的。" (sad)
    @ "两人同时开口。"          # 旁白也可以参与
```

`together` 块内所有对话行同时触发，共享同一个等待点，用户点击一次推进所有。各角色语音独立播放，点击时打断所有正在播放的语音。各角色对话修饰符正常修改各自的持久表情状态。

**`line=` 语法糖**：`together` 和 `chorus` 块支持 `line=` 块级参数，用 `|` 分隔，按顺序对应块内各行施加文本标签效果。单个值（无 `|`）时应用到整块所有行；多个值时按行对应，多出来的段静默忽略，不足的行无效果。同一段内可用逗号叠加多个标签：

```apy
together (line="<nw>|<w>"):
    autumn: "第一句。"    # 应用 <nw>，说完立即推进不等点击
    sophia: "第二句。"    # 应用 <w>，中途暂停等点击后继续

together (line="<fast>"):
    autumn: "整块都快速显示。"
    sophia: "整块都快速显示。"

together (line="<nw>,<fast>|<w>"):
    autumn: "第一句同时应用 nw 和 fast。"
    sophia: "第二句应用 w。"
```

`line=` 是纯语法糖，展开后等价于在每行对话文本头部插入对应标签，不引入新的渲染逻辑。

**`inp=` 注入参数**：`together` 和 `chorus` 块支持 `inp=` 块级参数，接受 `store` 变量（必须是 `dict`），key 为修饰符参数名。`dict` 内容作为额外修饰符统一应用到块内所有行，等价于每行都加了对应的具名参数：

```apy
$ shared_mods = {"speed": store["text_speed"], "voice_delay": 0.1}

together (inp=shared_mods):
    autumn: "你好。"    # 等价于 (speed=store["text_speed"], voice_delay=0.1)
    sophia: "再见。"    # 同上
```

`inp=` 的优先级低于行内修饰符——行内显式声明的参数覆盖 `inp=` 提供的同名参数：

```apy
together (inp={"speed": 0.5}):
    autumn: "慢速。"              # speed=0.5
    sophia: "正常速度。" (speed=1.0)  # 行内覆盖，speed=1.0
```

`line=` 和 `inp=` 可以同时使用，互不干扰：

```apy
together (line="<nw>|<w>", inp={"speed": 0.7}):
    autumn: "第一句。"
    sophia: "第二句。"
```

`inp=` 的值必须是 `dict` 类型，传入非 `dict` 时运行时抛 `AxnRuntimeError`：

```
AxnRuntimeError: [together] 'inp=' requires a dict, got str.
  → scene.apy, line 8
```

`together` 内出现 `nowait` 时忽略并输出警告：

```
AxnWarning: [together] 'nowait' has no effect inside a 'together' block.
  'together' manages its own wait point.
  → scene.apy, line 8
```

`together` 整体作为一个回滚单元。

**对话框布局**：多个对话框同时显示，位置策略在 `options_window.apy` 中配置：

```apy
engine:
    together:
        layout = "follow_sprite"    # 对话框跟随各自立绘位置（默认）
        layout = "split"            # 画面左右分割
        layout = "stack"            # 垂直堆叠
```

**交叉快速对话**

使用现有 `nowait` 或 `parallel` 处理，无需新语法：

```apy
# nowait：说完立刻推进，无重叠
autumn: "你说什么？" (nowait)
sophia: "我说——"   (nowait)
autumn: "什么！"

# parallel：真正的重叠，A还没说完B就开口
parallel:
    track dialogue (interactive):
        autumn: "你说什——"
    track sophia_line:
        wait 0.5
        sophia: "闭嘴！"
```

**`chorus` 块：合唱**

多个角色说同一句话，显示在同一个对话框：

```apy
chorus autumn sophia:
    "我们绝不放弃！"

chorus autumn sophia kenji:
    "这一刻，所有人都明白了。" (speed=0.8)
```

注意：`narrator` 不是角色，不允许出现在 `chorus` 成员列表中；`chorus` 只接受已用 `define char` 定义的角色名，使用未定义角色时解析期报错：

```
AxnParseError: 'narrator' is not a defined character and cannot be used in 'chorus'.
  Use '@' or 'narrator:' for narrator lines outside 'chorus'.
  → scene.apy, line 8
```

对话框显示所有角色的名字，格式在 `options_window.apy` 中配置：

```apy
engine:
    chorus:
        name_format    = "join"     # "autumn & Sophia"（默认）
        name_format    = "list"     # "autumn, Sophia"
        name_separator = " & "      # join 模式下的分隔符
```

语音：所有参与角色同时播放各自的语音文件，走各自的 `voice_prefix`：

```apy
chorus autumn sophia:
    "我们绝不放弃！" (voice="chorus_001")
    # autumn: vo/autumn/chorus_001.ogg
    # sophia: vo/sophia/chorus_001.ogg
```

`chorus` 整体作为一个回滚单元。

**旁白在非交互轨道**

`@` 和 `narrator:` 不算对话行，允许出现在非交互轨道，无需新语法：

```apy
parallel:
    track dialogue (interactive):
        autumn: "她慢慢地说着。"
    track narration:
        @ "窗外的雨还在下。"    # 旁白不需要用户输入，不产生输入路由冲突
```

---

#### 启动屏幕（Startup Screen）

在 `options_window.apy` 中声明 `startup_sequence` 块，引擎启动时自动执行，早于 `flow.apy` 的 `start` label。

**执行顺序**：

```
引擎初始化
  → startup (before) 块
  → startup 块
  → startup (after) 块
  → startup_sequence（splash → warning → loading）
  → flow.apy 的 start label
```

**`startup_sequence` 完整示例**：

```apy
# options_window.apy

startup_sequence:
    input disable:
        splash axn_logo (duration=2.0)                    # 不可跳过
        splash "ui/splash/aniplex.png" (duration=3.0)     # 不可跳过

    splash "ui/splash/studio.png" (duration=2.5, skippable=true)

    warning (skippable=false, once=true):
        vstack gap=24:
            image "ui/rating_18.png"
            translate zh:
                text "本作品含有成人内容，请确认您已年满18岁。"
            translate en:
                text "This title contains adult content. Please confirm you are 18 or older."
        hstack gap=16:
            translate zh:
                button "确认，继续" on_click: Return()
                button "离开" on_click: engine.quit()
            translate en:
                button "Confirm" on_click: Return()
                button "Leave" on_click: engine.quit()

    loading:
        background "ui/loading_bg.png"
        progress_bar bind=engine.load_progress (style=loading_bar)
        tips (interval=3.0):
            "提示：按住Ctrl可以跳过对话。"
            "提示：右键打开历史记录。"
            "提示：F12截图。"
```

**`splash` 指令**

支持静态图片、视频、图像序列三种形式：

```apy
# 静态图片
splash "ui/splash/logo.png" (
    duration   = 3.0,
    skippable  = true,
    fadein     = 0.5,
    fadeout    = 0.5,
    background = #000000
)

# 引擎内置 logo（保留关键字）
splash axn_logo (duration=2.0, skippable=true)

# 视频
splash video "ui/splash/intro.mp4" (
    skippable  = false,
    fadein     = 0.3,
    fadeout    = 0.3,
    background = #000000
)

# 图像序列 + 声音
splash sequence "ui/splash/frames/" (
    fps        = 24,
    audio      = "ui/splash/jingle.ogg",
    skippable  = true,
    fadein     = 0.2,
    fadeout    = 0.2
)

# 非约定命名时显式声明 pattern
splash sequence "ui/splash/frames/" (
    pattern = "frame_{:04d}.png",
    fps     = 24,
    audio   = "ui/splash/jingle.ogg"
)
```

图像序列文件命名约定：目录内按文件名字典序排列（`000.png`、`001.png`……），引擎自动扫描。

`skippable=true` 时点击触发 `fadeout` 后推进，不硬切。图像序列的 `audio` 走独立 `splash_audio` 内部通道，不影响游戏音频状态，跳过时立刻停止。

**`warning` 块**

平台合规用，内容完全自定义，走完整 `gui` 控件语法。`once=true`（默认）时确认后写入 `persistent`，之后启动直接跳过；`once=false` 时每次启动都显示。`Return()` 推进到下一个 `startup_sequence` 步骤。`translate` 在 `warning` 块内走与脚本层相同的语言选择逻辑。

**`loading` 块**

资源加载在后台线程进行，`loading` 块显示期间同步推进。未声明 `loading` 块时，引擎利用最后一个 `splash` 显示期间后台加载，`splash` 在加载完成前不消失。`engine.load_progress` 是引擎内置的 `0.0`-`1.0` 进度变量。

`tips` 是 `loading` 块内的可选子块，按 `interval` 秒数轮播提示文字。

**`input disable` 与 `startup_sequence`**

`input disable` 块可直接用于 `startup_sequence`，块内的 `splash` 完全禁用输入（包括跳过），`skippable` 参数在此语境下无效并输出警告：

```
AxnWarning: [startup] 'skippable' has no effect inside an 'input disable' block.
  Input is already disabled.
  → options_window.apy, line 8
```

**开发模式跳过**

```apy
engine:
    dev:
        skip_startup = true    # 跳过整个 startup_sequence，含 skippable=false 的步骤
    show_axn_logo = false      # 商业项目禁用引擎内置 logo
```

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

## 用户偏好系统（Preferences）

`preferences` 是引擎维护的全局单例，存储**玩家系统偏好**，与游戏内容数据（`store`）和跨存档内容数据（`persistent`）严格分离。游戏退出时自动持久化，下次启动自动恢复。

**`preferences` vs `store` vs `persistent` 的边界：**

| 对象 | 存储内容 | 生命周期 | 例子 |
|------|---------|---------|------|
| `store` | 当前存档槽的游戏状态 | 跟随存档槽 | 好感度、当前天数、flag |
| `persistent` | 跨存档的游戏内容数据 | 永久 | 已解锁 CG、已通关路线 |
| `preferences` | 玩家系统偏好 | 永久，跨游戏实例 | 音量、文字速度、跳过策略 |

### 内置偏好项

引擎管理以下内置偏好项，在 `options_window.apy` 中声明默认值：

```apy
# options_window.apy
preferences:
    # 文字
    text_speed        = 1.0       # 文字显示速度倍率（0.5 = 半速，0 = 瞬显）
    skip_mode         = "seen"    # "seen"（只跳已读）/ "all"（跳全部）
    skip_transitions  = true      # 跳过模式下略过过渡动画
    skip_voice        = true      # 跳过模式下略过语音
    auto_forward_time = 2.0       # Auto 模式无语音时的等待秒数
    auto_delay        = 0.3       # Auto 模式语音播完后额外等待秒数

    # 音量（0.0–1.0）
    music_volume      = 0.8
    sound_volume      = 0.8
    voice_volume      = 0.9
    ambient_volume    = 0.6

    # 显示
    fullscreen        = false
    display_index     = 0         # 多显示器时使用哪块屏幕
    renderer          = "auto"    # "auto" / "hardware" / "software"

    # 无障碍
    self_voicing      = false     # TTS 自动朗读（见无障碍章节）
    self_voicing_rate = 1.0       # TTS 语速倍率
```

### 脚本层访问

```apy
# 读取
$ speed = preferences.text_speed
$ vol   = preferences.music_volume

# 写入（立即生效，自动持久化）
$ preferences.text_speed   = 0.5
$ preferences.music_volume = store["saved_vol"]
```

引擎在写入时立即应用对应系统（改音量时立即调整播放器，改全屏时立即切换窗口模式）。

### 开发者自定义偏好项

在 `options_window.apy` 的 `preferences:` 块中添加自定义条目，引擎自动纳入持久化管理：

```apy
preferences:
    music_volume  = 0.8       # 内置项
    show_cg_name  = true      # 自定义项：画廊中是否显示 CG 名称
    dialogue_font = "default"
```

自定义项通过 `preferences.show_cg_name` 访问，与内置项完全一致。

### 音量通道绑定

内置音量偏好与音频通道自动绑定：

| 偏好项 | 自动作用于 |
|--------|----------|
| `music_volume` | `music` 通道 |
| `sound_volume` | `sound` 通道 |
| `voice_volume` | `voice` 通道 |
| `ambient_volume` | `ambient` 通道 |

自定义通道在 `options_window.apy` 中声明绑定关系：

```apy
engine:
    audio:
        channel_bindings:
            bg_layer = preferences.bg_layer_volume
```

### 关键设计决策

**`preferences` 不写入 `store`**：读档时 `store` 恢复到存档时的游戏状态，`preferences` 保持玩家当前设置不变，不随存档回退。

**写入立即生效**：`$ preferences.music_volume = 0.5` 执行后立刻调整播放音量，引擎内部通过属性钩子实现，无需额外 `apply` 调用。

**`text_speed = 0`**：瞬间显示全部文字，等价于对每行文本触发 `<fast>` 标签效果。

**跨版本兼容**：新版本新增的偏好项，旧持久化文件中缺失时使用 `options_window.apy` 声明的默认值补齐，不报错。

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

**`@saveable` 静态字段扫描**：`@saveable` 装饰器在**类定义时**（而非运行时存档时）对 `__init__` 的字段赋值做静态扫描，发现明显不可序列化的类型（`lambda`、文件句柄、`socket`、`threading.Lock` 等）时立即输出警告，不等到存档时才报错：

```
AxnWarning: [save] @saveable class 'QuestState' contains field 'handler'
  of type 'function', which may not be picklable.
  Consider excluding it via __save__ / __load__ or using Saveable base class.
  → logic/quest.py, line 8
```

扫描基于 `__init__` 的类型注解（`field: type`）和默认值类型推断，不做运行时实例分析。无类型注解且默认值为安全类型时不警告；有注解为危险类型时警告。此扫描为尽力而为，不保证覆盖所有场景，复杂场景退回 `Saveable`。

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
  Cannot serialize 'QuestState' during save operation.
  → save triggered at: scene.apy::morning_scene, checkpoint "第一章"

Declare it with @saveable or inherit from Saveable:
  Option 1 (simple):   @saveable class QuestState: ...
  Option 2 (versioned): class QuestState(Saveable): __version__ = 1 ...
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

**控件本身不预设视觉。** `button` 不携带任何默认背景或颜色，`style button` 全局推导注入项目默认样式，不写则自然是素的。调用点推荐通过 `style=` 传入 `mixin` 名来覆盖样式。

**兼容写法（容错）**：`button` 的 `style=` 参数也接受内联样式属性，引擎运行时输出警告，可 ignore。内联样式绕过样式系统的优先级链，产生的覆盖冲突由开发者自行负责。推荐写法始终是先定义 `mixin`，再通过 `style=` 传入。GUI 编辑器对调用点使用内联样式时以黄色警告标注"建议改用 mixin"，但不阻止保存。

**三条样式传递路径：**

```
路径一：全局 style 自动推导（零配置，按命名约定自动绑定）
路径二：gui 定义内 apply mixin（项目级复用）
路径三：调用点 style= 参数（实例级覆盖，推荐只接受 mixin；传入内联属性会警告可 ignore）
```

优先级：`调用点 style= > gui 内 apply > 自动推导 style > theme token`

**`style=` 推荐只接受 `mixin`，不接受 `style`。** `style` 是全局推导用的，`mixin` 是手动 apply 用的，两者职责不混。传入 `style` 名时引擎输出警告并按 `mixin` 语义处理，可 ignore。

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

**`frame`**：九宫格拉伸图片，只拉伸中间区域，四角保持原始像素，适合对话框、面板、按钮等需要自适应尺寸的 UI 元素：

```apy
frame "ui/panel.png" (margin=(12, 12, 12, 12))   # 上右下左边距（px）
frame "ui/panel.png" (margin=12)                  # 四边相同
frame "ui/panel.png" (margin=(12, 8))             # 上下12，左右8
```

`frame` 作为 `background` 属性的值使用（最常见）：

```apy
gui dialogue_box:
    background frame("ui/panel.png", margin=16)
    padding (20, 12)
    slot children

gui option_button(label):
    style:
        background frame("ui/btn_normal.png", margin=8)
        hovered: background frame("ui/btn_hover.png", margin=8)
        pressed: background frame("ui/btn_press.png", margin=8)
    text label (anchor=center)
```

`frame` 也可以作为独立控件使用（作为容器背景）：

```apy
frame "ui/panel.png" (margin=12, size=(300, 200)):
    slot children
```

**`frame` vs `image`**：`image` 直接缩放整张图，圆角等装饰性元素会随尺寸变形；`frame` 只拉伸中间，四角像素完全保留，是 UI 面板的正确做法。

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

#### 头像（Side Image）

`side_image` 是角色对话时显示在对话框旁边的小头像图，通常在对话框左侧。引擎自动根据当前说话角色切换，无需手动管理。

**角色声明**（在 `define char` 中）：

```apy
define char autumn:
    name "autumn"
    side_image "ui/side/autumn_neutral.png"     # 静态单张

    # 或：跟随表情状态切换（states 模型）
    side_image:
        states:
            neutral  "ui/side/autumn_neutral.png"
            happy    "ui/side/autumn_happy.png"
            sad      "ui/side/autumn_sad.png"
        default_expression "neutral"

    # 或：引用角色自身的 states 定义（自动同步表情）
    side_image auto      # 使用与角色 states 同名的 ui/side/{char_name}_{state}.png
```

`side_image auto` 时，引擎按 `ui/side/{角色名}_{state}.png` 命名约定查找，找不到时回退到 `ui/side/{角色名}.png`，仍找不到则不显示（不报错）。

**渲染位置**：`side_image` 渲染在对话框内，位置和尺寸在对话框 UI 模板中通过 `slot side_image` 声明：

```apy
# ui/dialogue_box.apy
gui DialogueBox:
    hstack gap=12:
        slot side_image (size=(80, 80))     # 头像占位区，引擎自动填充当前角色头像
        vstack:
            slot name_label
            slot dialogue_text
```

引擎在渲染当前对话行时，将说话角色的 `side_image`（当前表情对应的图片）注入到 `slot side_image` 区域。旁白行（`@` / `narrator:`）说话时，`slot side_image` 渲染为空（由 UI 模板决定布局如何处理空插槽）。

**`same_turn` 行为**：同一轮对话（同一等待点内）切换表情时，`side_image` 是否随之更新：

```apy
# options_window.apy
engine:
    side_image:
        same_turn = true     # true（默认）：表情切换时立即更新头像；false：等到下一行才更新
        show_during_nvl = false   # NVL 模式下是否显示头像（默认 false）
```

**Round-Trip**：`side_image:` 子块在角色定义积木块中显示为头像配置面板；`auto` 关键字显示为"自动命名约定"标记；`slot side_image` 在对话框模板编辑器中显示为头像占位符，尺寸可调。

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
avatar src="portraits/autumn.png" size=64
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

#### 声明语法

`screen` 支持函数签名风格参数，与 `label`、`gui` 保持一致：

```apy
screen hud:                          # 无参数
    ...

screen common_header(title):         # 有参数
    text title (font_size=24, anchor=top_center)

screen save_slot(slot_id, label="空存档"):   # 带默认值
    ...
```

#### 调用语法

**`show screen`**：非阻塞显示，screen 持续存在直到手动 `hide`：

```apy
show screen hud                      # 显示，不阻塞执行流
show screen hud (layer=effect)       # 指定层级
hide screen hud                      # 手动关闭
```

**`call screen`**：阻塞执行流，等 screen 关闭后继续：

```apy
call screen save_menu                # 阻塞，不接返回值
call screen save_menu as result      # 阻塞，接返回值
```

`call screen` 默认在关闭时自动销毁。需要关闭后保留（如动画过渡期间）时，加显式参数：

```apy
call screen save_menu (keep)         # 关闭后保留，需手动 hide
```

screen 内部通过 `Return()` 传递返回值并触发关闭：

```apy
screen confirm_dialog:
    button "确认" on_click: Return("confirm")
    button "取消" on_click: Return("cancel")
    button "关闭" on_click: Return()    # 无值退出，result 为 None
```

```apy
call screen confirm_dialog as result
if result == "confirm":
    jump delete_save
```

**`Return()` 与 `Action()`**：`Return()` 是跨层边界的信号，用于从控件事件（`on_click` 等）中退出当前 screen 并向调用方传值，是 `call screen` 和 `modal show` 的唯一传值机制。引擎不引入 `Action()` 包装器——Ren'Py 需要它是因为 screen 语言与 Python 割裂，Axn-Plus 的 `gui` 块内可以直接写 `on_click: $ do_something()`，Python 就是 Python，不需要桥接层。

**三种 `as` 接返回值机制**：`.apy` 里有三处使用 `as` 接返回值，传值机制各不相同，注意不要混淆：

| 场景 | 写法 | 传值机制 |
|------|------|---------|
| `call label()` | `call morning_scene() as result` | label 内用 `return expr` 关键字传值 |
| `call screen` / `modal show` | `call screen confirm_dialog as result` | screen 内用 `Return(expr)` 函数传值 |
| `menu as` | `menu as answer:` | 选项内用 `->` 右侧表达式传值 |

三者的执行上下文本来就不同，不强行统一。在 label 内写 `Return("x")` 或在 menu 选项内写 `return expr` 均无效，解析期报错：

```
AxnParseError: 'Return()' is not valid inside a 'label' block.
  Use 'return expr' to return a value from a label.
  'Return()' is only valid inside 'screen' or 'gui' controls for 'call screen' / 'modal show'.
  → scene.apy, line 8

AxnParseError: 'return' is not valid inside a 'menu' option.
  Use '-> expr' to specify the return value for 'menu as'.
  → scene.apy, line 12
```

**`call screen` 与 `modal show` 的区别**：

| | `call screen` | `modal show` |
|---|---|---|
| 执行流 | 阻塞 | 阻塞 |
| 焦点接管 | ❌ 不接管 | ✅ 接管 |
| 输入屏蔽 | ❌ 不屏蔽底层 | ✅ 屏蔽底层 |
| 返回值 | `as result` | `as result` |
| 适用场景 | 普通界面切换 | 需要强制用户响应的模态交互 |

#### 生命周期

`screen` 默认挂在 `ui` 层。`scene` 切换时清除（与 `ui` 层上的其他元素行为一致）。需要持久时显式指定持久层：

```apy
show screen hud (layer=persistent_hud)   # 手动指定持久层，scene 不清除
```

多个 `show screen` 叠加显示，不互相替换：

```apy
show screen hud
show screen dialogue_box    # 与 hud 同时显示，叠加在上方
```

#### Python 块限制

`screen` 块内允许 `$` 和 `python:` 块，但**复杂逻辑必须先用 `$` 算好，不能内联**。简单变量引用和条件判断可以直接用：

```apy
screen status_panel:
    # 简单引用：直接内联，OK
    text "{store['player_name']}"
    if store["hp"] <= 0:
        text "已阵亡" (color=#ff0000)

    # 复杂逻辑：先算好
    $ status_text = get_status_summary(store["flags"])
    text status_text
```

绝对定位完全可用，同时扩展了语义化相对定位能力。

```apy
screen hud:
    pin top_right:
        text "00:30"
    pin bottom center:
        dialogue_box
```

**局部 `style`**：`screen` 内部可定义局部 `style`，只在此 `screen` 内生效，不污染全局：

```apy
screen pause_menu:
    style button:                  # 只在此 screen 内生效
        background #333333
        size (120, 36)

    button "继续" on_click: jump resume_game
    button "存档" on_click: jump save_menu
```

局部 `style` 优先级高于全局同名 `style`。

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

`gui` 定义可复用的控件组件，职责是**控件的封装与复用**。既可以嵌入 `screen` 内使用，也可以在脚本层单独 `show` / `hide`。

#### 调用语法

**嵌入 `screen` 内**：直接调用，无需 `show`：

```apy
screen hud:
    health_bar(80, 100)
    health_bar(60, 100, color=#0000ff)
```

**脚本层单独使用**：通过 `show gui` / `hide gui` 控制：

```apy
show gui hud                         # 显示，默认挂 ui 层（持久）
show gui effect_overlay (layer=effect)   # 指定层，跟随场景
hide gui hud                         # 手动隐藏
```

`gui` 没有 `call gui` 用法——阻塞与返回值是 `screen` 的职责。需要阻塞交互时，把 `gui` 控件嵌入 `screen` 内，再用 `call screen`。

#### 生命周期

**`gui` 是"控件"概念，`screen` 是"画面"概念，两者生命周期语义不同：**

| | `show gui` | `show screen` |
|---|---|---|
| 默认层 | `ui` 层（持久） | `ui` 层 |
| `scene` 切换 | **不清除**（持久，适合 HUD 等常驻控件） | 清除 |
| 手动关闭 | `hide gui 名字` | `hide screen 名字` |

**嵌入 `screen` 内的 `gui`**：生命周期跟 `screen` 走，`screen` 关闭时一并销毁。默认与 `screen` 同层，需要独立层级时在 `gui` 块内显式指定：

```apy
screen battle_hud:
    health_bar(80, 100)              # 跟 screen 同层，screen 关闭时销毁

gui persistent_notice:
    layer ui                         # 显式指定层，即使在 screen 内也保持独立生命周期
    ...
```

**`gui` 可以放在非 `ui` 层**，此时生命周期跟层走：

```apy
gui effect_overlay:
    layer effect        # 挂 effect 层，scene 切换时清除

gui hud:                # 默认 ui 层，持久，scene 不清除
    ...
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
    follow autumn offset=(0, -20):
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
- 同一控件内 `slot` 名称不可重复，引擎启动时检查并报错：

```
AxnParseError: Duplicate slot name 'body' in 'gui BaseCard'.
  Each slot within the same control must have a unique name.
  → ui/components.apy, line 24
```

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

**预定义状态关键字**

六个内置状态，直接用状态名作为缩进键：

| 状态 | 触发条件 |
|------|---------|
| `hovered` | 鼠标悬停在控件上 |
| `pressed` | 鼠标按下未松开（瞬时） |
| `active` | 持续激活态（如按住不松、长按操作） |
| `selected` | 静态选中态（radio 选中、tab 激活，由 `bind` 或 `on_click` 手动维护） |
| `focused` | 键盘/手柄焦点在此控件上 |
| `disabled` | 控件不可交互 |

`active` 与 `selected` 的区别：`pressed` 是鼠标按下的瞬时态；`active` 是按住不松的持续态（适合长按类操作）；`selected` 是开发者显式维护的静态选中态，与鼠标操作无关。

```apy
gui option_button(label):
    style:
        background #444444
        hovered:   background #555555
        pressed:   background #3a3a3a
        active:    background #1a6622
        selected:  background #226622
        focused:   border (2, #aaaaff)
        disabled:  background #222222
```

**遮罩层（`overlay` 与 `alpha`）**

两种遮罩语义不同，均可用于任意状态：

- **`overlay`**：在控件表面叠加一层半透明颜色，子元素不受影响，适合"蒙一层色"的场景
- **`alpha`**：降低整个控件（含子元素）的透明度，适合整体淡出的场景

```apy
gui option_button(label):
    style:
        background #444444
        disabled:
            background #222222
            overlay #00000055    # 半透明黑色蒙层叠在表面
        # 或者：
        disabled:
            alpha 0.4            # 整体降透明度（含文字、图标等子元素）
```

两者可以同时使用，`overlay` 先合成，`alpha` 作用于合成结果：

```apy
disabled:
    overlay #00000044
    alpha 0.7
```

`overlay` 接受 `#rrggbbaa` 格式（含 alpha 通道），`alpha` 接受 `0.0`–`1.0` 浮点数。两者均支持 `$token` 引用。

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

`when` 允许的表达式限制：预定义状态关键字 **或** `state` / `store` 变量的简单比较，支持 `and` / `or` / `not` 组合，不允许任意 Python 表达式，保证 GUI 编辑器完整可解析。复杂条件退回 `if` 块或 Python。

```apy
gui option_button(label):
    style:
        background #444444
        when hovered and not disabled:
            background #555555
        when selected or focused:
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

**`mixin` 继承**：支持 `extends` 继承另一个 `mixin`，子类覆盖父类同名属性，未声明属性全部继承：

```apy
mixin base_interactive:
    hovered:  background #555555
    pressed:  background #333333

mixin danger_interactive extends base_interactive:
    hovered:  background #ff3333   # 覆盖
    # pressed 继承 base_interactive
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

**`apply` 条件应用**：条件表达式限制为参数变量的简单比较，保证 GUI 可解析：

```apy
gui button(label, is_danger=False):
    apply danger_style    if is_danger
    apply default_style   unless is_danger
```

#### 第三层：`style`（全局具名样式）

借鉴 Ren'Py `style` 系统的自动推导能力。`style` 与 `mixin` 的职责严格分离：

- `style` 只用于自动推导，按命名约定自动绑定到对应控件类，**不支持手动 `apply`**
- `mixin` 只用于手动 `apply`，不参与自动推导
- 规则一句话：**想自动生效用 `style`，想手动控制用 `mixin`，两者不互换**
- `style=` 具名参数（调用点覆盖）只接受 `mixin`，不接受 `style`

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

`window` 在 Qt 控件之上包一层薄抽象，屏蔽 Qt 概念，同时保留 Qt 逃逸出口。核心原则：**高频场景用声明式语法，低频复杂需求用 `qt:` 逃逸块。**

开发者不需要了解 Qt 的布局管理器或信号槽机制即可完成大部分 UI 工作；确实需要原生 Qt 能力时，`qt:` 块提供完整访问权限。

---

#### 声明语法与组件继承

`window` 支持函数签名风格参数，与 `gui` 保持一致。通过 `template` / `extends` 进行组件继承：

```apy
# ui/base.apy
template BaseBox:
    background "ui/box_default.png"
    font "fonts/default.ttf"
    padding (20, 10)
    text_color #ffffff

# ui/autumn_box.apy
import "ui/base.apy"

template autumnBox extends BaseBox:
    background "ui/box_autumn.png"
    name_color #ff8800
    font "fonts/handwriting.ttf"
```

带参数的 `window` 定义：

```apy
window DialogueBox(char_name, theme="default"):
    background "ui/box_{theme}.png"
    padding (20, 12)
    slot name_label
    slot dialogue_text
```

---

#### 布局能力

`window` 复用与 `gui` 相同的布局关键字，引擎内部翻译为对应的 Qt layout manager：

| `.apy` 布局 | Qt 对应 |
|------------|---------|
| `vstack` | `QVBoxLayout` |
| `hstack` | `QHBoxLayout` |
| `grid` | `QGridLayout` |
| `pin` | `QStackedLayout` + 绝对定位 |
| `grow` | `setSizePolicy(Expanding)` |
| `split` | `QSplitter` |
| `scroll` | `QScrollArea` |

```apy
window InventoryPanel:
    size (400, 600)
    vstack gap=8:
        text "背包" (font_size=20)
        scroll vertical:
            grid columns=4 gap=4:
                slot items
        hstack:
            spacer grow
            button "关闭" on_click: Return()
```

布局语义与 Pygame 后端完全一致，同一套规则无需记两套。

---

#### 条件样式

`window` 支持与 `gui` 相同的六态条件样式（`hovered`、`pressed`、`active`、`selected`、`focused`、`disabled`）和遮罩层（`overlay`、`alpha`），引擎内部通过 Qt 样式表（QSS）或属性动画实现：

```apy
window OptionButton(label):
    style:
        background #444444
        hovered:  background #555555
        pressed:  background #3a3a3a
        selected: background #226622
        disabled:
            overlay #00000055
            alpha 0.5
    text label (anchor=center)
```

Qt 后端不支持任意 `border_radius` 动画（QSS 限制），复杂圆角动画退回 `qt:` 逃逸块。

---

#### `qt:` 逃逸块

当声明式语法表达不了所需的原生 Qt 能力时，使用 `qt:` 逃逸块，块内为完整的 Python/Qt 代码：

```apy
window VideoPlayer:
    size (800, 450)
    qt:
        python:
            from PySide6.QtMultimediaWidgets import QVideoWidget
            from PySide6.QtMultimedia import QMediaPlayer, QAudioOutput

            player = QMediaPlayer()
            audio  = QAudioOutput()
            widget = QVideoWidget()

            player.setAudioOutput(audio)
            player.setVideoOutput(widget)
            layout.addWidget(widget)

            # 暴露给引擎的接口：play / pause / stop
            self._player = player

    slot controls
```

`qt:` 块内可访问：
- `layout`：当前控件的 Qt layout 实例（`QLayout`）
- `widget`：当前控件的 Qt widget 实例（`QWidget`）
- `self`：当前 `window` 控件的包装对象，可挂自定义属性供其他方法访问

`qt:` 块作为代码节点处理，Axn-Editor 不解析内部结构，但归属关系在编辑器中保留。

---

#### 插槽系统

`window` 同样支持具名 `slot`，调用方填充内容：

```apy
window BaseDialog(title):
    background "ui/panel.png"
    padding (20, 20)
    vstack gap=12:
        text title (font_size=18)
        slot body
        slot footer:
            default:
                button "确认" on_click: Return()

# 调用
use BaseDialog(title="设置"):
    slot body:
        toggle "全屏" bind=preferences.fullscreen
        slider bind=preferences.music_volume min=0.0 max=1.0
```

---

#### `window` 与 `gui` 的对比

| | `gui`（Pygame） | `window`（Qt） |
|---|---|---|
| 后端 | Pygame | Qt / PySide6 |
| 参数化 | ✅ 函数签名风格 | ✅ 函数签名风格 |
| 布局关键字 | ✅ 完整支持 | ✅ 复用同一套 |
| 条件样式 | ✅ 六态 + overlay/alpha | ✅ 同上（QSS 实现） |
| 继承 | `extends`（属性+参数） | `extends`（属性覆盖） |
| 逃逸 | `canvas python:` | `qt: python:` |
| 实例化 | 直接调用 `health_bar(80, 100)` | 直接调用或引用路径 `"ui/x.apy::X"` |
| 插槽 | ✅ 具名 slot | ✅ 具名 slot |
| Round-Trip | ✅ 完整解析 | ✅（`qt:` 块降级代码节点） |

`window` 不支持 `screen` 的 `call screen` / `show screen` 语义——Qt 后端的页面切换通过 `QStackedWidget` 或 `window show` / `window hide` 指令控制，不引入新的执行流阻塞机制。需要阻塞等待用户选择时，使用 `modal show`（`modal` 是引擎层语义，两个后端均支持）。

---

### 关键设计决策（UI 系统）

**`Return()` 保留，`Action()` 不引入**：`Return()` 是跨执行层的信号，是 `call screen` 和 `modal show` 的唯一传值机制，必须存在。引擎不引入 `Action()` 包装器——Ren'Py 需要它是因为其 screen 语言与 Python 割裂，Axn-Plus 的 `gui` 块内可以直接写 `on_click: $ do_something()`，不需要桥接层。`Return()` 无参数调用时 result 为 `None`。

**三种 `as` 接返回值的传值机制不同**：`.apy` 里有三处 `as` 接返回值，传值方式各不相同，不强行统一。在错误的上下文使用错误的传值方式（如在 label 内写 `Return()`，或在 screen 内写 `return expr`）解析期报错，不静默失败：

| 场景 | 写法 | 传值机制 |
|------|------|---------|
| `call label()` | `call morning_scene() as result` | label 内 `return expr` 关键字 |
| `call screen` / `modal show` | `call screen confirm_dialog as result` | screen 内 `Return(expr)` 函数 |
| `menu as` | `menu as answer:` | 选项内 `-> expr` 表达式 |

**内置控件不分 `textbutton` / `imagebutton`**：内容类型不混入控件类型。`button` 只负责交互语义，内容通过第一个位置参数（字符串 `label`）或 `slot children` 填充。`label` 只接受字符串字面量，图片必须走 `slot children`，规则唯一无歧义。引擎标准库提供 `text_button` / `image_button` 作为便利封装，但它们是复合控件，不是独立控件类型。

**`button` 不预设视觉**：`button` 本身不携带任何默认背景或颜色。样式完全由 `style button` 全局推导或调用点 `style=` 参数决定。不定义 `style button` 时控件自然是素的，无需任何 flag。

**`style=` 只接受 `mixin`**：调用点样式覆盖只接受 `mixin`，不接受 `style`。`style` 只用于自动推导，不支持手动 `apply`；`mixin` 只用于手动 `apply`，不参与自动推导。两者职责严格分离，边界明确，规则一句话可以说清楚。

**`typewriter` 与对话文本共享 `TextRenderer`**：脚本层对话行和 UI 层 `typewriter` 控件底层共享同一套 `TextRenderer` 核心模块，两个入口行为完全一致。富文本标签、语音同步、`nowait` 等特性在两个入口同步生效，不维护两套实现。

**`dialog` vs `modal`**：脚本层 `modal` 动词负责执行流控制（阻塞、焦点接管、返回值写入）；UI 层视觉容器控件命名为 `dialog`，只负责视觉呈现和布局。两者职责不重叠，命名不冲突。`modal show "x.apy::MyDialog"` 调用的是 `gui MyDialog` 内含 `dialog` 容器的控件定义。

**`grid` 内置**：`grid` 是背包、图鉴、CG 画廊等高频场景的必需控件，不依赖 `vstack` + `hstack` 嵌套模拟。支持 `columns`、`gap`、`row_gap` 参数，与 `scroll` 配合处理超出场景。动态列表内控件必须显式指定 `key=`，规则与其他动态列表一致。

**`dropdown` / `radio_group` / `checkbox_group` 内置**：Ren'Py 原生缺失这些控件，需要手搓。Axn-Plus 直接内置，通过 `bind=` 绑定 `store` 变量，`options=` 接受静态列表或 `store` 变量。

**后端绑定**：后端在项目初始化时选定，之后固定。Pygame 项目使用 `screen` + `gui`，Qt 项目使用 `window`，不混用。

**`window` 抽象层原则**：`window` 是薄抽象，不是完整的跨后端 UI 框架。覆盖高频场景（布局、基础控件、条件样式、插槽），低频复杂需求通过 `qt:` 逃逸块访问原生 Qt API。`qt:` 块内代码不受引擎约束，开发者自行负责线程安全（不得在 `qt:` 块内直接访问 `store`，须通过 `UICommand` 机制）。

**`window` 布局复用同一套关键字**：`vstack`、`hstack`、`grid`、`pin`、`grow`、`split`、`scroll` 在 `window` 内语义与 `gui` 完全一致，引擎内部分别翻译为对应的 Qt layout manager，开发者不需要记两套规则。

**`window` 参数化支持**：`window` 定义支持函数签名风格参数（与 `gui` 一致），`template` 关键字可省略直接用 `window 名称(参数):`，两者等价。`extends` 只继承属性，不继承参数列表，子类需独立声明参数。

**六态条件样式**：预定义状态扩展为六个——`hovered`、`pressed`、`active`、`selected`、`focused`、`disabled`。`pressed` 是鼠标按下的瞬时态；`active` 是持续激活态（按住不松、长按类操作）；`selected` 是开发者显式维护的静态选中态，与鼠标操作无关。三者语义不重叠，不允许混用。

**`overlay` 与 `alpha` 遮罩**：`overlay` 在控件表面叠加半透明色，子元素不受影响；`alpha` 降低整个控件（含子元素）的透明度。两者均可用于任意状态，可同时使用（`overlay` 先合成，`alpha` 作用于合成结果）。`overlay` 接受 `#rrggbbaa` 格式，`alpha` 接受 `0.0`–`1.0` 浮点数，均支持 `$token` 引用。

**`screen` 与 `gui` 的职责**：`screen` 负责布局——把控件组合成完整的 UI 画面；`gui` 负责控件的封装与复用。`use` 只用于 screen 嵌套 screen；`gui` 控件用直接调用语法。有插槽填充时必须写 `use`，无插槽填充时 `use` 可省略。

**持久性跟层走**：`gui` 控件放在哪一层，就遵守那一层的持久性规则。`clear` 不清除 `ui` 层（持久），但会清除 `effect` / `sprite` 层上的 `gui` 控件。需要持久但不在 `ui` 层时，显式声明 `persistent`。

**Python 逃逸的 surface 作用域**：`gui` 块内 Python 块拿到的是控件局部 surface，坐标从 `(0, 0)` 开始，引擎负责合成到正确位置。保证控件封装性，不暴露全局 surface。

**布局关键字语义化**：关键字描述意图（`pin`、`stack`、`grow`、`split`、`vstack`、`hstack`），不描述实现，不照搬前端术语。绝对定位（Ren'Py 风格）永远可用作退路。

**插槽系统**：`screen` 和 `gui` 均支持具名 `slot` 声明。未填充的 `slot` 直接不渲染，不报错。`slot` 支持 `default:` 块声明默认内容。禁止 `slot` 嵌套透传，需要多层组合时用 `extends` 或拆成独立 screen。`slot children` 作为匿名插槽语法糖保留，多插槽场景用具名 `slot`。

**简写语法**：`style` 块集中声明样式（`when` 可省略，状态名直接作为缩进键）；单行事件处理（`on_click` 只有一个表达式时可内联）；属性内联括号（叶子控件用具名参数内联）；`vstack` / `hstack` 是 `stack vertical` / `stack horizontal` 的别名。完整写法始终有效，简写是语法糖。

**样式系统四层优先级**：`自身声明 > 手动 apply mixin（后 > 前）> 自动推导 style > extends 父类自身声明 > extends 父类 apply > theme token 默认值`。`style` 只参与自动推导（按 `控件名`、`控件名_状态`、`控件名_子元素`、`控件名_子元素_状态` 命名约定），不支持手动 `apply`；`mixin` 只用于手动 `apply`，不参与自动推导。自动推导只拾取自身名字匹配的 `style`，不沿 `extends` 继承链向上查找。`mixin` 支持参数化（函数签名风格）。冲突时规则唯一，引擎启动时对已知冲突输出警告。

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

**事件名命名空间**：`axn:` 前缀为引擎标准库保留，开发者不应使用。标准库内部事件（如 `axn:gallery_unlocked`、`axn:save_completed`）使用此前缀，避免与项目事件冲突。开发者自定义事件直接用不带前缀的名称。使用 `axn:` 前缀时引擎启动输出警告：

```
AxnWarning: [ui] Event name 'axn:my_event' uses the reserved 'axn:' prefix.
  This prefix is reserved for engine standard library events.
  Rename your event to avoid potential conflicts with future engine updates.
  → ui/my_panel.apy, line 12
```

**`theme` token 类型系统**：token 类型由字面量决定（数字 → `int`/`float`，`#rrggbb` → `color`，字符串 → `str`），同类型支持算术运算，类型不匹配时引擎启动报错：

```apy
font_size $font.size.base + 2       # int + int，合法
padding   $spacing.md               # int，合法
color     $color.primary            # color 类型，合法
# color $color.primary + 2         # 类型不匹配，AxnThemeError
```

```
AxnThemeError: Type mismatch in token expression
  '$color.primary + 2' — cannot add color and int
  → options_window.apy, line 34
```

**`style` 局部作用域**：`screen` 内部可定义局部 `style`，只在该 `screen` 内生效，优先级高于全局同名 `style`，不污染其他 `screen`。`mixin` 同样支持 `extends` 继承，子 `mixin` 覆盖父 `mixin` 同名属性，未声明属性全部继承。`apply` 支持条件应用（`apply mixin if condition`），条件表达式限制为参数变量的简单比较，保证 GUI 可解析。

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
| `style`（局部，screen 内） | screen 节点内的局部样式面板，标注"局部，不影响全局" |
| `mixin extends` | 样式库面板显示 mixin 继承关系；继承字段以灰色标注来源 |
| `apply mixin if condition` | apply 节点显示条件字段，条件参数变量名可编辑 |
| `when ... and/or/not ...` 组合条件 | 样式编辑器内组合条件字段，支持 and/or/not 可视化编辑 |
| `window show/hide/auto` | 脚本区对话框控制节点；`hide (all)` 节点显示 mode 下拉选择 |
| `active` 状态 | 样式编辑器内与 `pressed`、`selected` 并列；标注"持续激活态，区别于 pressed（瞬时）和 selected（静态）" |
| `overlay` 遮罩 | 样式编辑器内颜色选择器，带 alpha 通道滑块；与 `alpha` 字段并排展示，标注"叠加在表面，子元素不受影响" |
| `alpha` 透明度 | 样式编辑器内 0.0–1.0 滑块；标注"影响整个控件含子元素" |
| `window` 定义 | 编辑器 Qt 后端专用控件库面板；参数列表完整可解析；`qt:` 逃逸块降级为代码节点，归属关系保留 |
| `window extends` | 控件继承树，继承字段以灰色标注来源；子类参数列表独立展示 |
| `window` 布局关键字 | 与 `gui` 共用同一套布局编辑器；编辑器标注"Qt 后端：翻译为对应 Qt layout manager" |
| `qt:` 逃逸块 | 代码节点，`layout` / `widget` / `self` 变量在节点属性面板中列出并标注用途 |

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

**扩展管理**

引擎内置扩展和外部扩展（`main/axn/`）默认全部关闭，必须在 `options_window.apy` 中显式声明开启：

```apy
engine:
    extensions:
        # 引擎内置扩展，逐个声明，默认全 false
        builtins:
            gallery          = true
            panning_sprite   = true
            cursor_manager   = true
            scramble_text    = false
            downloader       = false
            archive          = false
            notify_ui        = true     # 游戏内通知 UI 模板（核心 notify 指令需要此 UI）
            error_screen     = false

        # 外部扩展（main/axn/ 目录），整体开关
        external = true

        # 黑名单：external = true 时排除指定模块（模块名为文件名去掉 .apy）
        disable:
            axn::map
            axn::clue_board

    autosave:
        enabled          = true
        trigger          = "checkpoint"   # checkpoint：只在 checkpoint 指令处自动存（推荐）
                                          # on_dialogue：每条对话后存（性能较差）
        slot             = "autosave"     # 独立存档槽，不覆盖手动存档

    screenshot:
        enabled          = true           # 开发模式默认开启
        key              = "f12"          # 快捷键，Steam 风格
        path             = "screenshots/" # 保存目录，相对项目根目录
        include_ui       = false          # true 时截图包含 UI 层
```

规则说明：
- `builtins` 下每项独立控制，未声明的项默认 `false`
- `external = true` 时引擎扫描整个 `main/axn/` 目录并加载所有 `.apy` 文件
- `disable` 黑名单只在 `external = true` 时生效，对 `builtins` 无效
- `external = false`（默认）时 `disable` 块无意义，引擎启动时输出警告
- 引擎启动时检查 `builtins` 下声明了但实际未安装的模块，报 `AxnExtensionError` 并列出缺失项

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

**`engine.open_file()`**：在 Python 块中读取项目内任意文件时，使用 `engine.open_file()` 而非直接 `open()`。引擎透明处理文件系统路径和归档（`.npa`）内路径的差异，保证打包后行为一致：

```python
# python: 块内
import json
with engine.open_file("data/config.json") as f:
    data = json.load(f)

with engine.open_file("data/dialogue_extra.txt") as f:
    lines = f.read().splitlines()
```

直接使用 `open()` 在开发期有效，但打包后资源被归档时会找不到文件。`engine.open_file()` 同时支持两种场景，推荐始终使用。返回标准 `io.IOBase` 对象，接口与内置 `open()` 一致（只读模式）。

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
show autumn        # 符号表：define char → 立绘
show hud           # 符号表：gui 定义 → UI 控件
show MyEffect()    # 符号表：自定义可显示类 → 自定义对象
```

自定义可显示类（继承 `AnimatedSprite` 的 Python 类）必须在 `.apy` 文件中显式声明才能进入符号表：

```apy
import MyEffect from "effects/my_effect.py"
import LivePortrait from "characters/live_portrait.py"
```

符号表里同一名字对应多种类型时，引擎启动时报错，要求重命名消歧义。

### 平台变体检测（Platform Variants）

引擎提供 `engine.variant()` 接口，检测当前运行平台特征，用于脚本层和 UI 层做条件适配。

**内置变体名：**

| 变体 | 为 `true` 的条件 |
|------|----------------|
| `"pc"` | Windows / macOS / Linux |
| `"mobile"` | Android |
| `"touch"` | 触摸屏可用（Android，或 PC 触摸屏） |
| `"small"` | 屏幕宽度 < 960px（竖屏手机） |
| `"wide"` | 屏幕宽高比 ≥ 16:9 |
| `"windows"` | Windows |
| `"macos"` | macOS |
| `"linux"` | Linux |
| `"android"` | Android |

**脚本层使用：**

```apy
if engine.variant("mobile"):
    $ preferences.text_speed = 1.2    # 移动端默认稍快

if engine.variant("small"):
    show gui mobile_hud
else:
    show gui pc_hud
```

**UI 层使用：**

```apy
screen main_menu:
    if engine.variant("touch"):
        use touch_menu_layout
    else:
        use pc_menu_layout
```

**`options_window.apy` 自定义变体：**

```apy
engine:
    variants:
        handheld = engine.variant("mobile") or (engine.variant("touch") and engine.variant("small"))
```

```apy
if engine.variant("handheld"):
    show gui handheld_controls
```

**发布包中变体值在打包时固化为常量**（`axn build --platform android` 时），编译器将已知为 `false` 的分支剥离，减少包体积。

---

### 定制按键映射（Custom Keymap）

引擎内置行为的按键绑定可以在 `options_window.apy` 中完整重映射，覆盖默认值。

**内置行为的默认按键：**

| 行为 | 默认键 |
|------|--------|
| `advance` | 左键单击、`Space`、`Enter` |
| `rollback` | 鼠标右键、`PageUp` |
| `skip` | `Ctrl`（按住） |
| `auto` | `a` |
| `hide_window` | `h` |
| `screenshot` | `F12` |
| `pause_menu` | `Escape` |
| `history` | 鼠标中键、`PageDown` |
| `self_voicing_toggle` | `v` |
| `director` | `Shift+d`（仅开发模式） |
| `devtools` | `` Shift+` ``（仅开发模式） |

**重映射语法：**

```apy
# options_window.apy
engine:
    keymap:
        advance:       ["mouseup_1", "space", "return", "KP_ENTER"]
        rollback:      ["mouseup_3", "pageup", "backspace"]
        skip:          ["ctrl"]
        auto:          ["a"]
        hide_window:   ["h", "mouseup_2"]
        screenshot:    ["f12"]
        pause_menu:    ["escape"]
        history:       ["mouseup_2", "pagedown"]
```

键名格式遵循与 `on key` 相同的规范（修饰键小写，顺序固定 `ctrl→shift→alt→key`），鼠标按键用 `mouseup_N` / `mousedown_N`（N 为按键编号，1=左键，2=中键，3=右键）。

**手柄映射：**

```apy
engine:
    keymap:
        gamepad:
            advance:      ["gamepad_a", "gamepad_cross"]
            rollback:     ["gamepad_b", "gamepad_circle"]
            skip:         ["gamepad_lt"]
            auto:         ["gamepad_rt"]
            pause_menu:   ["gamepad_start"]
            history:      ["gamepad_select"]
```

手柄键名格式：`gamepad_` 前缀 + Xbox 命名（`a`/`b`/`x`/`y`/`lt`/`rt`/`lb`/`rb`/`start`/`select`/`up`/`down`/`left`/`right`）。PS 命名作为别名支持（`cross`=`a`，`circle`=`b`，`square`=`x`，`triangle`=`y`）。

**Steam Input API**：开启 Steamworks 时，手柄输入优先走 Steam Input，`gamepad_*` 映射自动转换为 Steam Input Action 名称，支持玩家在 Steam 大屏幕模式中自定义手柄映射。

**按键冲突检测**：引擎启动时检测同一行为集合内是否有重复按键绑定，发现时输出警告（不阻止运行）：

```
AxnWarning: [keymap] Key 'mouseup_3' is bound to both 'rollback' and 'history'.
  Last binding wins: 'history'.
  → options_window.apy, keymap section
```

---

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
      loader.py              # 资源加载、缓存；engine.open_file() 透明路径接口实现
      audio.py               # play / pause / stop / 多通道管理
      video.py
   apy/
      __init__.py
      stdlib.py              # 引擎内置指令实现
      transition.py          # 内置过渡效果
      transform.py           # keyframe 动画系统
      saveable.py            # @saveable / Saveable 基类
      color_matrix.py        # ColorMatrix 基类及内置矩阵
      nvl.py                 # NVL 模式渲染器
      bubble.py              # Speech Bubble 渲染器
   core/
      preloader.py           # 图片预加载系统
      preferences.py         # Preferences 单例
      achievement.py         # 成就系统
      read_tracker.py        # 已读对话行追踪（persistent）
      variant.py             # 平台变体检测
   platform/
      __init__.py
      steam.py               # Steamworks API 封装
      sdl2_native.py         # SDL2 原生扩展（手柄震动、传感器、剪贴板等）
      tts.py                 # TTS / Self-Voicing 后端抽象
      tts_windows.py         # SAPI 5
      tts_macos.py           # AVSpeechSynthesizer
      tts_linux.py           # espeak-ng
      tts_android.py         # Android TextToSpeech (Pyjnius)
   audio/
      __init__.py
      filters.py             # DSP 滤波器链
   cli/
      __init__.py
      init.py                # axn init，生成项目骨架
      build.py               # axn build，打包发布
      run.py                 # axn run，开发期启动
      test.py                # axn test，自动化测试 runner
      extract_strings.py     # axn extract-strings，翻译字符串提取
      check_strings.py       # axn check-strings，翻译完整性检查
```

`parser/` 独立，供 Axn-Editor 的 LSP 插件直接复用，不与运行时耦合。LSP 插件复用 `parser/incremental_parser.py`，不使用全量三遍扫描器，保证实时补全响应速度。两层实现共享 `lexer.py` 和 `ast_nodes.py`。`core/` 中无任何 pygame 或 Qt import，后端通过 `backends/base.py` 的抽象接口交互。`cli/` 提供 `axn init` / `axn run` / `axn build` / `axn test` / `axn extract-strings` / `axn check-strings` 子命令。

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

四个内置通道名为保留关键字，不允许自定义通道使用相同名称，引擎启动时检查并报错：

```
AxnParseError: Channel name 'music' is reserved by the engine.
  Built-in channels: music, sound, voice, ambient.
  Choose a different name for your custom channel.
  → options_window.apy, line 12
```

`play music` 等价于 `play audio (channel="music")`，内置通道是语法糖。

### 通道 UI 可见性配置

在 `options_window.apy` 中声明每个通道在设置界面的显示与交互行为：

```apy
# options_window.apy
engine:
    audio:
        channels:
            music:   ui=true      # 显示滑条，玩家可拖动（内置通道默认值）
            sound:   ui=true
            voice:   ui=true
            ambient: ui=locked    # 显示滑条但灰掉，不可拖动，只能脚本控制
            # 整行不写 = 通道不在设置界面显示（但仍然可以用 play 指令播放）
```

三种状态：

| `ui` 值 | 行为 |
|---------|------|
| `true` | 显示滑条，玩家可拖动调整 |
| `locked` | 显示滑条但控件灰掉，玩家可见当前值但无法操作，只能通过脚本控制 |
| 整行不写 | 通道不在设置界面出现，开发者通过 `$ preferences.xxx_volume` 或 `play` 指令控制 |

内置四个通道默认 `ui=true`，不写配置时行为不变。自定义通道默认不显示，需要显式声明才出现在设置界面。

`ui=locked` 适用场景：环境音量由剧情脚本控制（如梦境场景强制调低）、临时屏蔽某通道的玩家调整权限。

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
│   ├── AxnConstError    # 尝试修改 const 声明的常量
│   ├── AxnValueError    # flag validate 校验失败
│   ├── AxnInterpolationError  # 对话插值表达式求值失败
│   └── AxnSaveError     # 存档序列化失败
├── AxnAssetError        # 资源加载失败
│   └── AxnVoiceError    # voice 短路径推断失败
├── AxnExtensionError    # 扩展模块注册/加载失败
├── AxnThemeError        # theme token 类型不匹配（引擎启动时）
└── AxnInternalError     # 引擎自身 bug，直接暴露 Python traceback
```

**各错误类的触发时机：**

| 错误类 | 触发时机 | 示例场景 |
|--------|---------|---------|
| `AxnParseError` | Lexer/Parser 阶段 | 语法错误、循环继承、label 冲突 |
| `AxnCompileError` | AST → 字节码阶段 | 跳转目标编译期验证失败、成就 id 重复 |
| `AxnNameError` | VM 执行时 | 访问未在 `flag` 声明的变量（严格模式）|
| `AxnTypeError` | VM 执行时（debug 模式） | `flag: day: int` 被赋值 `str` |
| `AxnJumpError` | VM 执行时 | `jump` / `call` 动态目标在运行时不存在 |
| `AxnConstError` | VM 执行时 | `$ MAX_HP = 200`（MAX_HP 是 const）|
| `AxnValueError` | VM 执行时 | `flag validate` 校验失败 |
| `AxnInterpolationError` | VM 执行时 | 对话插值含函数调用 |
| `AxnSaveError` | 存档写入时 | 未声明可序列化的自定义类 |
| `AxnAssetError` | 资源加载时 | 图片文件不存在 |
| `AxnVoiceError` | 语音解析时 | voice 短路径找不到对应文件 |
| `AxnExtensionError` | 引擎启动时 | 扩展模块未安装、注册 verb 与内置冲突 |
| `AxnThemeError` | 引擎启动时 | theme token 类型运算不匹配 |
| `AxnInternalError` | 任意时机 | 引擎代码自身的 bug |

#### 错误信息格式

**脚本作者看到的（AxnParseError / AxnRuntimeError 等）：**

```
AxnParseError: Duplicate label 'morning_scene'
  → scene.apy, line 5
  → chapter2.apy, line 103

  3 |
  4 |
  5 | label morning_scene:
      ^

Hint: Labels are globally visible. Rename one to resolve the conflict.
```

**警告示例（AxnWarning）：**

```
AxnWarning: [parser] '$' line contains multi-line expression.
  Bracket continuation is allowed but not recommended.
  Behavior may be undefined in some contexts.
  → scene.apy, line 12

  10 |
  11 | autumn: "你好。"
  12 | $ x = (
            ^
  13 |     1 + 2

Hint: Use a 'python:' block for multi-line expressions.
  Add 'ignore_multiline_dollar = true' in options_window.apy to suppress this warning.
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
- 多位置错误（label 冲突、循环引用）：所有位置均列出

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
  autumn_adult → autumn_teen → autumn_adult

  → characters.apy, line 8   (define char autumn_adult extends autumn_teen)
  → characters.apy, line 15  (define char autumn_teen extends autumn_adult)
```

**警告格式：**

不中断执行，格式为 `AxnWarning: [模块名] 描述 → 位置`：

```
AxnWarning: [scheduler] 'wait for all' has no finite transforms to wait for.
  All transforms are 'repeat forever'. Did you mean to use 'wait'?
  → autumn_enter animation block, scene.apy, line 3
```

#### Hint 策略

不是每个错误都有 Hint，以下情况必须给：

| 错误场景 | Hint 内容 |
|----------|-----------|
| `$` 行括号续行 | 改用 `python:` 块 |
| `with store` 内出现下标访问 | 先在 `python:` 块算好再赋值 |
| `menu as` 内出现 `jump` | `menu as` 不允许跳转，改用 `menu` |
| `say` 传入静态角色名 | 改用 `角色:` 语法 |
| `define char narrator` | `narrator` 是保留关键字 |
| label 命名冲突 | 列出所有冲突位置 |
| 循环 `include` | 打印完整引用链 |
| 循环继承 | 打印继承链 |
| `AxnVoiceError` 短路径 | 列出已搜索路径，建议完整路径或声明 `voice_ext` |
| `AxnInterpolationError` | 提示先用 `$` 块预计算 |
| `AxnConstError` | 说明是 const，建议改用普通变量 |
| `then` 后接 `repeat forever` | 说明 `then` 要求有限动画 |
| `animation` 块内含 Python 块 | 建议改用 `label` + Python 块处理 |
| `sta label` 内含 `$` 块 | 说明 `sta` 不允许动态代码 |
| `parallel` 内出现 `checkpoint` | 建议移到 parallel 块之后 |
| `call ... as result if ...` | 条件不满足时返回值语义不明，退回 `if` 块 |
| `show` 位置参数含纯数字 | 缺失位置关键字，建议 `(duration=N)` 具名写法 |

#### 错误处理矩阵

```
错误类型                开发模式（引擎运行）          发布包
──────────────────────────────────────────────────────────────────
AxnParseError          终端完整报错 + 停止           编译阶段拦截，不进包
AxnCompileError        终端完整报错 + 停止           编译阶段拦截，不进包
AxnWarning             终端显示 + 继续运行           完全静默
AxnTypeError           终端完整报错 + 继续           完全静默（debug 专属）
assert                 执行 + 报错                   剥离
AxnNameError           终端完整报错 + 停止           错误界面 + crash.log
AxnJumpError           终端完整报错 + 停止           错误界面 + crash.log
AxnConstError          终端完整报错 + 停止           错误界面 + crash.log
AxnValueError          终端完整报错 + 继续           错误界面 + crash.log
AxnInterpolationError  终端完整报错 + 继续           错误界面（降级显示原始文本）+ crash.log
AxnSaveError           终端完整报错                  错误界面 + crash.log
AxnExtensionError      终端完整报错 + 停止           编译阶段拦截，不进包
AxnInternalError       完整 Python traceback         错误界面 + crash.log
AxnAssetError          终端完整报错 + 停止           静默 + crash.log
  └── 图片/立绘        同上                          不渲染 + crash.log
  └── 音视频           同上                          跳过 + crash.log
  └── AxnVoiceError    同上                          静默无声 + crash.log
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

#### 开发者工具

开发模式下（`axn run`）内置调试面板，Shift+\` 呼出，不依赖 Axn-Editor：

**控制台**：可执行任意 Python 表达式，结果直接打印到面板，等价于在当前 store 上下文里执行。

**变量浏览器**：树状展示 `store` 和 `persistent` 的所有变量，支持实时编辑基础类型值。

**Label 跳转**：输入 label 名，引擎立即跳转到该 label 开始执行，call 栈丢弃（与读档行为一致）。

**截图快捷键**：F12 保存当前画面到 `screenshots/` 目录（可在 `options_window.apy` 配置）。

发布包（`axn build`）中开发者工具完全剥离，不包含在包体内。后续集成进 Axn-Editor 作为调试面板，但命令行工具版本始终保留。

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
- 第一次遇到缩进时记录该文件的缩进单位，后续不一致时报错：

```
AxnParseError: Inconsistent indentation
  This file uses 3-space indentation (detected at line 2).
  Found 4-space indentation at line 8.
  → scene.apy, line 8, col 1
Hint: Use the same indentation unit throughout the file.
```
- 不限制缩进数量（2、3、4、5 均可），但同一文件必须统一
- Tab 与空格不允许在同一文件内混用
- Axn-Editor 默认使用 3 空格缩进

**字符串引号：**
- 只允许双引号 `"`，使用单引号时报错：

```
AxnParseError: Single quotes are not allowed in .apy strings.
  Use double quotes instead: "text" rather than 'text'.
  → scene.apy, line 8
```

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
    WHILE           = auto()
    FOR             = auto()    # for 循环关键字（区别于 wait for 中 for）
    BREAK           = auto()
    CONTINUE        = auto()
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
    NVL             = auto()
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
    SIGNAL          = auto()
    ON              = auto()
    FLAG            = auto()
    CONST           = auto()
    SET             = auto()
    CHECKPOINT      = auto()
    ASSERT          = auto()
    MODAL           = auto()
    INPUT           = auto()
    NOTIFY          = auto()
    EXPOINT         = auto()
    AFTER           = auto()    # after N: 延迟执行块
    DEFER           = auto()    # defer: label 退出时执行
    ONCE            = auto()    # once per_session/per_playthrough/ever
    UNWIND          = auto()    # unwind 展开调用栈
    EXIT            = auto()    # exit 退出游戏
    SNAPSHOT        = auto()    # snapshot/restore 手动状态快照
    RESTORE         = auto()
    REPEAT          = auto()    # repeat N: 固定次数重复块
    PYTHON          = auto()    # python: 块关键字
    NARRATOR        = auto()    # 保留关键字
    STA             = auto()
    DYN             = auto()
    AS              = auto()
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

缓存文件存放在项目的 `.cache/` 目录，构建发布包时不打入包内。

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

**`CompiledLabel.name` 是必填字段**：热重载时通过名字匹配正在执行的 CallFrame，找到后替换引用并重置 PC。匿名 parallel track 的命名规则为 `__parallel_{parent_label}_L{line}_{track_index}__`（含源码行号确保同一 label 内多个 parallel 块不冲突），有命名的 track 用命名作为后缀，保证名字全局唯一。

**`CompiledProject.label_index` 设计为可变 dict**：热重载就是在运行时替换这个索引里的 `CompiledLabel` 对象，不能是 frozen 结构。

#### 指令集

字节码指令分以下几类：

| 类别 | 指令举例 |
|------|---------|
| 控制流 | `JUMP` `JUMP_IF` `JUMP_IF_NOT` `CALL` `RETURN` `HALT` |
| Python 块 | `EXEC_PYTHON` `PUSH_EXPR` |
| 对话与旁白 | `DIALOGUE` `NARRATOR` `WAIT_CLICK` `NVL_APPEND` `NVL_CLEAR` `NVL_HIDE` |
| 显示控制 | `SHOW` `HIDE` `SCENE` `CLEAR` `EXPRESSION_CMD` `COLOR_MATRIX` `LAYER_COLOR_MATRIX` `LAYER_TRANSFORM` |
| 音视频 | `PLAY_AUDIO` `STOP_AUDIO` `PAUSE_AUDIO` `RESUME_AUDIO` `PLAY_VIDEO` `STOP_VIDEO` `AUDIO_FILTER` |
| 镜头 | `CAMERA` |
| 等待 | `WAIT_DURATION` `WAIT_FOR` |
| 菜单 | `MENU` `CHOICE` |
| 存档 | `CHECKPOINT` |
| 层管理 | `LAYER_MANAGE` |
| 输入控制 | `INPUT_DISABLE` `INPUT_ENABLE` |
| 模态框 | `MODAL_SHOW` `MODAL_HIDE` |
| 并行 | `PARALLEL_BEGIN` `PARALLEL_END` |
| 存取 | `LOAD_CONST` `STORE_VAR` `LOAD_VAR` `WITH_STORE` `PREF_SET` |
| 动画 | `CALL_ANIMATION` |
| 模式控制 | `AUTO_SET` `SKIP_SET` |
| 成就 | `UNLOCK_ACHIEVEMENT` `UNLOCK_SCENE` `UNLOCK_MUSIC` |
| 资源 | `PRELOAD` |
| 调试 | `ASSERT`（release 模式不生成） `DEBUG_BREAK` |

每条指令携带：opcode、operand（常量池索引或直接值）、源码行号、源码文件名。行号和文件名用于运行时错误定位。

#### 常量池

所有复杂数据（字符串、命令对象、Python code object）统一放常量池，指令只存索引。可哈希对象自动去重，不可哈希对象（code object）直接追加。

#### 关键设计决策（编译器）

**跨文件跳转 operand 格式用 `(文件名, label名)` 元组**：视觉小说项目规模不会有性能瓶颈；调试时 operand 直接可读；展平方案在增量重编译时需要重新计算所有地址，维护成本高。VM 运行时通过全局 label 索引定位目标，天然支持跨文件跳转。

**跳转地址延迟回填**：编译单个文件时跨文件 label 地址尚不完整，所有文件编译完后统一验证回填。跳转目标不存在时在此阶段报 `AxnCompileError`，不等到运行时。

**Python 块编译为 code object**：`compile()` 在编译阶段执行，运行时只做 `exec()`。语法错误在编译阶段暴露，文件名和行号偏移传入 `compile()`，保证 traceback 正确指向 `.apy` 源文件。

**parallel 块编译为多个独立 CompiledLabel**：每个 track 生成一个匿名 `CompiledLabel`，命名规则为 `__parallel_{parent_label}_L{line}_{track_index}__`（`line` 为 `parallel` 块的源码行号，保证同一 label 内多个 parallel 块不冲突），有命名的 track 用命名作为后缀。`PARALLEL_BEGIN` 指令的 operand 存这些匿名 label 名的列表，交给 Scheduler 并发管理。

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

**Python 块执行环境**：`_exec_globals`（内部 dict，`Store` 的代理目标）作为 `globals`，frame.locals 作为 `locals`。变量写入 `_exec_globals` 后通过 `Store` 代理层对外可见，跨 label、跨 jump 天然持久化。引擎内置符号通过 `__builtins__` 注入为只读层，用户代码可见但不可覆盖。序列化时 `Store` 过滤所有 dunder key，`exec()` 自动写入的 `__builtins__`、`__name__` 等内部符号不进入存档。

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

## 内置功能库（Built-in Library）

Axn-Plus 随引擎附带一批开箱即用的功能模块。按实现层级分为三类：

| 类别 | 说明 |
|------|------|
| **引擎内置（Python核心层）** | 需要访问引擎内部状态、后端渲染API或平台底层能力，必须在Python层实现 |
| **标准库 `.apy`** | 纯业务逻辑+UI组合，用现有引擎指令可以完整表达，直接以 `.apy` 文件分发 |
| **两者都要** | 底层能力引擎提供，上层UI和配置以 `.apy` 模板分发 |

---

### 引擎内置（Python核心层）

#### 画廊（Gallery）

持久化解锁状态存入 `persistent`，缩略图离线缓存，资源懒加载（大图集按需解码，不全量预载）。纯 `.apy` 无法实现懒加载，超量图集会直接卡死渲染线程。

- 引擎层：缩略图缓存管理、懒加载调度、`persistent` 读写
- `.apy` 模板：默认画廊UI（`grid` 布局、解锁遮罩、全屏预览、翻页）

#### 图片预加载系统（Image Preloader）

视觉小说的图片切换密集，不预加载会产生明显卡顿。引擎在编译期从 AST 静态分析 `show` / `scene` / `expression` 指令的资源引用，构建预加载提示表；运行时在当前画面显示期间后台加载接下来若干步可能出现的资源。

**静态分析**：编译器第三遍扫描时，对每个 label 建立资源依赖列表（当前 label 用到的图片路径集合）。`$` 动态 `show $sprite` 无法静态分析，标记为"不可预测"，不纳入提示表。

**运行时行为**：VM 执行到一条指令时，预加载器在后台线程加载距离当前位置 N 步内（默认 `preload_lookahead = 3`）所有静态可知的图片资源。加载完成前被用到的资源走同步加载（阻塞一帧），并输出开发模式警告。

**缓存管理**：

```apy
# options_window.apy
engine:
    preload:
        lookahead      = 3        # 向前预看的指令步数
        cache_limit_mb = 256      # 图片缓存上限（MB），超出时 LRU 淘汰
        cache_limit_mb = 0        # 0 = 不限制（桌面端可选）
        evict_on_scene = true     # scene 切换时主动淘汰当前场景资源（节省内存）
```

**手动预加载**（用于分支预测不到的场景）：

```apy
preload "assets/autumn/angry.png"
preload "assets/bg/throne_room.png" "assets/autumn/formal.png"   # 多个
preload label route_a_start          # 预加载指定 label 的静态资源集合
```

**Android 特殊处理**：Android 系统在低内存时会主动回收进程，`cache_limit_mb` 默认值在 Android 平台自动降为 `128`，可在 `options_window.apy` 中覆盖。

**Round-Trip**：`preload` 指令对应脚本区独立积木块，路径列表可编辑；`preload label` 显示目标 label 名字段。

#### 动态/静态精灵（Sprite）

静态精灵为引擎现有能力。动态精灵即 `AnimatedSprite` 接口，已在核心设计中定义。此条目确认两者均为引擎内置，不通过 `.apy` 实现。

#### 自动演出——大图平移（PanningSprite）

超出视口的大图通过代码控制移动速度、位置、缩放。现有 `transform` 系统作用于标准尺寸对象，不处理 oversized surface 的裁剪与偏移，需要引擎渲染层新增支持。

引擎内置 `PanningSprite` 可显示类，实现 `AnimatedSprite` 接口，暴露以下参数给 `.apy` 层：

```apy
show PanningSprite("bg/city_wide.png") center (speed=(0.5, 0), loop)
show PanningSprite("bg/city_wide.png") (pos=(0.3, 0.0), size=1.2, duration=5.0)
```

| 参数 | 说明 |
|------|------|
| `speed` | `(x, y)` 每秒移动比例，相对图片尺寸 |
| `pos` | 当前视口锚点，`(0.0, 0.0)` 为左上角，`(1.0, 1.0)` 为右下角 |
| `size` | 缩放系数，`1.0` 为原始大小 |
| `duration` | 从当前位置移动到目标位置的时长（秒） |
| `loop` | 到达边界后反向，形成往返循环 |

#### 音频滤波器（Audio Filters）

实时 DSP 效果链，作用于单个音频通道的播放输出。纯 `.apy` 无法实现，需要后端音频管线支持（Pygame 通过 `pygame.mixer` 的后处理钩子实现，Qt 通过 `QAudioSink` 过滤器链实现）。

**使用语法：**

```apy
# 对指定通道施加滤波器
filter music (reverb(room=0.6, damp=0.5))           # 混响
filter music (lowpass(cutoff=800))                   # 低通（电话效果）
filter music (highpass(cutoff=3000))                 # 高通
filter music (pitch(factor=0.85))                    # 音调变换（降调）
filter music (echo(delay=0.3, decay=0.4))            # 回声
filter music (distortion(drive=0.5))                 # 失真

# 叠加多个效果
filter music (lowpass(cutoff=1200), reverb(room=0.3))

# 清除滤波器
filter music none

# 带过渡时间（在 N 秒内平滑插值到新参数）
filter music (reverb(room=0.8)) 1.0
```

**`voice` 通道的特殊用途**：常用于角色声音变形（危险状态、幻觉、变身场景）：

```apy
filter voice (pitch(factor=0.7), reverb(room=0.4))
autumn: "……这是你的声音吗？"
filter voice none 0.5
```

**`options_window.apy` 默认滤波器**：可为指定通道设置全局默认效果，游戏启动时自动应用：

```apy
engine:
    audio:
        default_filters:
            ambient: reverb(room=0.2, damp=0.7)   # 环境音默认带轻微混响
```

**内置滤波器参数：**

| 滤波器 | 参数 | 说明 |
|--------|------|------|
| `reverb` | `room` (0–1), `damp` (0–1), `wet` (0–1) | 混响 |
| `echo` | `delay` (秒), `decay` (0–1) | 回声 |
| `lowpass` | `cutoff` (Hz) | 低通，截断高频 |
| `highpass` | `cutoff` (Hz) | 高通，截断低频 |
| `pitch` | `factor` (倍率，1.0=原音调) | 音调变换 |
| `distortion` | `drive` (0–1) | 失真/破音 |
| `compressor` | `threshold` (dB), `ratio` | 动态压缩 |

**Round-Trip**：`filter` 指令对应脚本区音频滤波积木块；通道名下拉选择；效果列表可编辑；过渡时间字段可选；`filter music none` 显示为"清除效果"节点。

#### 鼠标自动切换（Cursor Manager）

涉及系统鼠标指针API（Pygame：`pygame.mouse.set_cursor`；Qt：`QCursor`），`.apy` 层无法直接触达。

引擎内置 cursor 注册表，支持按控件类型或命名区域自动切换指针样式。注册规则在 `options_window.apy` 中配置：

```apy
engine:
    cursor:
        default   "assets/cursor/default.png"
        hover     "assets/cursor/pointer.png"
        text      "assets/cursor/beam.png"
        drag      "assets/cursor/grab.png"
        wait      "assets/cursor/spinner.png"
```

`button`、`slider`、`input_field` 等交互控件自动触发对应 cursor 状态，无需手动声明。自定义区域通过 `cursor=` 具名参数覆盖：

```apy
image "map/region_a.png" (cursor=hover, on_click: jump region_a)
```

#### 乱码效果（ScrambleText）

全局乱码和局部乱码均需劫持 `TextRenderer` 渲染管线，在字符级别做随机替换或扰动，必须在引擎层实现。

**全局乱码**：在 `options_window.apy` 中开关，影响所有对话文本：

```apy
engine:
    scramble:
        enabled   = False
        intensity = 0.3        # 0.0–1.0，被替换字符的比例
        charset   = "glitch"   # 替换字符集：glitch / katakana / binary / custom
        speed     = 0.05       # 每帧刷新间隔（秒），控制闪烁频率
```

运行时通过 `store` 变量动态切换：

```apy
$ engine.scramble.enabled = True
$ engine.scramble.intensity = 0.6
```

**局部乱码**：在 `style` 系统中新增 `scramble` 类型，作用于单个控件或对话行：

```apy
style glitch_text:
    scramble:
        intensity 0.5
        charset   "katakana"
        speed     0.03

# 对话行内联
autumn: "̴̢̛y̵̛o̷̕u̸̧ ̶͝c̵̀a̸͝n̴̛'̸̀t̸̡ ̷̕ȩ̴s̷̀c̷͝a̵͝p̴̛e̸̡" (style=glitch_text)

# 控件内联
text "ERROR" (scramble=(intensity=0.8, charset="binary"))
```

`charset` 内置选项：

| 值 | 替换字符集 |
|----|-----------|
| `glitch` | Unicode 控制字符 + 组合字符（̴̢̛ 类） |
| `katakana` | 全角片假名（ァ-ン） |
| `binary` | `0` / `1` |
| `hex` | `0-9A-F` |
| `custom` | 由 `charset_chars` 字段自定义字符串 |

#### Downloader（资源下载器）

支持空壳包体启动后在线下载完整资源，适用于包体大小受限的发布渠道（如 Google Play 初始包体限制）。

引擎层实现：
- 网络请求（`httpx`，支持断点续传）
- 文件完整性校验（SHA-256）
- 下载队列与并发控制
- Android 存储权限运行时申请（Android 13+ 需要 `READ_MEDIA_*` 权限）
- 下载状态持久化（异常退出后续传）

`.apy` 模板提供默认下载界面（进度条、网速显示、错误重试UI），开发者可替换样式。

在 `flow.apy` 的 `start` label 中设置调用点：

```apy
label start:
    call axn::downloader.check_and_download
    call chapter1.apy::prologue
```

`options_window.apy` 中配置下载源：

```apy
engine:
    downloader:
        manifest_url  = "https://example.com/game/manifest.json"
        target_dir    = "downloaded/"
        verify_hash   = true
        max_retries   = 3
        concurrent    = 2
```

#### 归档（Asset Archive）

将资源文件打包为单个 `.npa` 归档文件，引擎在 `asset/loader.py` 层拦截路径解析，透明地从归档内读取资源。开发者和 `.apy` 脚本无需感知资源来自文件系统还是归档。

`axn build` 扩展参数：

```
axn build --pack assets/dlc1/ --output dlc1.npa
```

运行时导入归档（须在 `flow.apy` 中有调用点）：

```apy
label start:
    $ engine.mount_archive("dlc1.npa")
    call chapter1.apy::prologue
```

归档内的资源引用方式与普通路径完全一致，无新语法。

归档格式：自定义二进制格式，文件头包含索引表（路径 → 偏移+长度），支持可选的整体加密（AES-256-GCM，密钥在构建时注入）。

#### 自定义引擎报错（Error Screen）

允许脚本层主动拉起引擎原生报错界面，用于游戏内"故意报错"的叙事演出（如第四面墙破坏、系统崩溃彩蛋等）。

拆分为两条指令，语义明确不混用：

**`raise engine_error`**：走真实错误流程，中断执行，写入 `crash.log`，触发引擎错误处理矩阵。用于真正的错误场景。

**`show error_screen`**：纯视觉模拟，不中断执行流，不写 log，本质是特殊样式的 `modal`。用于叙事演出：

```apy
show error_screen:
    title    "FATAL ERROR"
    code     "0x0000DEAD"
    message  "Memory access violation at 0x{store['player_id']:08X}"
    style    "bsod"          # 内置样式：bsod / axn_native / custom
    on_close: jump after_glitch_scene
```

内置样式：

| 样式 | 外观 |
|------|------|
| `bsod` | Windows 蓝屏风格 |
| `axn_native` | 引擎原生报错界面（与真实错误界面视觉完全一致） |
| `custom` | 开发者自定义，通过 `template` 提供完整控件树 |

`axn_native` 样式下显示的内容与真实报错界面完全相同，包括 traceback 区域（内容由脚本填入，非真实调用栈）。开发者需在 `options_window.apy` 中显式启用此功能，避免误用：

```apy
engine:
    allow_fake_error_screen = false   # 默认关闭，设为 true 才允许 show error_screen
```

未启用时使用 `show error_screen` 会运行时输出警告并忽略：

```
AxnWarning: [engine] 'show error_screen' is disabled.
  Set 'engine: allow_fake_error_screen = true' in options_window.apy to enable.
  → scene.apy, line 42
```

#### 成就系统（Achievement）

成就注册、触发、持久化存储（`persistent`），以及平台 API 对接。

**成就声明**（在 `options_window.apy` 或任意顶层 `.apy` 文件中）：

```apy
define achievement first_meeting:
    name        "初次相遇"
    description "第一次见到了 autumn。"
    icon        "ui/ach/first_meeting.png"
    hidden      = false       # true 时在未解锁前隐藏名称和描述

define achievement true_ending:
    name        "真实结局"
    description "到达了游戏的真正结局。"
    icon        "ui/ach/true_ending.png"
    hidden      = true
```

**触发**：

```apy
unlock achievement first_meeting        # 解锁，重复调用静默忽略
unlock achievement true_ending

# 条件解锁语法糖
unlock achievement first_meeting if not persistent.achievements["first_meeting"]
```

**查询**：

```apy
$ is_unlocked = achievement.is_unlocked("first_meeting")
$ all_unlocked = achievement.all_unlocked()    # list[str]
```

**持久化**：解锁状态自动写入 `persistent.achievements`（`dict[str, bool]`），无需开发者手动管理。

**Steamworks API 对接**（见下文）：`unlock achievement` 触发时，若 Steam 平台可用，引擎自动调用 `ISteamUserStats::SetAchievement`，无需脚本层额外代码。

**`options_window.apy` 配置**：

```apy
engine:
    achievements:
        provider = "auto"    # "auto"（自动检测平台）/ "steam" / "local"（仅本地，不上报）
```

**Round-Trip**：`define achievement` 对应成就定义积木块，字段完整可解析；`unlock achievement` 对应解锁节点，成就名下拉选择（来自符号表）；编辑器成就面板显示所有已定义成就和当前解锁状态。

---

#### Steamworks API

Steam 平台功能的引擎层封装，通过 `ctypes` 加载 `steam_api.dll` / `libsteam_api.so`，不依赖第三方 Python 包。在 `options_window.apy` 中开启：

```apy
engine:
    extensions:
        builtins:
            steamworks = true
```

**自动初始化**：引擎启动时检测 `steam_appid.txt` 是否存在，存在则自动初始化 Steamworks，失败时静默降级（不阻止游戏启动）。

**成就自动同步**：`unlock achievement` 触发后，引擎自动调用 `SetAchievement` + `StoreStats`，无需脚本层额外代码。

**Steam 云存档**：与 Cloud Save 插件集成，Steamworks 作为一种 provider：

```apy
engine:
    cloud_save:
        provider = "steam"    # 使用 Steam 云存档
```

**DLC 检测**：

```apy
$ has_dlc = engine.steam.is_dlc_owned(480)   # 传入 DLC AppID
if has_dlc:
    call axn::dlc_chapter3.start
```

**排行榜**：

```apy
# 上传分数
$ engine.steam.upload_leaderboard("speedrun_ch1", store["clear_time"])

# 下载排行榜（异步，结果通过回调或 store 变量返回）
$ engine.steam.fetch_leaderboard("speedrun_ch1", count=10, callback="on_leaderboard_fetched")

on_event channel="global" "on_leaderboard_fetched": (entries):
    $ store["leaderboard"] = entries
    show screen leaderboard_panel
```

**开发模式**：未安装 Steam 或 `steam_appid.txt` 不存在时，所有 `engine.steam.*` 调用返回安全默认值（成就静默忽略，排行榜返回空列表），不报错，方便无 Steam 客户端的开发环境调试。

---

#### SDL2 原生扩展（Native SDL2）

Pygame 底层已基于 SDL2，此扩展暴露 Pygame 未封装的原生 SDL2 能力，通过 `pysdl2` bindings 实现。**现阶段绝大多数项目不需要此扩展**，仅在有明确需求时启用。

在 `options_window.apy` 中开启：

```apy
engine:
    extensions:
        builtins:
            sdl2_native = true
```

**当前暴露的能力：**

| 功能 | API | 说明 |
|------|-----|------|
| 手柄震动（Rumble） | `engine.sdl2.rumble(low, high, duration_ms)` | 视觉小说用途有限，存在即可 |
| 高精度定时器 | `engine.sdl2.get_ticks_ns()` | 纳秒级时间戳 |
| 传感器（陀螺仪等） | `engine.sdl2.sensor` | Android 特有，适合体感交互彩蛋 |
| 剪贴板读写 | `engine.sdl2.clipboard` | 读取 / 写入系统剪贴板 |
| 屏幕亮度 | `engine.sdl2.set_brightness(val)` | 0.0–1.0，部分平台支持 |

**不在此扩展覆盖范围内：**

- HDR / 高刷新率显示——Pygame 2.x 不支持，SDL2 原生支持有限，视觉小说无实际需求
- 多窗口——Axn-Plus 为单窗口设计，不计划支持
- Vulkan / Metal 渲染——超出当前架构范围

**使用示例：**

```apy
# 手柄震动（对话中的演出效果）
if engine.variant("gamepad"):
    $ engine.sdl2.rumble(0.3, 0.8, 500)   # 低频0.3, 高频0.8, 持续500ms

# 剪贴板（复制对话文本）
button "复制台词" on_click:
    $ engine.sdl2.clipboard.set(store["last_dialogue"])
```

**依赖说明**：`pysdl2` 需要系统已安装 SDL2 动态库（Windows / macOS 打包时随包附带，Linux 由系统包管理器提供，Android 由 Pygame 的 SDL2 构建覆盖）。未安装时 `engine.sdl2` 所有调用静默返回 `None` 并输出警告，不阻止游戏运行。

---

#### 自配音 / 无障碍朗读（Self-Voicing）

系统 TTS（Text-To-Speech）朗读对话文本和 UI 标签，供视障用户使用。

**平台 TTS 后端：**

| 平台 | 实现 |
|------|------|
| Windows | SAPI 5（`win32com.client`） |
| macOS | `AVSpeechSynthesizer`（通过 `subprocess` 调用 `say` 命令） |
| Linux | `espeak-ng`（需系统安装） |
| Android | `android.speech.tts.TextToSpeech`（通过 Pyjnius） |

**开关**：

```apy
$ preferences.self_voicing = True    # 开启
$ preferences.self_voicing = False   # 关闭
```

键盘快捷键默认绑定在 `engine.keymap.self_voicing_toggle`（默认 `v`）。

**朗读优先级**：

1. 对话行的 TTS 文本（`voice_text=` 具名参数，用于与显示文本不同的朗读内容）
2. 对话行的显示文本（去除富文本标签后的纯文字）
3. UI 控件的 `accessibility_label=` 参数
4. UI 控件的 `label` 内容（`button`、`text` 等）

```apy
autumn: "…" (voice_text="autumn 沉默了。")    # 显示省略号，朗读描述性文字
button "×" (accessibility_label="关闭对话框") on_click: Return()
```

**朗读时机**：

- 对话行：文字开始显示时触发（不等打字机效果完成）
- UI 控件获得焦点时触发
- `menu` 选项获得焦点时朗读选项文本

**`options_window.apy` 配置**：

```apy
engine:
    self_voicing:
        enabled_by_default = false
        read_narrator      = true    # 旁白是否朗读
        read_ui            = true    # UI 控件是否朗读
        interrupt          = true    # 新文本触发时打断上一段朗读
```

---

#### 角色回调（Character Callbacks）

在角色说话生命周期的关键节点注入自定义逻辑，不需要在每条对话行前后手动插入代码。

**声明**（在 `define char` 内）：

```apy
define char autumn:
    name "autumn"
    sprites "assets/autumn/"
    callbacks:
        on_start:    autumn_on_start      # 对话行开始显示时触发
        on_advance:  autumn_on_advance    # 用户点击推进时触发（含 <w> 中途点击）
        on_end:      autumn_on_end        # 对话行完全结束后触发
        on_voice:    autumn_on_voice      # 语音开始播放时触发
```

回调函数在 Python 中定义：

```python
def autumn_on_start(event):
    # event.char     : 角色对象
    # event.text     : 当前对话文本（已插值）
    # event.line     : 源码行号
    # event.filename : 源码文件名
    store["last_speaker"] = "autumn"

def autumn_on_end(event):
    store["dialogue_count"] += 1
```

**全局回调**（作用于所有角色）：

```apy
# options_window.apy
engine:
    character_callbacks:
        on_start:   global_dialogue_start
        on_end:     global_dialogue_end
```

全局回调在角色自身回调之后触发。

**`voice_tag`**：为角色声音分组，方便统一控制音量（如"所有 NPC 音量"独立于"主角音量"）：

```apy
define char autumn:
    voice_tag "heroine"    # 音量分组标签

define char merchant:
    voice_tag "npc"
```

```apy
# options_window.apy
preferences:
    heroine_volume = 1.0    # 自定义偏好项，绑定到 voice_tag
    npc_volume     = 0.8

engine:
    audio:
        voice_tag_bindings:
            heroine = preferences.heroine_volume
            npc     = preferences.npc_volume
```

**Round-Trip**：`callbacks:` 子块在角色定义积木块中显示为回调面板，各回调函数名字段可编辑；`voice_tag` 显示为独立字段，下拉选择已定义的标签。

---

#### 场景回放与音乐空间（Scene Replay / Music Room）

**场景回放**：允许玩家从主菜单重新观看已解锁的剧情段落。

解锁触发：在剧情脚本中调用 `unlock scene`：

```apy
label ch1_ending:
    unlock scene "第一章结局" (
        label     = ch1_ending,
        thumb     = "ui/replay/ch1_end.png",
        category  = "第一章"
    )
    autumn: "再见。"
```

重放时的 `store` 隔离：回放期间引擎创建一个 `store` 快照副本，回放结束后恢复原始 `store`，游戏状态不受影响。回放内的 `checkpoint` 指令静默忽略（不触发存档）。

引擎层处理：`persistent` 解锁记录、`store` 隔离沙箱、回放期间 Auto 模式默认开启；`.apy` 模板提供默认回放选择界面。

```apy
call axn::scene_replay.show     # 打开回放选择界面
```

**音乐空间**：展示已解锁 BGM，可试听，带进度条和曲目信息。

解锁触发：

```apy
unlock music "bgm/morning.ogg" (
    title    = "清晨",
    composer = "...",
    thumb    = "ui/music/morning.png",
    loop     = true
)
```

```apy
call axn::music_room.show     # 打开音乐空间界面
```

`options_window.apy` 配置：

```apy
engine:
    scene_replay:
        store_isolation = true     # 回放期间隔离 store（默认 true，强烈建议保持）
        auto_mode       = true     # 回放期间默认开启 Auto 模式
    music_room:
        loop_by_default = true
```

---

#### HTTP 请求（Fetch）

引擎提供异步 HTTP 接口，不阻塞游戏主循环，基于 `httpx` 实现（与 Downloader 共用同一底层）。

```apy
# 发起请求，结果通过回调或 store 变量返回
$ engine.fetch(
    url      = "https://api.example.com/leaderboard",
    method   = "GET",
    headers  = {"Authorization": f"Bearer {store['token']}"},
    callback = "on_fetch_done"
)

# 等待结果（阻塞执行流，适合加载界面）
$ result = engine.fetch_sync("https://api.example.com/save", method="POST", json=store.to_dict())
```

**异步回调模式**：

```apy
on_event channel="global" "on_fetch_done": (response):
    if response.status == 200:
        $ store["leaderboard"] = response.json()
    else:
        $ store["fetch_error"] = response.status
```

**安全限制**：发布包中，所有允许的域名须在 `options_window.apy` 中白名单声明，未声明域名的请求在发布包中被拒绝（开发模式不限制）：

```apy
engine:
    fetch:
        allowed_domains:
            - "api.example.com"
            - "cdn.example.com"
        timeout = 10.0    # 超时秒数
```

---

### 标准库 `.apy`

以下模块均以 `.apy` 文件形式分发，开发者按需放入项目的 `main/axn/` 目录，通过 `include` 或跨文件 `call` 引入。所有扩展统一放在 `main/axn/` 下，不分子目录。

#### 密码校验（Password Gate）

`input_field` + `hashlib` 校验 + 条件跳转的组合模板。默认使用 SHA-256，支持自定义 hash 函数。

```apy
call axn::password_gate.show(
    prompt   = "输入密码",
    hash     = "5e884898da...",   # SHA-256 of "password"
    on_pass  = secret_route,
    on_fail  = wrong_password
)
```

#### 彩蛋（Easter Egg）

提供快速的彩蛋注册与触发模板。触发方式支持按键序列（Konami code 风格）和隐藏点击区。

```apy
# 在 options_window.apy 或任意顶层位置注册
on key "up up down down left right left right b a":
    call axn::easter_egg.trigger("konami")

# 彩蛋内容定义
label easter_egg_konami:
    $ engine.scramble.enabled = True
    autumn: "你找到我了。" (glitch_text)
    $ engine.scramble.enabled = False
```

#### 演出人员名单（Credits）

滚动文字 + 背景音乐 + 跳过按钮。支持分组（章节/角色/Staff）、字体大小分级、Logo 插入。

```apy
call axn::credits.show(
    data     = "credits/staff.json",
    bgm      = "bgm/ending.ogg",
    speed    = 60,               # px/秒
    on_done  = ending_label
)
```

#### 背包（Inventory）

数据层：`store` 内的 `list[dict]`，每个 item 包含 `id`、`name`、`icon`、`description`、`count` 等字段。UI层：`grid` + `tooltip` + 过滤/排序面板。

```apy
# 添加物品
$ axn_inventory.add(store["inventory"], {"id": "key_001", "name": "古旧的钥匙", "icon": "items/key.png", "count": 1})

# 打开背包界面
call axn::inventory.show
```

#### 选项框（Option Dialog）

`menu` 的标准UI封装，带标题、描述文本、选项列表。

```apy
call axn::option_dialog.show(
    title   = "选择你的路线",
    options = [("前往北区", "north"), ("留在原地", "stay")]
) as choice
jump choice
```

#### 复合确认框（Confirm Dialog）

集成返回、确认与可展开提示的三态确认框，基于 `modal show` + `dialog` 实现。

```apy
modal show axn::confirm_dialog(
    title   = "确定要删除存档吗？",
    hint    = "删除后无法恢复，包括所有解锁内容。",
    confirm = "删除",
    cancel  = "取消"
) as result

if result == "confirm":
    $ engine.delete_save(slot_id)
    jump save_menu
```

#### 信息聚合（Clue Board）

类解密游戏线索板。数据结构存 `store`，UI 支持节点卡片 + 连线（连线通过 `canvas` Python逃逸绘制）+ 解锁动画。

```apy
# 添加线索
$ axn_clues.add(store["clues"], {"id": "clue_001", "title": "碎纸片", "content": "...", "unlocked": True})
$ axn_clues.connect(store["clues"], "clue_001", "clue_003")

# 打开线索板
call axn::clue_board.show
```

#### 章节选择（Chapter Select）

解锁状态存 `persistent`，UI 用 `grid` 布局，每章节显示标题、缩略图、解锁状态遮罩。

```apy
call axn::chapter_select.show(
    chapters = [
        {"id": "ch1", "title": "第一章", "thumb": "ui/ch1_thumb.png"},
        {"id": "ch2", "title": "第二章", "thumb": "ui/ch2_thumb.png"},
    ]
)
```

#### 动态标题（Dynamic Title Screen）

标题界面以 `screen` + `gui` 实现，所有视觉元素绑定 `store` 变量，支持运行时通过代码修改排版、背景、BGM、按钮组合。

```apy
# 修改标题界面配置
$ axn_title.config.background = "bg/title_winter.png"
$ axn_title.config.bgm = "bgm/title_winter.ogg"
$ axn_title.config.show_continue = persistent.has_save

call axn::title_screen.show
```

#### 地图（Map）

静态地图：`image` + 透明 `button` 热区覆盖。动态地图：节点解锁状态存 `store`，节点间连线通过 `canvas` 绘制，支持节点动画（新解锁闪烁）。

```apy
call axn::map.show(
    background = "map/world.png",
    nodes      = store["map_nodes"],   # list[dict]，含位置、解锁状态、跳转目标
    on_select  = map_node_selected
)
```

#### 标准 UI 画面（Standard Screens）

提供开箱即用的标准界面模板，开发者可直接使用或替换样式。所有界面基于 `screen` + `gui` 实现，通过 `call axn::screens.xxx` 调用。

**存档 / 读档界面**：

```apy
call axn::screens.save_menu                       # 存档界面（阻塞）
call axn::screens.load_menu                       # 读档界面（阻塞）
call axn::screens.save_menu (slots=15)            # 自定义存档槽数量
```

存档缩略图自动截取当前画面（如有 `checkpoint (thumbnail=current)` 则用 checkpoint 缩略图），显示日期、时间、章节名。

**设置界面**：

```apy
call axn::screens.settings                        # 设置界面（阻塞）
```

自动覆盖所有 `preferences` 内置项（文字速度、音量、全屏、跳过模式等），自定义偏好项按类型渲染控件（`bool` → toggle，`float` → slider，`str` → dropdown）。

**暂停菜单**：

```apy
call axn::screens.pause_menu
```

默认包含：继续游戏、存档、读档、设置、回到标题、退出。各按钮跳转目标可在 `options_window.apy` 中配置：

```apy
engine:
    pause_menu:
        show_save   = true
        show_load   = true
        show_title  = true
        title_label = "main_title"    # 点击"回到标题"跳转的 label
```

**右键菜单**：

鼠标右键 / 手柄 B 键呼出，默认内容：历史记录、自动模式、跳过、隐藏对话框、截图、设置。

```apy
# options_window.apy
engine:
    right_click_menu:
        enabled  = true
        items:
            - history
            - auto
            - skip
            - hide_window
            - screenshot
            - settings
```

**确认退出对话框**：

```apy
call axn::screens.confirm_quit         # 弹出"确定要退出吗？"，确认后 engine.quit()
```

**关于界面**：

```apy
call axn::screens.about (
    title   = "游戏名称",
    version = "1.0.0",
    credits = "制作：...",
    engine  = true         # 是否显示"Powered by Axn-Plus"
)
```

所有标准界面的样式通过 `style` 系统覆盖，不需要修改模板文件本身。

---

#### 互动调试器（Interactive Director）

开发模式专属工具，游戏运行时的实时演出调试，不依赖 Axn-Editor，通过键盘快捷键呼出。

**呼出方式**：`Shift+D`（可在 `options_window.apy` 中重映射）。

**功能：**

- **立绘调整**：实时拖动当前场景中任意立绘的位置，调整后生成对应的 `show` 指令代码，可一键复制到剪贴板
- **转场预览**：浏览并预览所有已注册的过渡效果（内置 + 自定义），选择后生成对应参数代码
- **表情切换**：对当前可见角色实时切换表情（`states` / `layers` 均支持），生成对应 `expression` 指令
- **Transform 预览**：对当前选中对象实时应用并预览 transform，调整 duration/easing，生成代码
- **颜色矩阵预览**：实时预览 `color_matrix` 效果，滑块调整参数

**生成代码格式**：所有调整操作在右侧面板实时显示等效 `.apy` 代码，可整体复制。

**发布包中完全剥离**，不包含在包体内。

---

#### 自动化测试（Automated Testing）

无头模式下模拟用户操作，自动跑通游戏流程，用于回归测试。

**测试脚本格式**（`.test` 文件）：

```
# tests/basic_flow.test
start at: start
click                          # 模拟点击（推进对话）
click 5                        # 连续点击 5 次
choose: "答应她"               # 选择菜单选项（按文本匹配）
choose index: 0                # 选择第一个选项
wait label: route_a            # 等待执行流到达指定 label
assert store: flag_agreed == True
assert store: day >= 1
assert label: morning_scene    # 断言当前在指定 label
click until label: ch1_end     # 持续点击直到到达指定 label（有安全上限）
screenshot: "tests/shots/ch1_end.png"   # 截图对比（与基准图对比，差异超阈值报错）
```

**运行方式**：

```
axn test tests/basic_flow.test          # 运行单个测试
axn test tests/                            # 运行目录下所有测试
axn test --headless tests/                 # 无头模式（不显示窗口）
axn test --record tests/basic_flow.test # 录制模式：实际操作游戏，自动生成测试脚本
```

**录制模式**：开发者实际操作游戏，引擎记录所有点击和选择，自动生成 `.test` 文件，降低编写测试的成本。

**截图对比**：首次运行时生成基准截图存入 `tests/baseline/`，后续运行时对比，像素差异超过阈值（默认 1%，可配置）时测试失败。

**`options_window.apy` 配置**：

```apy
engine:
    test:
        screenshot_threshold = 0.01    # 截图对比差异阈值（0–1）
        max_clicks           = 10000   # click until 的安全上限
        timeout              = 300     # 单个测试超时秒数
```

---

### 两者都要

#### 视差跟随（Parallax）

渲染层需要感知各图层的视差系数，按系数差异响应鼠标移动或镜头位移，这是后端 `renderer.py` 的责任。上层配置通过扩展 `layer create` 指令声明：

```apy
layer create bg_far   (above=bg, parallax=0.1)    # 移动量为鼠标偏移的 10%
layer create bg_near  (above=bg_far, parallax=0.4)
layer create sprite   (above=bg_near, parallax=1.0)  # 跟随鼠标1:1移动（默认）
```

`parallax` 值为 `0.0`（完全固定）到 `1.0`（完全跟随）。未声明时默认 `1.0`。视差响应源在 `options_window.apy` 中配置：

```apy
engine:
    parallax:
        source    = "mouse"    # mouse / camera / both
        intensity = 0.05       # 鼠标偏移到实际移动量的缩放系数
        smoothing = 0.15       # 插值平滑系数（0 = 即时，1 = 不移动）
```

#### 通知系统（Notify）

**引擎层**：

游戏内通知：通知队列管理（并发上限、优先级、去重）、持续时间计时、进入/退出动画调度。

系统级通知（平台API封装）：
- Windows：`win10toast` 或 Windows Runtime `ToastNotification` API
- macOS：`NSUserNotificationCenter`（10.14以下）/ `UNUserNotificationCenter`（10.14+）
- Android：通过 Pyjnius 调用 `NotificationManager`，Android 13+ 运行时申请 `POST_NOTIFICATIONS` 权限
- Linux：`libnotify` / `notify-send`

统一抽象接口，`.apy` 层无需感知平台差异：

```apy
# 游戏内通知（核心指令）
notify "解锁了新CG：夏日记忆" (icon="ui/cg_icon.png", duration=3.0, priority=normal)

# 系统级通知（游戏在后台或最小化时）
notify system "新内容已下载完成" (subtitle="第三章现已可玩", icon="app_icon.png")
```

**`.apy` 模板**：

游戏内通知UI（位置、样式、动画、堆叠方式），开发者可替换默认模板：

```apy
engine:
    notify:
        position  = "top_right"      # top_right / top_center / bottom_right 等
        max_stack = 3                # 同时显示上限
        duration  = 3.0              # 默认持续时间（秒）
        gap       = 8                # 通知之间的间距（px）
```

#### Cloud Save（可选插件）

**定位**：可选插件，不随引擎默认安装，开发者按需引入。不同平台的账号体系差异过大，不适合强制内置。

**引擎层**：网络请求（`httpx`）、本地/云端存档冲突检测与解决、序列化格式与本地存档统一。

**`.apy` 模板**：同步状态UI（上传中、下载中、冲突选择界面）。

冲突解决策略在 `options_window.apy` 中配置：

```apy
engine:
    cloud_save:
        provider      = "custom"          # custom（自建后端）
        endpoint      = "https://..."
        conflict      = "newer_wins"      # newer_wins / local_wins / cloud_wins / ask_user
        auto_sync     = true
        sync_interval = 300               # 秒
```

`ask_user` 策略触发时拉起 `.apy` 冲突选择模板，展示本地与云端存档的时间戳、截图和关键变量差异，由玩家决定保留哪个版本。

---

## 执行流控制补充

### `unwind`（展开调用栈）

`unwind` 清空当前整个 call 栈，回到最顶层执行流继续，等价于"从任意嵌套深度强制返回到起点"。

```apy
unwind                        # 清空 call 栈，回到顶层继续执行
unwind to morning_scene       # 展开到指定 label（必须在当前栈上，否则运行时报错）
```

与 `return` 的区别：`return` 只退出当前 label，回到直接调用方；`unwind` 展开整个栈。

**不使用 `return exit` 命名**：视觉上容易误读为"返回并退出游戏"，与 `exit` 指令产生歧义。

### `exit`（退出游戏）

```apy
exit                          # 立即退出
exit (confirm=true)           # 弹出确认框，用户确认后退出
exit (save=autosave)          # 退出前触发自动存档
```

`exit` 不支持返回值语义。需要跨会话传递数据时，使用 `persistent` 或存档机制：

```apy
$ persistent["last_choice"] = "agreed"
exit (save=autosave)
```

`exit (save=...)` 接受存档槽名称，退出前自动调用 `engine.save(slot=...)`，不产生回溯点，只做数据持久化。

### `expoint` 调用点返回行为

`expoint name` 调用时：

- 有定义：执行定义块，遇到末尾或 `return` 后回到调用点的下一行继续，与 `call` 语义完全一致
- 无定义：静默跳过，继续执行下一行

定义块内禁止 `jump`，允许 `call`、`return`。

### 程序化存档 / 读档（Python API）

`Save()` / `Load()` 不作为独立 `.apy` 指令存在，这是 Ren'Py 因 screen 语言与 Python 割裂产生的历史遗留设计。

在 `gui` / `screen` 块内直接调用 Python API：

```apy
button "存档" on_click: engine.save(slot=store["selected_slot"])
button "读档" on_click: engine.load(slot=store["selected_slot"])
```

---

## 项目结构补充

### 入口 Label

引擎只识别 `start` 作为唯一合法入口 label：

| 情况 | 行为 |
|------|------|
| `start` 存在 | 正常启动 |
| `start` 不存在 | 启动时报 `AxnCompileError`，提示需要定义 `start` label |

不支持 `main` 或其他命名作为入口。`main` 在 Python 生态有特殊含义（`if __name__ == "__main__"`），用作 label 名会造成认知干扰，不做兼容。

### `project.json`

项目根目录下的 `project.json` 负责项目元数据和构建配置，与 `options_window.apy` 职责严格分离：

| 文件 | 内容 | 读取时机 |
|------|------|---------|
| `project.json` | 项目元数据、平台配置、构建策略 | `axn build` 时 |
| `options_window.apy` | 引擎运行时行为、UI 模板、偏好默认值 | 游戏启动时 |

`axn init` 创建项目时自动生成，启动器 GUI/TUI 直接编辑此文件，无需手动修改 JSON。

```json
{
    "name": "My Visual Novel",
    "version": "1.0.0",
    "author": "Studio Name",
    "backend": "pygame",
    "resolution": [1280, 720],
    "theme": "default",

    "build": {
        "runtime": "auto",
        "assets": "bundle",
        "exclude_packages": [],
        "include_packages": []
    },

    "android": {
        "package": "com.studio.gamename",
        "version_code": 1,
        "version_name": "1.0.0",
        "min_sdk": 21,
        "target_sdk": 34,
        "keystore": "release.jks",
        "keystore_alias": "mykey"
    },

    "macos": {
        "bundle_id": "com.studio.gamename",
        "sign_identity": ""
    },

    "windows": {
        "output": "installer"
    },

    "linux": {
        "output": "appimage"
    }
}
```

`options_window.apy` 中之前提到的构建相关配置（`engine.extensions`、平台参数等）统一迁移到 `project.json`，`options_window.apy` 只保留运行时有效的配置。

---

## 构建系统

### 构建维度

构建参数分两个独立维度，可自由组合：

**`--runtime`：Python 运行时裁剪策略**

| 值 | 行为 |
|----|------|
| `auto`（默认） | 静态扫描依赖，按需打包 |
| `full` | 完整 CPython + 完整标准库，不裁剪 |
| `minimal` | 只打引擎核心所需，极致压缩 |

**`--assets`：资源打包策略**

| 值 | 行为 |
|----|------|
| `bundle`（默认） | 资源打进包体 |
| `remote` | 资源从服务器拉取（Downloader 模式） |
| `split` | 核心资源打包，大文件远程拉取 |

组合示例：

```
axn build --platform=android --runtime=minimal --assets=remote
# → 最小启动包，资源全部远程拉取，目标包体 < 10MB
```

### 依赖裁剪机制

分三类处理：

**纯 Python 包**（httpx、pyyaml 等）：静态扫描所有 `.apy` 文件的 `python:` 块和 `$` 行，提取 `import` 语句。未出现的包不打入包体。

**有 C 扩展的 Python 包**（numpy、pillow 等）：同样按 import 扫描决定是否打入，但无法做子模块裁剪，整个 `.so` 都要带上。pillow 例外，按实际用到的图片格式只编译对应 codec。

**系统级 C 库**（FFmpeg、SDL2、Qt）：必须在编译期决定，按下文各节说明裁剪。

静态扫描无法覆盖动态 import（`importlib`、字符串拼接模块名等），开发者通过 `project.json` 补充声明：

```json
"build": {
    "include_packages": ["my_dynamic_import_lib"],
    "exclude_packages": ["tkinter"]
}
```

**标准库裁剪**：`--runtime=auto` 下默认排除以下模块（视觉小说项目不需要）：`tkinter`、`turtle`、`unittest`、`pdb`、`idlelib`、`ensurepip`、`lib2to3`。

### FFmpeg 裁剪

`axn build` 时扫描项目内所有资源文件扩展名 + `.apy` 脚本里静态可知的 `play`/`play video`/`scene` 路径引用，得到实际用到的格式集合，据此决定编译哪些 codec。

支持的格式与对应 FFmpeg 组件：

**图片**

| 格式 | 说明 |
|------|------|
| png | 内置，无额外依赖 |
| jpg / jpeg | libjpeg |
| webp | libwebp |
| svg | librsvg（仅静态 SVG，不支持动画） |

**视频**

| 格式 | 解码器 |
|------|--------|
| mp4 / mov | libx264 + AAC |
| webm | libvpx + libvorbis / libopus |
| avi | 按实际编码决定 |

**音频**

| 格式 | 解码器 |
|------|--------|
| mp3 | libmp3lame |
| wav | 内置 |
| flac | 内置 |
| m4a | AAC（与 mp4 共用） |
| ogg | libvorbis |

`--runtime=auto` 下默认只打 mp4 + ogg + wav 的解码器（覆盖视觉小说 95% 的场景），其他格式在扫描结果中出现时自动追加。

### Qt 模块裁剪

扫描 `.apy` 文件中 `qt:` 块的 import 语句，加上引擎本身用到的 Qt 模块，合并决定打哪些：

```
axn build --qt-modules=auto          # 扫描决定（默认）
axn build --qt-modules=core,gui,widgets   # 手动指定
```

默认排除：`QtWebEngine`、`Qt3D`、`QtDataVisualization`。`QtMultimedia` 由 FFmpeg 替代，默认不打。

### 平台构建

**Windows**

```
axn build --platform=windows --output=installer   # NSIS 安装包（默认）
axn build --platform=windows --output=portable    # 可执行目录，无需安装
```

不推荐 `--output=exe`（单文件）：启动时需解压到临时目录，视觉小说启动体验差。

**macOS**

```
axn build --platform=macos --output=dmg     # DMG 磁盘镜像（默认）
axn build --platform=macos --output=app     # 只生成 .app bundle
```

需要代码签名和公证（Notarization），macOS 14+ 未签名应用会被拦截。通过 `project.json` 的 `macos.sign_identity` 字段配置签名证书，留空时生成未签名版本并输出警告。

**Linux**

```
axn build --platform=linux --output=appimage   # 推荐，依赖自包含，任意发行版可运行
axn build --platform=linux --output=deb        # Debian/Ubuntu 包
axn build --platform=linux --output=tar        # 纯压缩包
```

**Android**

```
axn build --platform=android --output=apk    # 直接安装包
axn build --platform=android --output=aab    # Google Play 要求格式
```

**Android 发布构建的准备顺序**（借鉴 Ren'Py 流程）：

1. **设置包名**：在 `project.json` 的 `android.package` 字段填写包名（如 `com.studio.gamename`），包名一旦发布后不可更改。使用 `axn android set-package com.studio.gamename` 可直接修改并验证格式合法性。

2. **创建签名密钥**：
   ```
   axn android generate-keystore
   ```
   交互式引导填写：别名（alias）、密码、有效期、组织信息。生成 `release.jks` 并自动写入 `project.json`。**密钥文件须妥善备份，丢失后无法更新已发布应用。** 引擎会在每次构建前检查密钥文件是否存在，不存在时报错并提示备份路径。已有密钥时跳过此步骤，直接在 `project.json` 中指定路径。

3. **构建发行版**：
   ```
   axn build --platform=android --output=aab   # Google Play
   axn build --platform=android --output=apk   # 直接安装
   ```

构建前自动检测依赖环境，不存在时报错给出官网链接，不自动下载：

```
[✓] JDK 17        /usr/lib/jvm/java-17
[✓] 签名密钥      release.jks（别名: mykey）
[✗] Android NDK   未找到
    请在 Android Studio 中安装 NDK，或前往：
    https://developer.android.com/ndk/downloads
    安装后设置 ANDROID_NDK_HOME 环境变量
[✗] bundletool    未找到
    请前往：https://github.com/google/bundletool/releases
    下载后设置 BUNDLETOOL_PATH 环境变量
```

引擎不自带、不捆绑、不自动下载 JDK / NDK / bundletool，维护成本过高且开发者机器通常已有。

密钥库不存在时自动生成调试密钥（`debug.jks`），适合开发测试，不适合发布。调试密钥构建产物在构建日志中以 `[DEBUG KEY]` 标注。

**Android 资源路径**：构建时 `main/` 目录整体打入 APK 的 `assets/`，路径映射不变，`engine.open_file()` 内部处理平台差异，开发者无感知。

### 预期包体大小参考

| 配置 | 预期大小 |
|------|---------|
| Pygame + `--runtime=auto` | 20–30MB |
| Qt + `--runtime=auto` | 40–55MB |
| Pygame + `--runtime=minimal` + `--assets=remote` | < 10MB |

Ren'Py 默认包体约 60MB（未裁剪 CPython + 完整标准库 + 未裁剪 pygame）。Axn-Plus 从设计阶段将裁剪纳入构建流程，不是事后压缩。

---

## 文本编码

`.apy` 源文件默认编码为 UTF-8（无 BOM）。可在 `project.json` 中调整：

```json
"source_encoding": "utf-8"
```

支持的编码：

| 值 | 说明 |
|----|------|
| `utf-8`（默认） | 推荐，覆盖所有语言 |
| `gb18030` | 简体中文遗留项目 |
| `shift-jis` | 日语遗留项目 |

不单独列 ASCII——ASCII 是 UTF-8 的子集，UTF-8 模式下天然兼容。

`source_encoding` 只作用于 `.apy` 源文件本身的解码，不影响资源文件路径。资源文件路径统一要求 UTF-8，避免 Windows ANSI 路径问题。

引擎内部处理统一转为 UTF-8 字符串，编码信息不向下传递。

---

## 资源系统补充

### 扩展名省略

`.apy` 脚本中引用资源时可省略扩展名：

```apy
show home          # 引擎自动查找 home.png / home.jpg / home.webp 等
play music rain    # 引擎自动查找 rain.ogg / rain.mp3 / rain.wav 等
```

**实现机制**：引擎启动时扫描 `main/` 目录，建立路径索引 `name_without_ext → full_path`。同名不同扩展名时启动报错，要求开发者显式指定扩展名消歧义。

**打包后**：`axn build` 将索引固化为查找表打进包体，运行时零 I/O 查表，与 voice 短路径机制复用同一套。

**`$` 动态路径**：无法在编译期处理，运行时查内存索引，成本可接受。

### 资源缺失策略

**开发期**：启动时输出警告，可忽略，不阻止运行。

**发布包**：通过 `project.json` 配置：

```json
"build": {
    "release_asset_missing": "placeholder"
}
```

| 值 | 行为 |
|----|------|
| `placeholder`（默认） | 图片显示灰色占位，音频静音 |
| `silent` | 完全静默，不渲染 |
| `crash` | 写 crash.log + 显示错误界面 |

不向玩家显示原始文件路径。

### Android 资源访问

Android 构建后资源在 APK 的 `assets/` 目录内，不能用普通文件系统路径访问。`engine.open_file()` 内部处理平台差异：

```python
def open_file(path: str):
    if PLATFORM == "android":
        return _android_asset_open(path)
    elif _current_archive:
        return _archive_open(path)
    else:
        return open(_resolve_path(path), "rb")
```

开发者始终使用 `engine.open_file()`，不直接调用 `open()`——后者在打包后资源被归档时会找不到文件。

---

## 启动器（Axn-Plus Launcher）

Axn-Plus 提供三种交互层，共享同一套业务逻辑，只换前端：

| 启动方式 | 说明 |
|---------|------|
| `axn-plus` | 直接启动 GUI（编译后的 Qt 产物） |
| `axn-plus --tui` | 启动 TUI（基于 `textual`，不依赖 Qt） |
| `axn-plus <command>` | 直接执行 CLI 命令，不启动交互界面 |

GUI 是编译产物，不是运行时检测后的降级，三种方式是独立的启动入口。

### 创建项目流程

```
Step 1: 项目名称
        输入项目名称
        → 自动推导目录名（去除特殊字符）
        → 可手动修改目录名

Step 2: 后端选择
        ○ Pygame（推荐，轻量，适合大多数视觉小说）
        ○ Qt（复杂 UI 需求，包体较大）

Step 3: 分辨率
        ○ 1280×720（默认）
        ○ 1920×1080
        ○ 自定义

Step 4: 主题
        ○ 默认（推荐）
        ○ [更多主题即将推出]  （占位，灰色不可选）

Step 5: 确认创建
        → 显示项目路径预览
        → 创建完成，可选"立即在编辑器中打开"
```

Android 包名、签名密钥等平台配置**不在创建阶段询问**，在构建时或 `project.json` 中配置。创建阶段询问包名时机不对（用户还没写一行代码）。

**不提示"正在编写一个简短的程序"**：这是 Ren'Py 的历史遗留，早期硬件上动态生成胶水代码需要时间。Axn-Plus 的三遍扫描在现代硬件上极快，不需要提示。大型项目扫描超过 1 秒时显示进度条，不显示文字提示。

### Android 构建环境检测

构建前自动检测，缺失时报错给出官网链接，不自动下载，不捆绑工具链。GUI/TUI 中以状态图标列表展示，链接可点击。

---

## 开发者工具补充

### `axn lint`

静态分析工具，集中输出 warning 并支持配置严格度：

```
axn lint                          # 全项目扫描
axn lint scripts/scene.apy        # 单文件
axn lint --strict                 # warning 升级为错误
```

检查项（不限于）：未使用的 label、未定义的跳转目标、未覆盖的 menu 分支、store 变量未在 `flag` 块声明、使用废弃写法（`as` 句柄、`compose=sequence` 等）。

### `axn test --from-state`

从指定 store 状态启动测试，不需要从头点击到目标场景：

```
axn test --from-state tests/states/ch2_angry.json --label chapter2_morning
```

`ch2_angry.json` 是 store 快照（可从开发模式变量浏览器导出），引擎注入此状态后从指定 label 开始执行。

### 引擎内嵌代码编辑器

**不在引擎层实现**。热重载已覆盖主要需求（改外部文件，自动重载）。嵌入式编辑器维护成本高，推迟到 Axn-Editor 集成阶段。

### 开发者工具完整列表

| 工具 | 交付形式 | 说明 |
|------|---------|------|
| 控制台 | 开发模式内置，Shift+\` | Python 表达式执行，变量浏览器 |
| 交互式编导器 | 开发模式内置，Shift+D | 立绘/表情/镜头实时调整，生成代码 |
| label 跳转（传送门） | 控制台内 | `vm.jump_to(label)` |
| 自动重载 | 开发模式默认开启 | `watchdog` + HotReloader |
| 自动测试 | `axn test` | `.test` 脚本 + `--from-state` |
| 翻译提取 | `axn extract-strings` | 生成翻译模板 |
| 翻译检查 | `axn check-strings` | 完整性验证 |
| 静态分析 | `axn lint` | 废弃用法、未定义跳转等 |
| 归档打包 | `axn build --pack` | 生成 `.npa` 归档 |
| 截图 | F12（开发模式） | 保存到 `screenshots/` |

所有开发者工具在发布包（`axn build`）中完全剥离。



---

## 外部命令注册接口

引擎提供统一的指令扩展接口，允许开发者注册自定义动词，与内置指令使用完全相同的语法规则（位置参数、具名参数、子命令）。

### 注册方式

在 `startup (before):` 块或引擎初始化钩子中调用：

```python
from axn_plus.apy.stdlib import register_command

@register_command(
    verb="notify_discord",
    subcommands=["send", "update"],          # 可选，不需要子命令时省略
    positional=["message"],                  # 位置参数名列表，按顺序
    named={"channel": str, "color": str}     # 具名参数名 → 类型
)
def handle_notify_discord(verb, subcmd, args, named, store):
    # verb:   "notify_discord"
    # subcmd: "send" 或 "update"，无子命令时为 None
    # args:   位置参数值列表
    # named:  具名参数 dict
    # store:  当前 Store 对象（只读访问，不允许在此修改）
    discord_webhook(named.get("channel"), args[0])
```

注册后 `.apy` 脚本中直接使用：

```apy
notify_discord send "任务完成" (channel="general", color=#00ff00)
notify_discord update "进度更新" (channel="log")
```

### 注册规则

- 注册必须在引擎启动三遍扫描之前完成（`startup (before):` 块或 `axn_init` 钩子），否则 Parser 第一遍扫描时无法识别 verb，报 `AxnParseError`
- verb 名不能与内置指令冲突，注册时检查，冲突报 `AxnExtensionError`
- 子命令集合由注册时声明，不可运行时动态增减
- 同一 verb 不允许重复注册，第二次注册报 `AxnExtensionError`；需要覆盖时显式传 `override=True`
- GUI 编辑器对外部注册命令降级为代码节点，归属关系保留；若扩展同时提供 `gui_schema`（节点描述 dict），编辑器可渲染为专用积木块

### 与 `axn::` 扩展库的关系

`axn::` 标准库内部使用同一套注册机制，`register_command` 是对外开放的同一接口。内置指令分发表、外部注册指令分发表在引擎启动时合并为统一的全局分发表，Parser 和 VM 无需区分来源。

---

## `options_window.apy` 最小配置规范

一个没有复杂功能的普通视觉小说，`options_window.apy` 最少需要包含以下内容才能正常运行：

```apy
# ── 引擎运行时配置 ──────────────────────────────────────────
engine:
    window:
        hide_behaviour = "overlay"   # 对话框隐藏时文字处理策略

    history:
        max_entries = 200

    autosave:
        enabled = true
        trigger = "checkpoint"

# ── 玩家偏好默认值 ───────────────────────────────────────────
preferences:
    # 文字
    text_speed        = 1.0
    skip_mode         = "seen"
    auto_forward_time = 2.0
    auto_delay        = 0.3

    # 音量
    music_volume      = 0.8
    sound_volume      = 0.8
    voice_volume      = 0.9

    # 显示
    fullscreen        = false

# ── 必须存在的 UI 模板 ────────────────────────────────────────
# 对话框（最简版，只有文字区和名字区）
gui DialogueBox:
    background "ui/box.png"
    padding (20, 12)
    vstack:
        slot name_label
        slot dialogue_text

# 标题画面（最简版，只有开始和退出）
screen title_screen:
    background "ui/title_bg.png"
    vstack gap=16:
        button "开始游戏" on_click: jump start
        button "退出"     on_click: exit
```

**不写就不存在**：画廊、成就、音频通道 `ui=` 配置、自定义偏好项等，不声明时引擎不报错，对应功能直接不可用。

**`options_window.apy` 与 `project.json` 的职责边界**：运行时行为写 `options_window.apy`，构建/打包配置写 `project.json`，两者不互换。

---

## 最小参考项目

引擎开发阶段应同步维护一个最小参考项目，用于验收每个子系统、提供测试环境和文档示例。不要等到引擎"完成"后再补 UI，两者同步推进，问题当场暴露。

**参考项目结构：**

```
axn_sample/
    flow.apy                  # 跑通对话、菜单、跳转、存档的最小流程
    options_window.apy        # 最小配置（见上节）
    main/
        scripts/
            ch1.apy           # 第一章：包含对话、菜单、checkpoint
        image/
            gui/
                box.png       # 占位图，1×1 像素灰色
                title_bg.png
        audio/
            bgm_test.ogg      # 5秒循环音频
```

**参考项目的三个用途：**

1. **子系统验收**：每实现一个子系统（对话渲染、存档、菜单等）就在参考项目里接一下，立即发现接口问题
2. **自动化测试环境**：`axn test` 的执行载体，测试脚本直接跑参考项目
3. **文档代码示例**：文档里所有 `.apy` 示例来自参考项目的真实代码，不是手写伪代码

---

## 引擎核心扩展（计划集成）

以下功能在功能性上必须由引擎核心提供，不适合外包给开发者自己实现。

### 存档槽元数据查询 API

开发者编写存档界面时必然需要查询所有槽位的信息，无此接口则存档界面无法实现：

```python
engine.save_slots()                # 返回所有槽位的元数据列表
engine.save_slot_info(slot)        # 单个槽位：时间戳、章节名、缩略图路径、游戏时长
engine.save_exists(slot)           # bool
engine.delete_save(slot)           # 删除存档槽
```

### 游戏内计时器

视觉小说的限时选择、计时解谜、游戏总时长统计都依赖此接口。不内置则每个开发者自己用 `store` + `$` 拼，且读档后时间状态会乱：

```python
engine.timer.start("battle_timer")
engine.timer.elapsed("battle_timer")    # 返回已过秒数（float）
engine.timer.pause("battle_timer")
engine.timer.resume("battle_timer")
engine.timer.stop("battle_timer")
engine.timer.reset("battle_timer")
```

计时器状态自动纳入存档快照，读档后恢复，无需开发者手动处理。

### `persistent` 版本迁移钩子

`store` 已有 `Saveable.__load__(data, version)` 处理跨版本兼容，`persistent` 同样会遇到版本升级问题（旧玩家的解锁记录结构可能随游戏更新而变化）：

```python
# options_window.apy 的 startup (before): 块中注册
engine.persistent.register_migration(
    from_version = 1,
    to_version   = 2,
    handler      = migrate_persistent_v1_to_v2
)

def migrate_persistent_v1_to_v2(data: dict) -> dict:
    # 旧版本 achievements 是 list，新版本改为 dict
    if isinstance(data.get("achievements"), list):
        data["achievements"] = {k: True for k in data["achievements"]}
    return data
```

迁移按版本号顺序链式执行，跨多个版本时自动串联。`persistent` 文件写入版本号，读取时检查并按需执行迁移链。

---

## 可选插件（外部 `.apy` 模块）

以下功能封装为外部脚本，开发者按需放入 `main/axn/` 目录自行选用。

### 存档导出（`axn::save_export`）

将存档槽导出为文件或字符串，用于备份、跨设备迁移、Bug 复现等场景：

```apy
# 导出为文件
$ engine.save_export.to_file(slot="slot1", path="backups/save_slot1.axnsave")

# 导出为 base64 字符串（可粘贴到文本框分享）
$ export_str = engine.save_export.to_string(slot="slot1")
show screen export_dialog(data=export_str)

# 导入
$ engine.save_export.from_file("backups/save_slot1.axnsave", slot="slot1")
$ engine.save_export.from_string(export_str, slot="slot2")
```

导出格式：存档内容（pickle）+ 版本号 + SHA-256 校验和，整体 base64 编码。导入时校验完整性，版本不匹配时提示但允许继续（兼容性由开发者的 `Saveable.__load__` 处理）。

### 本地化辅助（`axn::i18n`）

数字、日期、复数格式随语言变化，不只是翻译字符串的问题：

```python
axn_i18n.format_number(1234)       # 按当前语言格式化数字
axn_i18n.format_date(timestamp)    # 按当前语言格式化日期
axn_i18n.plural("day", count)      # 英文复数处理（"1 day" / "2 days"）
```

### 键鼠/手柄按键提示图标（`axn::input_icons`）

"按 A 键继续" 类提示，PC 上显示键盘图标，手柄接入后自动切换为对应品牌的按键图标：

```apy
image axn_input_icons.get("advance")    # 自动根据当前输入设备返回对应图标
```

包含 Xbox / PlayStation / Switch 三套按键图标资源包，可替换。

### 气泡自动避让（`axn::bubble_layout`）

`together` 块多角色气泡在立绘位置动态变化时的自动避让，基于简单贪心算法：气泡检测重叠后沿垂直方向推开，无需手动调整 `offset`。

### 数值系统（`axn::stat`）

带上下限、变化回调、历史记录的通用数值类，避免每个项目重复实现边界检查：

```python
@saveable
class Stat:
    def __init__(self, initial: float, min: float = 0, max: float = 100):
        ...

    def modify(self, delta: float, reason: str = "") -> float:
        # 返回实际变化量（受边界限制后）
        ...

    def history(self) -> list[dict]:
        # 返回变化历史：[{"delta": 5, "reason": "gift", "timestamp": ...}]
        ...
```

```apy
$ relationship_autumn = Stat(50, min=0, max=100)
$ relationship_autumn.modify(+10, reason="gave_gift")
```

---

## 脚本语法扩展

### `after` — 非阻塞延迟执行

```apy
after 2.0:
    play sound "sfx/thunder.ogg"
    camera shake 8 0.3
```

`after` 块内的指令在指定秒数后触发，不阻塞当前执行流。对话在进行的同时，背景音效在 2 秒后自动触发，两者互不干扰。

与 `parallel` 的区别：`parallel` 用于需要精细 track 控制的场景；`after` 是轻量的单次延迟触发，不产生 track，GUI 解析为带延迟标签的独立指令节点。

`after` 块内只允许引擎指令，不允许对话行和 Python 块（保证 GUI 完整可解析）。

---

### `repeat` 块 — 固定次数重复执行

```apy
repeat 3:
    camera shake 5 0.2
    wait 0.3
```

固定次数重复，语义明确，不需要声明 `rollback=`（块内不允许对话行，回滚语义无歧义）。与 `while` / `for` 的区别：`repeat N` 是编译期已知次数的演出重复，不是逻辑循环。`animation` 块内同样支持 `repeat N` 结构，两者共享语义。

---

### `defer` — label 退出时执行

```apy
label battle_scene:
    defer:
        stop music 0.5
        hide gui battle_hud
        $ store["in_battle"] = False

    show gui battle_hud
    play music "bgm/battle.ogg"
    $ store["in_battle"] = True

    menu:
        "逃跑" -> escape_route
        "继续战斗" -> battle_continue
```

无论通过 `jump`、`return`、`unwind` 哪种方式离开 label，`defer` 块都会在退出时执行。对 BGM 切换、UI 清理、状态重置特别有用，避免清理逻辑散落在各个出口。

规则：
- `defer` 块只允许出现在 `label` 块内，不允许嵌套 `defer`
- 块内只允许引擎指令和 Python 块，不允许对话行（避免退出时产生等待点）
- 多个 `defer` 块按声明顺序逆序执行（后声明的先执行，类似栈展开）
- `unwind` 触发时，经过的每个 label 的 `defer` 块依次执行

GUI 处理：label 节点底部显示 defer 块，标注"离开时执行"，视觉上与正常执行流区分。

---

### `once` 块 — 生命周期内只执行一次

```apy
label daily_event:
    once per_session:            # 本次游玩只执行一次（存入 store 临时标记）
        autumn: "今天的天气真不错。"

    once per_playthrough:        # 整个周目只执行一次（存入 store）
        $ unlock_scene("first_morning")

    once ever:                   # 永久只执行一次（存入 persistent）
        $ persistent["seen_tutorial"] = True
        call tutorial_label
```

三种生命周期对应三个存储层级：

| 生命周期 | 存储位置 | 重置时机 |
|---------|---------|---------|
| `per_session` | 内存临时标记 | 游戏退出时 |
| `per_playthrough` | `store` | 新游戏时 |
| `ever` | `persistent` | 永不重置 |

引擎自动为每个 `once` 块生成唯一标识（`文件名 + label名 + 行偏移` 哈希），开发者无需手动命名。`once` 是高频模式的语法糖，消除手动写 flag 检查的样板代码。

GUI 处理：对应带生命周期标签的条件节点，标签下拉选择三种生命周期。

---

### `rollback fence` — 精确回滚边界

```apy
label shop_scene:
    rollback fence:
        $ store["gold"] -= item.price
        $ store["inventory"].append(item)
        autumn: "购买成功。"
```

在特定操作后设置回滚边界，玩家无法回滚到此点之前。比 `rollback=none`（禁止整个 label 回滚）更精细——只封锁消费类操作（花钱、存档覆盖确认、不可逆操作），其余对话仍可正常回滚。

`rollback fence` 块执行完毕后，该块成为新的回滚起点。块内对话行正常产生回滚检查点，只是无法越过 fence 回到更早的状态。

---

### `unwind` — 展开调用栈

```apy
unwind                        # 清空 call 栈，回到顶层继续执行
unwind to morning_scene       # 展开到指定 label（必须在当前栈上，否则运行时报错）
```

`return` 只退出当前 label，回到直接调用方；`unwind` 展开整个栈，适合从任意嵌套深度强制返回到起点（如全局错误恢复、章节强制结束）。

`unwind to label` 的目标必须在当前 call 栈上，否则运行时抛 `AxnRuntimeError`：

```
AxnRuntimeError: 'unwind to morning_scene': target label is not on the current call stack.
  Current call stack: start → chapter1_battle → boss_phase_2
  'morning_scene' was not found in the call chain.
  → scene.apy, line 8
Hint: Use 'jump morning_scene' to unconditionally jump, or ensure the label was 'call'ed before 'unwind to'.
```

---

### `snapshot` / `restore` — 手动状态快照

```apy
snapshot "before_choice" keys=[relationship, day, flag_agreed]

$ relationship["autumn"] += 10
$ flag_agreed = True

restore "before_choice"
```

脚本层轻量快照，不写磁盘，存在内存里。用于叙事机制（时间回溯、蝴蝶效应预览、"如果当初"场景）。

规则：
- 快照只保存 `keys=` 指定的顶层 store 变量，省略 `keys=` 时快照全部 store
- 快照在 label 退出时自动清除，除非显式 `persist snapshot "name"`
- `restore` 目标不存在时运行时警告，不报错，静默跳过
- 与 `with store` 原子块不同：`snapshot/restore` 是手动控制的跨时间线状态管理，不提供自动回滚

---

### `define position` — 具名位置

```apy
define position stage_left:
    x 0.2
    y 0.85
    scale 0.9
    z_order 10

define position stage_right:
    x 0.8
    y 0.85
    scale 0.9
    z_order 10

define position center_close:
    x 0.5
    y 0.9
    scale 1.1
    z_order 20

show autumn stage_left
show sophia stage_right
show autumn center_close 0.3 (enter=slidein)
```

`define position` 进符号表，`show` 指令的位置参数支持具名位置关键字。项目级统一管理位置，改一处全部生效。

与内置位置关键字（`left`、`center`、`right`）的关系：内置关键字是默认位置的别名，`define position` 可以覆盖内置关键字的默认值，也可以定义新的位置名。

GUI 处理：位置下拉列表中同时显示内置关键字和用户定义的具名位置，具名位置显示坐标预览。

---

### `define group` — 角色组

```apy
define group classroom:
    members [autumn, sophia, kenji]

define group main_cast:
    members [autumn, sophia]

show group classroom left (spread=true)    # 均匀分布到预设位置
hide group classroom 0.5
expression group classroom sad
expression group main_cast happy
```

`show group` 编译期展开为多条 `show` 指令，`spread=true` 时引擎自动计算均匀分布位置。`expression group` 同时修改组内所有角色的表情状态。

`define group` 只是成员列表的编译期简写，不创建新的运行时对象，组内成员仍然独立可操作。

---

### `define sound_group` — 音效组随机播放

```apy
define sound_group footsteps:
    "sfx/step1.ogg"
    "sfx/step2.ogg"
    "sfx/step3.ogg"
    mode random    # random（默认）/ cycle（循环轮播）

define sound_group page_turn:
    "sfx/page1.ogg"
    "sfx/page2.ogg"
    mode cycle

play sound footsteps    # 每次随机取一个
play sound page_turn    # 依次轮播
```

重复音效（脚步、翻书、键盘、攻击）每次都播同一个音频会很明显，随机/轮播选取让重复播放更自然。`define sound_group` 进符号表，`play sound` 指令直接引用组名。

---

### `label alias` — 标签别名

```apy
label morning_scene alias ["ch1_morning", "morning"]:
    autumn: "早上好。"
```

重构时重命名 label 会破坏存档（存档记录的是 label 名）。`alias` 声明兼容别名，引擎在跳转和存档恢复时同时接受别名。

规则：
- 别名只用于向后兼容，不推荐在新代码中引用别名跳转
- `axn lint` 检测别名的使用，提示迁移到新名字
- 别名不进全局符号表（不会在 GUI 的 label 列表里出现），只在跳转解析和存档恢复时参与匹配
- 跨文件时别名同样有效

---

### `computed` — 派生变量

```apy
flag:
    relationship_autumn: int = 50
    relationship_sophia: int = 30
    flag_met_autumn: bool = False

computed:
    total_affection = relationship_autumn + relationship_sophia
    autumn_route_available = relationship_autumn >= 70 and flag_met_autumn
    dominant_route = "autumn" if relationship_autumn > relationship_sophia else "sophia"
```

`computed` 变量只读，每次访问时重新计算，不存入存档（可从基础变量推导）。消除手动维护派生状态的错误。

规则：
- 只允许出现在文件顶层，与 `flag` 块平级
- 右值只允许引用 `flag` 声明的变量和简单表达式（算术、比较、三元）
- 不允许函数调用（保证每次求值无副作用）
- `axn lint` 检测循环依赖（`computed` 变量相互引用）
- GUI 变量面板显示计算公式，标注"派生，不存档"

---

### `flag` 扩展声明

```apy
flag:
    relationship: int = 50
        clamp 0 100                      # 赋值时自动 clamp，不需要手动写边界检查
        on_change: relationship_changed  # 变化时调用（只在通过 set/store 赋值时触发，不轮询）

    player_name: str = ""
        validate: lambda v: len(v) <= 12    # 不满足时抛 AxnValueError
        transform: lambda v: v.strip()      # 存入前转换

    volume: float = 0.8
        clamp 0.0 1.0
```

`clamp` 是最高频的需求，专门的关键字比 lambda 更清晰。`on_change` 范围限定在明确的赋值操作（`set`、`$`、`with store`），不做每帧轮询，实现成本可控。

`validate` 和 `transform` 接受 Python lambda，降级为代码节点，归属关系保留。

`clamp` 在 release 模式下内联为边界检查代码，零运行时开销。

---

### `flag namespace` — 变量命名空间

```apy
flag namespace autumn:
    relationship: int = 50
    first_met: bool = False
    route: str = ""

flag namespace sophia:
    relationship: int = 30
    first_met: bool = False

# 访问
$ autumn.relationship += 5
$ sophia.first_met = True
if autumn.relationship >= 70:
    jump autumn_route
```

不强制使用，但大型项目里 flag 变量命名前缀泛滥时（`autumn_relationship`、`sophia_relationship`）命名空间更清晰。

命名空间在 store 内部存为点分隔的 key（`autumn.relationship`），不是嵌套 dict，序列化和访问语义不变。存档兼容性：读档时按完整 key 名恢复，命名空间重构不影响已有存档。

---

### `store` 查询方法

```apy
$ keys = store.query("inventory", where=lambda i: i["type"] == "key")
$ sorted_items = store.query("inventory", order_by="rarity", limit=10)
$ total = store.count("inventory", where=lambda i: i["equipped"])
```

`Store` 类方法扩展，不引入新脚本语法。视觉小说里对背包、任务列表、解锁内容做查询很频繁，消除重复的 list comprehension 样板。

---

### `store` 变量历史追踪

```apy
flag:
    relationship: int = 50
        track_history max=20         # 记录最近 20 次变化

# 查询
$ history = store.history("relationship")
# [{"value": 50, "delta": 0, "timestamp": ...},
#  {"value": 55, "delta": +5, "reason": "gave_gift", ...}]
```

`track_history` 是 `flag` 的可选修饰符，不声明则不追踪（零额外开销）。历史记录存入 store，随存档一起保存。用于蝴蝶效应展示、数值变化回顾等叙事场景。

---

### `beat` — 节拍同步

```apy
play music "bgm/action.ogg" (bpm=128, beats_per_bar=4)

wait beat            # 等到下一个节拍
wait beat 4          # 等 4 个节拍
wait bar             # 等下一小节开始

show autumn (transform=jump_in) at beat   # 在下一个节拍触发 show
camera shake 5 0.3 at beat 2              # 2 个节拍后触发
```

音乐驱动的演出。现在只能用 `wait N.0` 手动算时间，BPM 变化时要重算所有等待时间。`beat` 让演出和音乐节奏自动对齐，改 BPM 不用改脚本。

实现：`play music` 记录 BPM 和开始时间戳，`wait beat` 计算下一节拍的绝对时间戳等待。BPM 未声明时 `wait beat` 运行时警告并降级为 `wait 0`。

`at beat` 是 `show` / `camera` 等指令的时机修饰符，等效于先 `wait beat` 再执行指令，但不阻塞执行流。

---

### `camera path` — 镜头路径动画

```apy
define camera_path sweep_panorama:
    keyframe 0.0: pos (0.1, 0.5) zoom 1.0
    keyframe 0.5: pos (0.5, 0.5) zoom 1.15
    keyframe 1.0: pos (0.9, 0.5) zoom 1.0
    easing ease_in_out

define camera_path zoom_focus(target_pos=(0.5, 0.5)):
    keyframe 0.0: pos (0.5, 0.5) zoom 1.0
    keyframe 1.0: pos target_pos  zoom 1.8
    easing ease_out

camera path sweep_panorama 4.0              # 执行路径动画，4 秒
camera path sweep_panorama 4.0 (handle=cam_anim)
wait for cam_anim

camera path zoom_focus(target_pos=(0.3, 0.6)) 2.0
```

复用 `transform` 的 keyframe 结构，概念统一，实现共享大部分代码。`pos` 为归一化坐标（0.0–1.0）。`camera path` 与 `camera follow` 可共存，`path` 在 `follow` 的基础上叠加偏移。

`define camera_path` 支持参数化（与 `animation` 参数化规则一致），GUI 解析为镜头路径积木块，keyframe 在时间轴编辑器中可视化编辑。

---

### `signal` / `on signal` — 脚本层事件总线

```apy
# 发送信号
signal "autumn_mood_changed" (mood="angry", intensity=0.8)

# 任意位置监听
on signal "autumn_mood_changed": (mood, intensity):
    expression autumn (face=angry)
    if intensity > 0.5:
        camera shake 3 0.2
```

与 UI 层 `emit channel=` 的区别：`signal` 是脚本执行流层面的事件（label 之间、animation 与脚本之间），`emit` 是控件树层面的事件。两者共用同一套底层事件总线，但语义层分开。

规则：
- `on signal` 只允许出现在文件顶层，与 `on enter` / `on key` 平级
- 信号在当前 tick 同步分发，不延迟到下一帧
- `parallel` track 内可以发送和接收信号
- `animation` 块内只允许发送信号（`signal`），不允许 `on signal` 监听
- `axn:` 前缀为引擎标准库保留，开发者不应使用

GUI 处理：独立事件钩子面板展示，与 `on enter` / `on key` 并列；发送方和接收方以连线标注关联关系。

---

### `#region` / `#endregion` — 代码折叠

```apy
#region 第一章：清晨

label morning_scene:
    autumn: "早上好。"

label breakfast_scene:
    sophia: "今天吃什么？"

#endregion

#region 第一章：下午

label afternoon_scene:
    ...

#endregion
```

纯注释，不影响解析和执行。编辑器和 IDE 插件（VSCode 扩展）支持折叠区域，大型脚本文件结构管理用。引擎在 `#region` / `#endregion` 不匹配时输出警告（不报错）。


---

## 渲染层扩展

### `mask` — 遮罩合成

```apy
show autumn (mask="assets/masks/spotlight.png")
# autumn 只在 mask 不透明区域内显示，实现聚光灯、窗帘遮挡效果

show bg_room (mask=store["reveal_mask"], mask_mode=reveal)
# mask 随时间变化，实现擦除/显现效果

show effect_overlay (mask="masks/vignette.png", mask_mode=invert)
```

`mask` 与 `color_matrix` 同级，作为显示对象的独立后处理属性，可与 `transform` 同时存在。

| `mask_mode` | 行为 |
|------------|------|
| `alpha`（默认） | mask 的 alpha 通道控制显示区域 |
| `reveal` | mask 亮度控制显示进度（擦除效果） |
| `invert` | 反向遮罩，mask 不透明处隐藏 |

`mask` 动画化：

```apy
show autumn (mask="masks/spotlight.png", mask_transition=0.5)
# 0.5 秒内从当前 mask 过渡到新 mask（线性插值）
```

Round-Trip：`mask=` 在 `show` 积木块中显示为独立遮罩字段，`mask_mode` 下拉选择，`mask_transition` 时间字段可选。

---

### `render_texture` — 离屏渲染

```apy
render_texture as scene_tex:
    show bg_room
    show autumn center
    show sophia right

# 对合成结果整体做处理
show scene_tex (color_matrix=grayscale, transform=shake_x)

# 复用纹理
show scene_tex left
show scene_tex right (color_matrix=TintMatrix(#8888ff, 0.3))
```

将一组对象渲染到离屏纹理，再对纹理整体做处理。比 `layer transform` + `layer color_matrix` 更灵活——可以跨层组合，可以复用纹理，可以做后处理链。

`render_texture` 块内只允许 `show` / `hide` / `expression` 等显示控制指令，不允许对话行、Python 块、跳转指令。纹理在 `render_texture` 块执行时一次性合成，不随内容实时更新——如需实时跟踪显示状态，改用 `layer transform`。

---

### `show` 的 `blend` 模式

```apy
show effect_overlay (blend=screen)      # 滤色（发光、光晕）
show effect_overlay (blend=multiply)    # 正片叠底（阴影、压暗）
show effect_overlay (blend=add)         # 相加（强发光）
show effect_overlay (blend=overlay)     # 叠加（对比度增强）
show effect_overlay (blend=screen, alpha=0.7)   # 混合模式 + 透明度
```

内置混合模式：

| 值 | 用途 |
|----|------|
| `normal`（默认） | 标准 alpha 混合 |
| `screen` | 滤色，发光、光晕特效 |
| `multiply` | 正片叠底，阴影、染色 |
| `add` | 相加，强发光、火焰 |
| `overlay` | 叠加，对比度增强 |
| `darken` | 取暗 |
| `lighten` | 取亮 |
| `color_dodge` | 颜色减淡 |
| `color_burn` | 颜色加深 |

Pygame 后端通过 `special_flags` 实现，Qt 后端通过 `QPainter::CompositionMode` 实现，引擎层统一封装。

---

### `define particle` — 粒子系统

```apy
define particle snow:
    texture "fx/snowflake.png"
    count 200
    spawn_area (fill, top)                   # 从顶部整个宽度生成
    velocity (random(-20, 20), random(30, 80))
    lifetime random(3.0, 6.0)
    fade_in 0.5
    fade_out 1.0
    rotation random(0, 360)
    rotation_speed random(-30, 30)
    scale random(0.5, 1.5)
    wind (5, 0)                              # 全局风力偏移

define particle sakura extends snow:
    texture "fx/petal.png"
    velocity (random(-30, -10), random(20, 50))
    wind (-8, 0)
    scale random(0.3, 0.8)

define particle magic_sparkle:
    texture "fx/sparkle.png"
    count 50
    spawn_area (center, 0.5, 0.5)            # 从指定位置生成
    spawn_radius 30                           # 生成半径（像素）
    velocity (random(-40, 40), random(-60, -20))
    gravity (0, 20)                           # 重力
    lifetime random(0.5, 1.5)
    fade_in 0.1
    fade_out 0.5
    scale random(0.5, 1.0)
    color_cycle ["#ffffff", "#ffdd88", "#ff8800"]   # 颜色随时间循环
```

`define particle` 进符号表，使用方式与 `show` 统一：

```apy
show particle snow (layer=effect)
show particle snow (layer=effect, alias=snow_1)
hide particle snow 2.0
expression particle snow (count=50)    # 运行时修改参数
```

粒子参数：

| 参数 | 说明 |
|------|------|
| `texture` | 粒子贴图路径 |
| `count` | 同时存在的最大粒子数 |
| `spawn_area` | 生成区域：`(fill, top/bottom/left/right)` / `(center, x, y)` / `(rect, x, y, w, h)` |
| `spawn_radius` | 生成半径（配合 `center` 使用） |
| `spawn_rate` | 每秒生成粒子数，省略时按 count 和 lifetime 自动计算 |
| `velocity` | 初速度 `(x, y)`，支持 `random(min, max)` |
| `gravity` | 重力加速度 `(x, y)` |
| `wind` | 风力（对所有粒子施加的额外速度） |
| `lifetime` | 粒子生命周期（秒），支持 `random(min, max)` |
| `fade_in` | 淡入时间（秒） |
| `fade_out` | 淡出时间（秒） |
| `rotation` | 初始旋转角度，支持 `random(min, max)` |
| `rotation_speed` | 旋转速度（度/秒），支持 `random(min, max)` |
| `scale` | 缩放，支持 `random(min, max)` |
| `color_cycle` | 颜色列表，粒子在生命周期内循环过渡 |

`define particle` 支持 `extends` 继承，子类覆盖父类同名参数。GUI 解析为粒子系统配置积木块，参数列表完整可编辑，编辑器内提供实时粒子预览。


---

## 音频系统扩展

### `music_transition` — BGM 切换策略

```apy
# 现有写法（仍然有效）
stop music 1.0
play music "bgm/new.ogg" 0.8 1.0

# 扩展写法：在 play 上声明切换策略
play music "bgm/new.ogg" (transition=crossfade 1.5)
play music "bgm/new.ogg" (transition=stinger "bgm/sting.ogg")
play music "bgm/new.ogg" (transition=wait_bar)
play music "bgm/new.ogg" (transition=cut)
```

| transition 类型 | 行为 |
|----------------|------|
| `cut`（默认） | 立即切换 |
| `crossfade N` | 旧 BGM 淡出同时新 BGM 淡入，持续 N 秒 |
| `stinger "path"` | 先播过渡音效，结束后接新 BGM |
| `wait_bar` | 等当前 BGM 播完当前小节再切换（需要 BPM 信息） |

`stinger` 走独立的内部通道，不影响 music 通道的队列状态。

---

### `audio_bus` — 音频总线

```apy
# options_window.apy
define audio_bus master:
    volume 1.0

define audio_bus music_bus:
    parent master
    volume 0.8
    effects [reverb(room=0.1, damp=0.5)]    # 总线级效果

define audio_bus sfx_bus:
    parent master
    volume 1.0

define audio_bus voice_bus:
    parent master
    volume 1.0
    effects [compressor(threshold=-6, ratio=3)]   # 人声压缩

engine:
    audio:
        channel_bus:
            music   = music_bus
            ambient = music_bus
            sound   = sfx_bus
            voice   = voice_bus
```

总线层级结构：`master` 为根总线，调整 `master.volume` 影响所有音频输出。`music_bus.volume` 同时影响 music 和 ambient 两个通道，符合游戏音频的实际工作流。

脚本层访问：

```apy
$ engine.audio_bus("music_bus").volume = 0.5
$ engine.audio_bus("master").mute = True
```

---

### `audio_snapshot` — 音频状态快照

```apy
audio_snapshot save "before_cutscene"

play music "bgm/boss.ogg"
filter music (lowpass(cutoff=800)) 0.5

audio_snapshot restore "before_cutscene" 1.0
# 1.0 秒内平滑过渡到恢复的状态（音量、滤波器参数线性插值）
```

保存和恢复所有通道的音频状态（当前播放文件、进度、音量、滤波器参数）。过场动画、模态框弹出/关闭时的音频管理，避免手动记录和恢复。

快照存在内存里，不写磁盘，label 退出时自动清除（与 `snapshot` 规则一致）。

---

### 通道 UI 可见性扩展

```apy
# options_window.apy
engine:
    audio:
        channels:
            music:      ui=true
            sound:      ui=true
            voice:      ui=true
            ambient:    ui=locked    # 显示但不可调，只能脚本控制
            bg_layer:   ui=false     # 完全不在设置界面显示（自定义通道默认值）
```

自定义通道在 `options_window.apy` 中声明后自动纳入存档管理：

```apy
engine:
    audio:
        custom_channels:
            bg_layer:
                default_volume = 0.6
                loop           = true
                bus            = music_bus
```


---

## 控件状态系统（完整设计）

### 完整状态机模型

控件状态机覆盖以下状态，视觉（`style`）和行为（事件钩子）统一描述：

```
idle
  ↓ mouse_enter / focus_gained
hovered（可与 focused 叠加）
  ↓ mouse_down / key_down（space / return）
pressed（瞬时）
  ↓ mouse_up on self（在控件内松开）
  → clicked → 回到 hovered
  ↓ mouse_up off self（拖出去松开）
  → cancelled → 回到 idle
  ↓ 持续按住超过 threshold
active（长按持续态）
  ↓ mouse_up
  → long_clicked → 回到 hovered

selected（静态，由逻辑维护，可与任意状态叠加）
disabled（排他，覆盖所有其他输入响应；视觉上可与 hovered 叠加用于光标提示）
```

**`pressed` / `active` / `selected` 的语义区别：**
- `pressed`：鼠标按下的瞬时态，松开即消失
- `active`：按住不松的持续态，适合长按类操作
- `selected`：开发者显式维护的静态选中态，与鼠标操作无关

### 完整状态声明语法

```apy
gui option_button(label):

    # ── 视觉状态 ──────────────────────────────────────────
    style:
        background #444444
        color #ffffff

        # 单态
        hovered:
            background #555555
            color #ffdd88

        pressed:
            background #3a3a3a
            scale 0.97                    # 轻微缩小，按压感

        active:                           # 长按持续态
            background #1a6622
            transform pulse

        selected:
            background #226622
            border (2, #44ff88)

        focused:
            border (2, #aaaaff)
            overlay #ffffff11

        disabled:
            alpha 0.4
            overlay #00000033

        # 组合态（优先级自动高于单态）
        hovered + selected:
            background #2a8833
            border (2, #66ffaa)

        focused + disabled:
            border (2, #666666)

        hovered + disabled:
            cursor no_drop                # 禁止光标

        # 状态过渡动画
        transition all 0.1 ease_out       # 所有属性变化的默认过渡
        pressed -> hovered:
            transition scale 0.15 bounce  # 定向过渡：松开时有弹性

    # ── 行为事件 ──────────────────────────────────────────
    on_hover_enter:
        play sound "sfx/ui_hover.ogg" 0.2

    on_hover_exit:
        pass

    on_press:
        play sound "sfx/ui_press.ogg" 0.3
        $ engine.sdl2.rumble(0.05, 0.1, 30)

    on_click:                             # press + release on self（推荐写法）
        emit "option_selected" label

    on_cancel:                            # press 后拖出去松开
        pass

    on_active (interval=0.15):            # 长按持续触发，interval 控制频率
        emit "option_held" label

    on_long_click (threshold=0.6):        # 长按后松开
        emit "option_long_selected" label

    on_double_click (interval=0.25):
        emit "option_double_selected" label

    on_focus:
        play sound "sfx/ui_focus.ogg" 0.15

    on_blur:
        pass

    on_key_press (key="return"):          # 焦点在此控件上时的按键响应
        emit "option_selected" label

    on_key_press (key="space"):
        emit "option_selected" label

    on_right_click:
        show screen context_menu (target=label)

    on_middle_click:
        pass

    # ── 状态变化钩子 ──────────────────────────────────────
    on_state_enter (state="selected"):
        play sound "sfx/select.ogg"

    on_state_exit (state="selected"):
        pass

    on_disabled:
        pass

    on_enabled:
        pass

    text label (anchor=center)
```

### 组合态优先级规则

```apy
style:
    background #444444          # 基础（优先级 0）
    hovered:   background #555555   # 单态（优先级 1）
    selected:  background #226622   # 单态（优先级 1，后声明优先）
    hovered + selected:             # 组合态（优先级 2，自动高于所有涉及单态）
        background #2a8833
```

不声明组合态时，回退到后声明单态优先的规则（与现有规则一致）。三个及以上状态的组合（如 `hovered + selected + focused`）同样支持，优先级高于任意二态组合。

### 状态过渡动画

```apy
style:
    transition all 0.1 ease_out       # 全局默认：所有属性 0.1 秒过渡

    hovered:
        background #555555
        transition background 0.08 ease_out   # 单属性覆盖全局

    pressed:
        scale 0.97
        transition scale 0.05 ease_in         # 按下要快

    pressed -> hovered:                       # 定向过渡：从 pressed 到 hovered
        transition scale 0.15 bounce          # 松开有弹性，反向不适用

    hovered -> idle:                          # 鼠标离开时慢慢恢复
        transition background 0.2 ease_out
```

`定向过渡（A -> B）`只在从状态 A 切换到状态 B 时生效，优先级高于 `transition all`。

支持过渡的属性：`background`（颜色插值）、`scale`、`alpha`、`border`（宽度+颜色）、`overlay`（透明度）、`color`。不支持过渡的属性（`size`、`font`）直接切换，不报错。

### `bind selected` — 状态绑定表达式

```apy
gui tab_button(label, tab_id):
    bind selected = store["active_tab"] == tab_id

    style:
        background #333333
        selected: background #226622
        hovered + selected: background #2a8833

    on_click:
        $ store["active_tab"] = tab_id

    text label
```

`bind selected =` 让 `selected` 状态由外部表达式驱动，不需要手动维护。表达式限制为简单比较（与 `when` 条件一致），GUI 完整可解析。

### 拖拽事件完整设计

```apy
gui inventory_item(item):
    draggable (data=item):
        on_drag_start:
            play sound "sfx/pickup.ogg"
            $ store["dragging_item"] = item

        on_drag (pos):                    # 拖拽中每帧，pos 为当前坐标
            pass

        on_drag_end:                      # 拖拽结束（无论是否落在目标）
            $ store["dragging_item"] = None

        on_drag_cancel:                   # 被取消（Escape、失焦）
            play sound "sfx/cancel.ogg"
            $ store["dragging_item"] = None

gui equipment_slot(slot_id):
    droptarget (accept_type="equipment"):
        on_drag_enter (data):             # 有效拖拽物进入目标区域
            play sound "sfx/slot_hover.ogg"
            emit "slot_highlighted" slot_id

        on_drag_leave:
            emit "slot_unhighlighted" slot_id

        on_drop (data):
            $ equip(slot_id, data)
            play sound "sfx/equip.ogg"

        on_drop_rejected (data):          # 放置但不满足 accept 条件
            play sound "sfx/denied.ogg"
```

### 触摸事件（Android）

```apy
gui swipe_area:
    on_swipe_left:
        jump next_scene

    on_swipe_right:
        jump prev_scene

    on_swipe_up:
        show screen history_panel

    on_swipe_down:
        hide screen history_panel

    on_pinch (scale):                     # 双指缩放
        $ store["zoom"] = clamp(scale, 0.5, 2.0)

    on_tap:                               # 单次点击（触摸版 on_click）
        pass

    on_long_tap (threshold=0.5):
        show screen context_menu
```

`engine.variant("touch")` 为 `true` 时触摸事件才会触发，PC 端静默忽略，不需要条件判断。

### 全局输入事件

与 `on key` 平级，处理连续输入：

```apy
on mouse_move (pos, delta):
    $ store["cursor_pos"] = pos

on mouse_wheel (delta):
    if store["active_panel"] == "map":
        $ store["map_zoom"] = clamp(store["map_zoom"] + delta * 0.1, 0.5, 3.0)

on gamepad_axis (axis, value):
    if axis == "left_x":
        $ store["aim_x"] = value
    elif axis == "left_y":
        $ store["aim_y"] = value
```

只允许出现在文件顶层，与 `on key` 规则一致。`on mouse_move` 在 Auto/Skip 模式下正常触发（不涉及对话推进）。

### 焦点导航完整规则

```apy
gui menu_panel:
    focus_group "main_menu"
    focus_wrap true                   # 到达末尾后回到开头
    focus_default btn_start

    # 线性导航（Tab 键、手柄上下）
    focus_order (btn_start, btn_load, btn_settings, btn_exit)

    # 2D 空间导航（手柄方向键）
    focus_2d:
        btn_start:    (right=btn_load,     down=btn_settings)
        btn_load:     (left=btn_start,     down=btn_exit)
        btn_settings: (up=btn_start,       right=btn_exit)
        btn_exit:     (up=btn_load,        left=btn_settings)
```

`focus_order` 只写线性导航，`focus_2d` 只写空间导航，两者可以同时声明，分别处理不同输入设备。

Round-Trip：`focus_group`、`focus_order`、`focus_2d` 在编辑器中以焦点导航面板独立展示，2D 导航以可视化连线图呈现。


---

## 内置转场库（完整版）

### 转场分类

```apy
# ── 淡化类 ────────────────────────────────────────────────
transition fade 1.0                          # 渐黑再渐亮（默认颜色黑）
transition fade_black 0.8
transition fade_white 0.5
transition fade_color (#ff8800) 1.0          # 自定义颜色
transition dissolve 0.5                      # 直接交叉溶解，不经过黑色
transition dip_black 0.5                     # 渐黑（单程，用于场景结尾）
transition dip_white 0.3

# ── 划像类 ────────────────────────────────────────────────
transition wipe_left 0.4
transition wipe_right 0.4
transition wipe_up 0.4
transition wipe_down 0.4
transition wipe_diagonal (angle=45) 0.5      # 斜向划像，angle 为角度

# ── 推移类 ────────────────────────────────────────────────
transition push_left 0.4                     # 旧场景被推走，新场景推入
transition push_right 0.4
transition push_up 0.4
transition push_down 0.4

# ── 滑入类（新场景动，旧场景不动）─────────────────────────
transition slide_in_left 0.4
transition slide_in_right 0.4
transition slide_in_up 0.4
transition slide_in_down 0.4

# ── 缩放类 ────────────────────────────────────────────────
transition zoom_in 0.5                       # 新场景从中心放大进入
transition zoom_out 0.5                      # 旧场景缩小消失，新场景显现
transition zoom_in_blur 0.5                  # 带模糊的放大
transition punch_in 0.3                      # 冲击感放大（overshoot 弹性）

# ── 图形类 ────────────────────────────────────────────────
transition iris_in 0.5                       # 圆形从中心扩展
transition iris_out 0.5                      # 圆形向中心收缩
transition iris_in (pos=(0.3, 0.7)) 0.5      # 从指定位置展开
transition blinds_h (slats=8) 0.4            # 水平百叶窗
transition blinds_v (slats=8) 0.4
transition checkerboard (size=32) 0.5
transition pixelate 0.4                      # 像素化过渡
transition ripple 0.6                        # 水波纹

# ── 特效类 ────────────────────────────────────────────────
transition flash (color=#ffffff, intensity=1.0) 0.2   # 闪光
transition shake_cut (intensity=10) 0.3               # 抖动后硬切
transition glitch 0.4                                  # 数字故障感
transition static 0.3                                  # 电视雪花噪点
transition dream 0.8                                   # 梦境感（模糊+亮度）
transition film_burn 0.6                               # 胶片灼烧

# ── 所有转场支持 easing 和 delay 参数 ─────────────────────
transition fade 1.0 (easing=ease_in_out)
transition dissolve 0.5 (delay=0.2)          # 延迟开始
transition push_left 0.4 (easing=ease_out)
```

### 转场全局配置

```apy
# options_window.apy
engine:
    transition:
        default_duration  = 0.4
        default_easing    = ease_in_out
        reduce_motion     = false    # 无障碍：true 时所有转场时长减半
```

`reduce_motion` 对应系统级无障碍设置（iOS/Android/Windows 均有对应系统偏好），引擎自动检测系统设置并覆盖此值，开发者也可强制设置。

### `TransitionLibrary.register` 扩展

```python
# startup 块内注册自定义转场
startup:
    python:
        from axn_plus.apy.transition import TransitionLibrary

        class PageFlip(Transition):
            def __init__(self, duration=0.6, direction="right"):
                self.duration  = duration
                self.direction = direction

            def apply(self, surface, progress):
                # progress: 0.0 → 1.0，引擎每帧调用
                angle = progress * 180
                return self._flip_surface(surface, angle, self.direction)

        TransitionLibrary.register("page_flip", PageFlip)
```

注册后与内置转场完全等价：

```apy
scene bg_new_room (with=page_flip)
scene bg_new_room (with=page_flip(direction="left", duration=0.8))
```


---

## 更新器（Auto-updater）

可选内置模块，开箱即用，不强制。在 `options_window.apy` 中开启后，`flow.apy` 的 `start` label 中设置调用点。

### 配置

```apy
# options_window.apy
engine:
    updater:
        enabled          = true
        check_url        = "https://example.com/game/version.json"
        download_base    = "https://cdn.example.com/game/patches/"
        channel          = "stable"        # stable / beta
        check_interval   = 86400           # 秒，0 = 每次启动都检查
        auto_apply       = false           # true = 下载完直接应用，不询问
        delta_update     = true            # 增量更新（只下载变化的文件）
        verify_signature = true            # 校验更新包签名
        signature_key    = "pubkey.pem"
        allow_downgrade  = false
```

### 调用

```apy
# flow.apy
label start:
    call axn::updater.check_and_apply
    call prologue
```

### 版本清单格式（服务端 `version.json`）

```json
{
    "latest": {
        "stable": "1.2.0",
        "beta":   "1.3.0-beta.2"
    },
    "versions": {
        "1.2.0": {
            "release_date": "2026-01-15",
            "changelog":    "修复了第三章存档问题，新增语音包支持",
            "full_package": {
                "url":    "patches/1.2.0-full.npa",
                "size":   52428800,
                "sha256": "abc123..."
            },
            "delta_from": {
                "1.1.0": {
                    "url":    "patches/1.1.0-to-1.2.0.npa",
                    "size":   3145728,
                    "sha256": "def456..."
                }
            }
        }
    }
}
```

### 更新包格式

`.npa` 归档扩展，增加差分支持：

```
patch.npa
├── manifest.json      # 操作列表
├── files/             # 新增或修改的文件
└── signature.sig      # 更新包签名（Ed25519）
```

```json
{
    "from_version": "1.1.0",
    "to_version":   "1.2.0",
    "operations": [
        {"op": "add",    "path": "main/scripts/ch3_extra.apy",   "sha256": "..."},
        {"op": "modify", "path": "main/scripts/ch2.apy",         "sha256": "..."},
        {"op": "delete", "path": "main/scripts/ch2_old.apy"}
    ]
}
```

### 更新 UI（`.apy` 模板，可替换）

```apy
gui UpdateDialog:
    dialog:
        vstack gap=16:
            text "发现新版本 {store['update_version']}" (font_size=18)
            text store["update_changelog"] (color=#aaaaaa)

            progress_bar bind=engine.update_progress (width=300)
                when store["update_state"] == "downloading"

            hstack gap=8:
                button "跳过此版本" on_click: Return("skip")
                button "稍后提醒"   on_click: Return("later")
                button "立即更新"   on_click: Return("update")
                    when store["update_state"] != "downloading"
```

### 存档兼容性联动

更新应用后首次启动，自动触发 `persistent` 版本迁移链，对旧存档运行 migration handler，不需要开发者额外处理。


---

## 翻译系统（完整设计）

### 三种模式

```apy
# options_window.apy
engine:
    i18n:
        mode = "renpy"      # Ren'Py 兼容模式
        # 或
        mode = "inline"     # Axn-Plus inline 模式
        # 或
        mode = "external"   # Axn-Plus external 模式
```

同一项目内只允许一种模式。混用时警告可 ignore。

---

### Ren'Py 模式

完整复现 Ren'Py 的翻译机制，Ren'Py 项目迁移时现有翻译文件零改动直接使用：

```apy
# 原始对话
label morning_scene:
    autumn "你好。"
    @ "阳光透过窗户照进来。"

# 翻译块（Ren'Py 风格，放在同文件底部或独立翻译文件）
translate en morning_scene_0:
    autumn "Hello."

translate en morning_scene_1:
    @ "Sunlight streams through the window."
```

Ren'Py 的 `auto voice` 同样保留：

```apy
engine:
    i18n:
        mode = "renpy"
    voice:
        mode       = "renpy"
        auto_voice = "voice/{id}.ogg"    # Ren'Py 风格自动路径模板
```

---

### Inline 模式（Axn-Plus 原有设计扩展）

翻译紧邻原文，适合小型项目和自行翻译：

```apy
label morning_scene:
    translate zh:
        autumn: "你好。"
        @ "阳光透过窗户照进来。"

    translate en:
        autumn: "Hello."
        @ "Sunlight streams through the window."
```

**块级翻译（减少重复）：**

```apy
translation block "morning_greeting":
    zh:
        autumn: "早上好。"
        @ "阳光透过窗户照进来。"
    en:
        autumn: "Good morning."
        @ "Sunlight streams through the window."
    ja:
        autumn: "おはよう。"
        @ "日光が窓から差し込んでいる。"
```

同一 block 内所有语言的行数必须一致，否则解析期报错：

```
AxnParseError: Line count mismatch in 'translation block "morning_greeting"'.
  zh: 3 lines, en: 2 lines, ja: 3 lines.
  All languages within a translation block must have the same number of dialogue lines.
  → scene.apy, line 45 (en section is missing a line)
```

**翻译继承（避免重复）：**

```apy
translate zh-tw extends zh:
    autumn: "早安。"    # 只覆盖这一条，其余继承 zh
```

**UI 字符串翻译：**

```apy
translate strings:
    zh:
        "开始游戏"    = "开始游戏"
        "设置"        = "设置"
        "退出"        = "退出"
        "存档槽 {n}"  = "存档槽 {n}"
    en:
        "开始游戏"    = "Start Game"
        "设置"        = "Settings"
        "退出"        = "Quit"
        "存档槽 {n}"  = "Save Slot {n}"
```

---

### External 模式

翻译完全从主脚本分离，放在独立文件里，主脚本只写原文：

```apy
# ch1.apy — 只写原文，不含任何 translate 块
label morning_scene:
    autumn: "你好。"
    @ "阳光透过窗户照进来。"
```

```apy
# strings/en.apy — 由 axn extract-strings 生成，手动填写翻译

# ch1.apy::morning_scene, line 5
translate en:
    autumn: "Hello."

# ch1.apy::morning_scene, line 6
translate en:
    @ "Sunlight streams through the window."
```

引擎启动时加载对应语言的翻译文件，覆盖原文。UI 字符串翻译同样独立：

```apy
# strings/en.apy
translate strings en:
    "开始游戏" = "Start Game"
    "设置"     = "Settings"
```

**External 模式优势：**
- 主脚本干净，不混入翻译内容
- 翻译人员只接触翻译文件，不需要理解 `.apy` 脚本结构
- 不同语言可以独立版本管理
- 适合翻译外包工作流

---

### 三种模式对比

| | Ren'Py 模式 | Inline 模式 | External 模式 |
|---|---|---|---|
| 翻译位置 | 同文件底部或独立文件 | 紧邻原文 | 完全独立文件 |
| 主脚本整洁度 | 中 | 低 | 高 |
| 迁移自 Ren'Py | 直接兼容 | 需转换 | 需转换 |
| 翻译外包友好度 | 中 | 低 | 高 |
| 推荐场景 | Ren'Py 迁移 | 小型项目自行翻译 | 多语言商业项目 |

---

### 翻译工具链完整设计

**提取：**

```
axn extract-strings --lang en --mode renpy
axn extract-strings --lang en --mode inline
axn extract-strings --lang en --mode external    # 读取 options_window 配置
axn extract-strings --lang en --format xlsx      # 表格格式，适合翻译外包
axn extract-strings --lang en --memory strings/memory.json   # 启用翻译记忆
```

**导入：**

```
axn import-strings --lang en --input strings/en.apy
axn import-strings --lang en --input strings/en.xlsx

# 导入时行为：
# - 按来源注释（文件 + label + 行号）精确匹配
# - 源码文本变化的条目标记 #CHANGED，不自动覆盖，需人工确认
# - 新增条目自动追加
# - 多余条目（源码已删除）标记 #OBSOLETE，保留不删除
```

**检查：**

```
axn check-strings --lang en --strict

[✓] 对话行完整性：247/247 (100%)
[✓] UI 字符串完整性：89/89 (100%)
[✗] 插值变量缺失（3 处）：
    → ch2.apy::battle_scene, line 42
      原文含 {player_name}，en 译文缺失此插值
[✗] 翻译块行数不匹配（1 处）：
    → translation block "ch2_battle"：zh 12行，en 11行
[⚠] 疑似未翻译（与原文相同，5 处）
[⚠] 译文长度超原文 150%（8 处，可能导致 UI 溢出）
[⚠] 标点不一致（3 处）：原文用"。"，译文用"."
```

**伪翻译：**

```
axn pseudo-translate --lang debug --length-factor 1.3
```

将原文字符替换为带变音的拉丁字母，长度为原文 1.3 倍。用途：验证所有文本走了翻译路径、测试 UI 对非 ASCII 字符的支持、测试长文本布局表现。

**迁移工具：**

```
axn migrate-strings --from renpy --to external --lang en
# 将现有 Ren'Py 格式翻译块转换为 external 模式独立文件
```

### 翻译记忆（Translation Memory）

```json
// strings/memory.json
{
    "version": 1,
    "entries": [
        {
            "source":     "你好。",
            "target":     "Hello.",
            "lang":       "en",
            "context":    "greeting",
            "confidence": 1.0,
            "uses":       12
        }
    ]
}
```

相似度高于 `--threshold`（默认 0.85）的字符串自动填充，标注置信度，翻译人员复查而不是从零翻译。

### 混用检测

```
axn check-strings

[⚠] 检测到翻译模式混用：
    当前配置：mode = "inline"
    但检测到以下非 inline 模式的写法：
    → strings/en.apy 存在（external 模式文件）
    → ch2.apy line 88：translate en morning_scene_0（renpy 模式格式）

    如需忽略：
    engine:
        i18n:
            ignore_mode_mismatch = true
```

### 运行时语言切换

```apy
$ engine.set_language("en")
$ engine.set_language("en", transition=fade 0.3)   # 带转场
```

切换时引擎重新渲染所有当前显示的 UI 文本，不需要重载场景。字体回退链随语言重新计算。

### 字体回退链

```apy
# options_window.apy
engine:
    fonts:
        fallback_chain:
            - "fonts/main.ttf"
            - "fonts/cjk_supplement.ttf"    # 主字体缺字时用此字体
            - "system"                        # 最终回退到系统字体
```

不同语言字符集要求不同字体，回退链保证不出现方块字。

---

## 语音系统（完整设计）

### 三种模式

```apy
# options_window.apy
engine:
    voice:
        mode     = "renpy"      # Ren'Py 兼容模式
        # 或
        mode     = "content"    # Axn-Plus content 模式
        # 或
        mode     = "sequence"   # Axn-Plus sequence 模式

        base_dir = "voice/"
        auto_play = true
```

同一项目内只允许一种模式，混用时警告可 ignore。

---

### Ren'Py 模式

完整复现 Ren'Py 的语音机制，手动标注或通过 `auto_voice` 模板自动查找：

```apy
# 手动标注
autumn: "你好。" (voice="autumn/autumn_01.ogg")
autumn: "今天天气不错。" (voice="autumn/autumn_02.ogg")

# auto_voice 模板（options_window.apy 中配置）
engine:
    voice:
        mode       = "renpy"
        auto_voice = "voice/{character}/{id}.ogg"
```

`axn extract-voice --mode renpy` 生成带 `voice=` 占位符的脚本副本，开发者手动填入路径。

---

### Content 模式（Axn-Plus 特有）

用对话内容的哈希作为文件名，解决特殊字符和跨平台兼容问题：

```
voice/
    autumn/
        a3f2c1.ogg      # hash("你好。") 的前 6 位
        b7e4d2.ogg      # hash("今天天气不错。") 的前 6 位
    sophia/
        c9f1a3.ogg
```

**重复对话处理：**

```apy
autumn: "你好。"           # a3f2c1.ogg
autumn: "你好。" (happy)   # a3f2c1_happy.ogg（修饰符不同时加修饰符后缀）
                           # 找不到 _happy 变体时回退到 a3f2c1.ogg，警告可 ignore

# 同一对话块内的重复
autumn: "你好。"           # a3f2c1.ogg
@ "她又说了一遍。"
autumn: "你好。"           # a3f2c1_2.ogg（块内第 2 次出现加序号后缀）
```

**对配音导演透明（manifest）：**

```
axn voice-manifest --output voice/manifest.csv
```

```csv
hash,   character, text,           file,              status
a3f2c1, autumn,    "你好。",        autumn/a3f2c1.ogg, found
b7e4d2, autumn,    "今天天气不错。", autumn/b7e4d2.ogg, missing
```

配音导演按 `text` 列录音，按 `hash` 列命名文件。

**内容变更检测：**

```
axn voice-check

[✗] 3 条对话文本已修改，对应语音文件哈希失效：
    → ch2.apy::battle_scene, line 42
      旧文本："我不会输的！"  旧文件：autumn/d4a2b1.ogg（仍存在但已失效）
      新文本："我绝不认输！"  新文件：autumn/e5c3d2.ogg（缺失）
[⚠] 2 个语音文件存在但未被任何对话行引用（可能是旧版本残留）
```

---

### Sequence 模式（Axn-Plus 特有）

按 label + 顺序编号，文件名 = 顺序编号（4位）+ 文本哈希（4位）：

```
voice/
    morning_scene/
        0001_a3f2.ogg    # 第 1 条，文本哈希前 4 位
        0002_b7e4.ogg
        0003_c9f1.ogg
```

顺序编号给配音导演看（录制顺序），文本哈希用于变更检测（编号不变但哈希变了说明内容被修改）。

**中间插入对话的处理：**

```
axn voice-resequence --label morning_scene
# 重新生成编号，输出重命名操作列表（不自动执行，需要确认）

需要重命名的文件（共 8 个）：
  morning_scene/0003_c9f1.ogg → morning_scene/0004_c9f1.ogg
  ...
执行重命名？[y/N]
```

**manifest 同样支持：**

```
axn voice-manifest --mode sequence --label morning_scene

seq,  hash, character, text,           file,                        status
0001, a3f2, autumn,    "早上好。",      morning_scene/0001_a3f2.ogg, found
0002, b7e4, sophia,    "你好啊。",      morning_scene/0002_b7e4.ogg, missing
```

---

### 三种模式对比

| | Ren'Py 模式 | Content 模式 | Sequence 模式 |
|---|---|---|---|
| 文件命名 | 开发者手动指定 | 文本哈希 | 顺序编号+哈希 |
| 对配音导演透明度 | 高（路径直观） | 需要 manifest | 需要 manifest |
| 剧本修改后稳定性 | 手动维护 | 哈希自动检测 | 编号+哈希双重检测 |
| 迁移自 Ren'Py | 直接兼容 | 需要转换 | 需要转换 |
| 推荐场景 | Ren'Py 迁移、精确控制 | 内容稳定的项目 | 按录制顺序交付的项目 |

---

### 对话块内的语音

两种 Axn-Plus 模式下对话块内的语音处理：

```apy
# Sequence 模式：对话块内继续顺序编号
with autumn (happy):
    "第一句。"     # morning_scene/0001_xxxx
    "第二句。"     # morning_scene/0002_xxxx
    "第三句。"     # morning_scene/0003_xxxx

# Content 模式：加 _with_N 后缀区分
with autumn (happy):
    "第一句。"     # hash("第一句。")_with_1.ogg
    "第二句。"     # hash("第二句。")_with_2.ogg

# together 块多角色语音（各自目录下独立文件）
together:
    autumn: "我们一起说！"    # autumn/hash.ogg
    sophia: "我们一起说！"    # sophia/hash.ogg
```

### 混用检测

```
axn voice-check

[⚠] 检测到 voice 模式混用：
    当前配置：mode = "content"
    但检测到以下非 content 模式的文件结构：
    → morning_scene/0001_a3f2.ogg（sequence 模式文件结构）
    → ch2.apy line 42：voice="autumn_01.ogg"（renpy 模式手动标注）

    如需忽略：
    engine:
        voice:
            ignore_mode_mismatch = true
```

---

## 字幕系统

### 基础配置

```apy
# options_window.apy
engine:
    subtitles:
        enabled        = false           # 默认关闭，玩家在设置里开启
        show_for_auto  = true            # Auto 模式下强制显示
        position       = "bottom_center"
        style          = subtitle_default
        sync_mode      = "voice"         # voice / text / both
```

`sync_mode`：

| 值 | 行为 |
|----|------|
| `voice` | 字幕跟随语音时长显示和消失 |
| `text` | 字幕跟随打字机效果 |
| `both` | 语音和文字都完成后才消失 |

### 字幕内容控制

```apy
autumn: "你好。" (voice="001")
# 字幕内容 = 对话文本（默认）

autumn: "……" (voice="001", subtitle="她沉默地点了点头。")
# 语音是叹气声，字幕是描述性文字

autumn: "你好。" (voice="001", no_subtitle)
# 不显示字幕
```

### 多语言字幕

字幕自动跟随当前界面语言，voice 是否跟随语言切换由 `voice_follow_lang` 控制：

```apy
# options_window.apy
engine:
    voice:
        voice_follow_lang = false    # false = 语音始终用原始语言（如日语配音+中文字幕）
```

### 外挂字幕文件

```apy
# options_window.apy
engine:
    subtitles:
        external_file    = "subtitles/{lang}.srt"    # SRT / VTT 均支持
        prefer_external  = false    # true = 优先用外挂字幕，找不到才用内嵌
```

允许不修改游戏本体的情况下替换字幕，适合玩家自制字幕或无障碍字幕。


---

## 工具链扩展

### `axn doctor` — 项目健康检查

```
axn doctor

[✓] flow.apy 存在，start label 已定义
[✓] options_window.apy 存在，最小配置完整
[✗] 7 个资源文件在脚本中引用但不存在
    → assets/autumn/angry.png (scene.apy:42)
    → bgm/battle.ogg (ch2.apy:18)
[✗] 3 个 label 被 jump/call 但未定义
    → route_c (scene.apy:87)
[⚠] 2 个 define char 使用了链式继承（超过 2 层）
[⚠] project.json 缺少 android.package，Android 构建将失败
[✓] 所有 Saveable 类有 __version__ 声明
[✓] 翻译完整性：zh 100%，en 87%（缺少 23 条）
[✓] 语音完整性：94%（缺少 18 条）
```

比 `axn lint` 更宽，覆盖资源完整性、跳转目标、配置就绪状态、翻译/语音完整性。发布前跑一次，不再靠记忆对照 checklist。

---

### `axn stats` — 项目统计

```
axn stats

脚本统计：
  总字数：         142,380 字
  对话行数：         3,847 行
  label 数：           234 个
  分支节点：            89 个
  平均游玩时长：    ~6.2 小时（按平均阅读速度估算）

角色统计：
  autumn:    1,203 行（31.3%）
  narrator:    891 行（23.2%）
  sophia:      445 行（11.6%）

资源统计：
  图片：    347 个，总计 284 MB
  音频：     89 个，总计 156 MB
  视频：     12 个，总计 1.2 GB
```

用途：了解项目规模、对外宣传（"50万字剧情"）、估算配音成本、制定本地化预算。字数统计去除标签和插值，只统计实际文本。

---

### `axn export-dialogue` — 对话导出

```
axn export-dialogue --format xlsx --output dialogue_export.xlsx
axn export-dialogue --format csv  --output dialogue_export.csv
axn export-dialogue --format json --output dialogue_export.json
```

导出全部对话为表格，列：文件、label、行号、角色、文本、语音路径（如有）、表情修饰符、备注。

用途：
- 发给配音导演做配音指导（含语音路径占位符）
- 发给翻译人员（比 `axn extract-strings` 更完整，含上下文）
- 字数统计和预算计算
- QA 检查对话内容

---

### `axn diff-save` — 存档对比

```
axn diff-save save_old.axnsave save_new.axnsave

store 差异：
  relationship.autumn:  50 → 75  (+25)
  flag_agreed:          False → True
  day:                  3 → 5  (+2)
  inventory:            [] → [{"id": "key_001", ...}]  (1 项新增)
```

比较两个存档的 store 差异，调试分支路由问题时比在代码里加 log 干净。

---

### `axn profile` — 性能分析

```
axn profile --label morning_scene

执行分析（label: morning_scene）：
  总耗时：       2,847 ms
  资源加载：       342 ms（12.0%）
    → autumn_happy.png:  89 ms（首次加载）
    → bg_room.png:       43 ms（首次加载）
  渲染帧：       1,203 ms（42.2%）
    → 最慢帧：   18 ms（第 42 帧，show autumn 触发布局重算）
  脚本执行：        87 ms（3.1%）
  等待用户输入：  1,215 ms（42.7%）

内存峰值：       187 MB
```

视觉小说的性能问题通常集中在特定场景，全局 profiling 噪声太大。`--label` 参数聚焦到具体场景分析。

---

### `axn check` — 存档兼容性预检

```
axn check --save saves/slot1.axnsave

[✓] 存档版本：1.1.0，当前版本：1.2.0
[✓] 所有 label 仍然存在（存档执行位置有效）
[✗] 3 个 flag 变量在存档中存在但新代码已删除：
    → old_debug_flag（可能误删，请确认）
    → temp_test_var
    → ch1_placeholder
[⚠] 2 个新增 flag 在存档中缺失（将用默认值补齐）：
    → autumn_voice_heard: bool = False
    → dlc1_unlocked: bool = False
[✓] 所有 Saveable 类版本匹配
```

在发布新版本前检查对旧存档的兼容性，不等到玩家读档时才暴露问题。

---

### `axn trace` — 执行追踪

```
axn trace --label morning_scene --output trace.json
```

记录一次完整 label 执行的所有指令、store 变化、资源加载事件，输出结构化 trace 文件。配合 `axn diff-save` 用于复现 bug，比加 log 精确，比 debugger 方便。

---

### `axn lint` 扩展检查项

在现有基础上追加：

- 未使用的 `define position`、`define group`、`define sound_group`
- `label alias` 中被引用的别名（提示迁移）
- `computed` 变量的循环依赖
- `defer` 块内包含对话行（报错）
- `once` 块的唯一标识冲突（同文件同行号）
- `#region` / `#endregion` 不匹配
- `beat` 指令但当前 music 未声明 BPM
- `signal` 发出但没有任何 `on signal` 监听（可能是遗漏，警告可 ignore）

---

## 引擎核心补充

### 存档槽元数据查询 API（扩展）

```python
engine.save_slots()
# 返回：[{
#   "slot":      "slot1",
#   "timestamp": 1716480000,
#   "label":     "morning_scene",
#   "chapter":   "第一章",
#   "thumbnail": "cache/thumbnails/slot1.png",
#   "playtime":  3672,    # 秒
#   "version":   "1.2.0"
# }, ...]

engine.save_slot_info(slot)
engine.save_exists(slot)
engine.delete_save(slot)
engine.copy_save(from_slot, to_slot)    # 新增：存档复制
```

---

### 游戏内计时器（扩展）

```python
engine.timer.start("battle_timer")
engine.timer.elapsed("battle_timer")     # float，已过秒数
engine.timer.pause("battle_timer")
engine.timer.resume("battle_timer")
engine.timer.stop("battle_timer")
engine.timer.reset("battle_timer")
engine.timer.remaining("battle_timer", total=60.0)  # 新增：倒计时剩余秒数
```

计时器状态自动纳入存档快照，读档后恢复，无需开发者手动处理。

---

### `persistent` 版本迁移钩子（补充）

迁移按版本号顺序链式执行，跨多个版本时自动串联：

```python
# options_window.apy 的 startup (before): 块中注册
engine.persistent.register_migration(
    from_version = 1,
    to_version   = 2,
    handler      = migrate_v1_to_v2
)

engine.persistent.register_migration(
    from_version = 2,
    to_version   = 3,
    handler      = migrate_v2_to_v3
)
# 从版本 1 读档时自动执行 v1→v2→v3 的完整迁移链
```

---

## Round-Trip Fidelity 补充（新增语法）

| 新增语法 | GUI 处理方式 |
|---------|-------------|
| `after N:` 块 | 脚本区带延迟标签的指令节点，delay 字段可编辑 |
| `repeat N:` 块 | 脚本区循环积木块，次数字段可编辑，标注"不含对话行" |
| `defer:` 块 | label 节点底部的退出清理区域，标注"离开时执行" |
| `once per_session/per_playthrough/ever` | 带生命周期标签的条件节点，下拉选择三种生命周期 |
| `rollback fence:` | 脚本区回滚边界节点，标注"玩家无法回滚到此点之前" |
| `unwind` / `unwind to` | 脚本区栈展开节点，目标 label 字段可选 |
| `snapshot "name"` / `restore "name"` | 脚本区状态快照节点，name 字段可编辑 |
| `define position` | 位置定义面板，坐标预览可视化；show 积木块位置下拉同时显示具名位置 |
| `define group` | 角色组定义面板，成员列表可编辑；show/hide/expression group 对应积木块 |
| `define sound_group` | 音效组定义面板，文件列表可编辑，mode 下拉选择 random/cycle |
| `label alias` | label 节点显示别名列表，标注"向后兼容，不推荐引用" |
| `computed:` 块 | 变量面板独立区域，公式字段可编辑，标注"派生，不存档" |
| `flag clamp` | flag 声明积木块内的约束字段，min/max 可编辑 |
| `flag on_change` | flag 声明积木块内的回调字段，函数名可编辑 |
| `flag namespace` | 变量面板以命名空间分组展示 |
| `flag track_history` | flag 声明积木块内的历史追踪开关，max 字段可编辑 |
| `store.query()` / `store.history()` | 脚本区代码节点 |
| `beat` / `wait beat` / `at beat` | 脚本区节拍等待节点；标注"需要当前 music 声明 BPM" |
| `define camera_path` | 镜头路径定义面板，keyframe 时间轴编辑器，与 transform 编辑器共享 |
| `camera path` | 脚本区镜头路径节点，路径名下拉选择 |
| `signal "name"` | 脚本区信号发送节点，信号名字段可编辑 |
| `on signal "name":` | 独立事件钩子面板，与 on enter/on key 并列；发送方和接收方以连线标注 |
| `#region` / `#endregion` | 编辑器折叠区域，区域名显示在折叠标签上 |
| `mask=` | show 积木块中独立遮罩字段，mask_mode 下拉选择 |
| `render_texture as` | 脚本区离屏渲染节点，块内 show 指令列表可编辑 |
| `blend=` | show 积木块中混合模式字段，下拉选择内置混合模式 |
| `define particle` | 粒子系统配置积木块，参数完整可编辑，编辑器内实时粒子预览 |
| `show particle` / `hide particle` | 脚本区粒子显示/隐藏节点，与普通 show/hide 同类 |
| `play music (transition=crossfade N)` | 音频积木块内的切换策略字段，下拉选择 cut/crossfade/stinger/wait_bar |
| `define audio_bus` | 音频总线定义面板，层级树状展示，effects 列表可编辑 |
| `audio_snapshot save/restore` | 脚本区音频快照节点，name 字段可编辑 |
| 控件事件钩子（`on_hover_enter` 等） | 控件节点上的事件面板，各事件钩子独立可编辑代码块 |
| `on_active (interval=)` | 事件面板中的持续触发字段，interval 可编辑 |
| `on_long_click` / `on_double_click` | 事件面板独立字段，threshold/interval 可编辑 |
| `on_state_enter/exit (state=)` | 事件面板状态变化钩子，state 下拉选择六态 |
| 组合态（`hovered + selected`） | 样式编辑器中以 `+` 分隔的组合状态标签，优先级自动标注高于单态 |
| 定向过渡（`pressed -> hovered`） | 样式编辑器中的定向过渡字段，source/target 状态下拉选择 |
| `bind selected =` | 控件节点 selected 绑定字段，表达式可编辑 |
| 拖拽事件（`on_drag_start` 等） | draggable/droptarget 积木块中的完整事件面板 |
| 触摸事件（`on_swipe_*` / `on_pinch`） | 触摸区域积木块的事件面板，标注"仅 touch 平台有效" |
| `on mouse_move` / `on mouse_wheel` / `on gamepad_axis` | 独立事件钩子面板，与 on key 并列 |
| `focus_2d:` | 焦点管理面板中的 2D 导航可视化连线图 |
| 内置转场完整分类 | 转场选择器按分类（淡化/划像/推移/滑入/缩放/图形/特效）分组展示 |
| `transition A -> B:` 定向过渡 | 样式编辑器定向过渡字段 |
| `transition all N easing` | 样式编辑器全局过渡字段 |
| 更新器配置 | options_window.apy 专属更新器配置面板 |
| 翻译三种模式 | 编辑器翻译面板顶部显示当前模式标签；Ren'Py 模式显示兼容标记 |
| `translation block` | 翻译面板中的块级翻译节点，多语言标签页并排展示 |
| `translate zh-tw extends zh` | 翻译继承关系标注，继承来源语言显示为灰色 |
| `translate strings` | 翻译面板独立 UI 字符串区域 |
| 语音三种模式 | 语音面板顶部显示当前模式标签 |
| `axn voice-manifest` 生成的 manifest | 语音面板中集成 manifest 预览，缺失文件高亮标红 |
| 字幕配置 | options_window.apy 专属字幕配置面板，sync_mode 下拉选择 |
| `subtitle=` / `no_subtitle` 修饰符 | 对话积木块上的字幕控制字段 |


---

## 不是什么

- 不是 Ren'Py 的分支或 fork
- 不是面向零编程基础用户的工具（引擎本身）
- 不追求最大化跨平台覆盖
