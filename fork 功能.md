• //fork 的作用是把一个已有会话“分叉”成新会话。新会话会带着原会话到分叉点为止的上下文开始，但之后两边各走各的，互不影
  响。适合“保留当前讨论，再试另一条思路”。

  从本机帮助看，底层对应的是：

  codex fork [SESSION_ID] [PROMPT]

  常用方式是：

  codex fork

  这会打开会话选择器，选一个旧会话来分叉。

  codex fork --last

  直接从最近一次会话分叉。

  codex fork --last "改用另一种实现方案继续"

  从最近会话分叉，并立刻给新分支一个新提示。

  codex fork <session_id> "继续这个分支，但只做前端部分"

  从指定会话 ID 分叉。

  几个实用点：

  - fork 和 resume 不一样
    resume 是接着原会话继续；fork 是复制一份历史，开新分支。
  - --all 会显示所有会话，而不只当前目录相关的会话。
  - 你也可以在分叉时顺手换模型、沙箱等参数，比如 -m、-s。

  如果你说的是交互界面里的 //fork，它大概率就是这个 codex fork 的快捷入口；如果界面里输 //fork 没反应，直接在 shell 里用
  codex fork --last 就行。
  如果你愿意，我可以顺手给你演示一组最常用的 fork / resume 对比命令。




• fork 之后没有“切回主分支”的 git 式切换命令，本质上是两个独立会话。

  想回到原来的主会话，做法是重新 resume 原会话：

  codex resume

  这会打开会话选择器，你选最初那个会话就行。

  如果你记得原会话的 SESSION_ID，可以直接：

  注意：

  - codex resume --last 只会回到“最近一次会话”，很多时候那其实是你刚 fork 出来的分支，不一定是主会话。
  - 所以从 fork 分支回主会话，最稳的是用 codex resume 走选择器，或者直接填原会话 ID。
  - 当前 fork 会话不用删，退出后随时还能再 resume 回来。

  最简单的理解就是：

  - fork = 从旧会话复制一条新线
  - 回主线 = resume 原来的那条线

  如果你要，我可以再给你一个“主会话 -> fork -> 回主会话”的完整操作示例。




 fork 可以理解成“从某个会话复制一条新分支”。原会话不会消失，之后两边各聊各的。这里没有真正的“主分支”概念，所谓主分支就
  是你原来的那个会话。

  4 个场景

  1. 当前会话马上分叉
     如果你的交互界面支持 //fork，直接输入它即可。更稳的通用方式是在 shell 里运行：

  codex fork --last

  要在分叉后立刻给新分支一个新任务：

  codex fork --last "继续当前上下文，但改用方案 B"

  2. 从旧会话分叉
     最简单是打开选择器：

  codex fork

  如果你想看所有目录下的历史会话：

  codex fork --all

  如果你知道会话 ID，直接指定：

  codex fork <SESSION_ID>
  codex fork <SESSION_ID> "只继续前端部分"

  3. 从 fork 分支回到原会话
     用 resume 回原来的那个会话：

  codex resume

  会弹出会话选择器，你选最初那个会话就行。
  如果知道原会话 ID：

  codex resume <SESSION_ID>

  注意：刚 fork 完不要盲用 codex resume --last，因为“最近一次会话”很可能就是你刚开的 fork 分支。

  4. 一套最常用命令

  # 分叉最近会话
  codex fork --last

  # 分叉最近会话，并立刻给新提示
  codex fork --last "试另一条实现路线"

  # 从选择器里挑一个旧会话分叉
  codex fork

  # 显示所有会话再分叉
  codex fork --all

  # 回到原会话
  codex resume

  # 直接回到指定原会话
  codex resume <SESSION_ID>

  最实用的工作流

  - 保留原会话不动，另开一个终端执行 codex fork --last
  - 在新分支里试不同方案
  - 要回原线时执行 codex resume，选原会话

  如果你愿意，我下一条可以直接给你一份“从当前会话 fork，再切回原会话”的最短操作清单，只保留 3 步。
