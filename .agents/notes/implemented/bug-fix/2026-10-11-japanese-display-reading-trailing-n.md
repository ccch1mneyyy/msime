# Agent Note: 日文罗马字画出的读音保留末尾单个 n

Status: implemented

## Problem

日文 26 键只打一个 `n` 就显示ん，打两个 `n` 还是一个ん，但退格要按两次：第一次退格后原始输入是 `n`，读音照样是ん，屏幕上什么都没变（#6997 第 4 项）。原因在共享引擎：`scan_romaji`（`crates/engine/src/japanese/romaji.rs`）把末尾单个 `n` 发成ん，这条规则同时供查词、确定和宿主画读音使用；`host::Session::snapshot` 把这个结果原样作为 `reading` 发给宿主，Android 把 `reading` 写进输入框的组字区。日文九键也受影响：な（`na`）删掉 `a` 剩下的 `n` 读作ん，`deleteJapaneseKana` 以为假名已经删完，留下一个ん。

## Decision

引擎快照新增 `display_reading`，查词和确定的规则不变：

- `romaji_display_reading_into` 与 `romaji_reading_into` 只差末尾单个 `n`：它留在待定尾部里，`n` 画成 `n`、`kan` 画成 `かn`，`nn`、`n'`、`n-`、`n` 加辅音照旧是ん。实现是给 `scan_romaji` 加一个只在这一处为真的 `keep_trailing_n` 开关。
- `host::Session::snapshot` 在日文罗马字、非本地模式、非专用英文时用它填 `display_reading`，其余情况与 `reading` 相同。`input-runtime` 的 `View` 原样转发。
- `reading` 的含义不变，仍是 `MSIME_COMMIT_READING` 上屏的读音（`kan` 是 `かん`）。`input-runtime` 的单假名占位规则、`CompositionDisplay.h` 说的「Enter 上屏的读音」、只读 `reading` 的宿主都不受影响。

Android 是第一个改读 `display_reading` 的宿主：组字区、键盘上的读音行和九键删除循环都改看它。Android 有两条不经引擎结束组字区的路径：光标被移走时先取消引擎组字再 `finishComposingText`，会话停下时直接 `finishComposingText`。两条路径上组字区里的文字都会留在文档里，所以它们改走 `finishCompositionInPlace`：画的和确定的不同时，用 `EditorBridge.settle` 先把组字区换成 `reading` 再结束，并把用户点到的选区放回原处。这样 `kan` 之后点别处，文档里留下的仍是 `かん`，和改动前一样。

## Alternatives considered

- 直接改 `reading` 的含义，让所有宿主一起显示 `かn`：改动最小、六个宿主同时生效。但 `reading` 已被当作「确定的读音」使用：`input-runtime` 靠它判断单个完整假名，给ん/ン保留前两个位置；iOS 光标移动时 `unmarkText` 留下画出的读音；网页引擎的 `routing.rs` 钉着 `nihon` 的预编辑是 `にほん`。改它会让这些地方一起变，而其中几个宿主这次没法验证。
- 由 Android 宿主自己判断「editing_text 末尾是单个 n 就把读音末尾的ん换回 n」：这是罗马字规则的第二份实现，违反「输入算法归引擎」的边界。
- 连同退格一起改成按假名删除（`nn` 一次退掉整个ん，微软和 Google 是这样）：要改 `JapaneseRomajiScheme` 的退格和光标编辑，属于另一项设计改动，这里不做。

## Consequences

Android 日文 26 键打 `n` 显示 `n`，再打一个 `n` 变成ん，退格一次回到 `n`、再退格清空，每一步都能看到变化；回车或空格转换照旧得到 `かん` / 漢。九键删除な、に、ぬ、ね、の时一次删掉整个假名。

`nn` 仍是两个罗马字、要退格两次才删掉一个ん，这一项的「当成一个字符」没有实现。iOS、Linux、HarmonyOS 和网页引擎仍画 `reading`，末尾单个 `n` 在那里还是立刻显示ん，要各自改读 `display_reading`，并按本篇处理各自不经引擎结束组字区的路径。macOS 和 Windows 画的是 `editing_text`，不受影响。代价是视图多了一个字段，每次按键多序列化一份短字符串。

## Verification

`romaji.rs` 的 `display_reading_keeps_a_trailing_lone_n` 和 `host/tests.rs` 的 `japanese_display_reading_keeps_a_trailing_lone_n` 在改动前（`display_reading` 先按 `reading` 填）失败、改动后通过；后者同时钉住 `reading` 仍是ん、`CommitReading` 上屏 `かん`、韩语两项相同。`cargo test -p msime-engine -p msime-input-runtime` 全部通过（含 golden 基准）。Android 的 `EditorSmoke` 覆盖 `settle` 的写入顺序和选区平移，`JapaneseNineKeyLayoutSmoke` 加上 `こn`。设备上的组字区显示和光标移走后的文档内容没有在模拟器上跑。
