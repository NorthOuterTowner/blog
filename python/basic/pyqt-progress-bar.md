# 用 QThread 和嵌套区间进度模型实现不卡顿的多阶段算法进度条

在给一个数据处理工具加算法模块时遇到一个问题：单条数据要跑 10+ 秒，界面直接卡死。用户点完按钮后不知道发生了什么的，只能盯着光标等，或者以为程序挂掉了。第一个反应是把算法扔进 QThread，但跑通之后发现进度条的表现很诡异——日志显示算法只用了 1 秒，界面上却已经过了 2 秒，而且进度条在某些步骤上会突然卡住十几秒不动。查下去才知道问题不在线程创建和销毁，而是进度上报的方式本身有性能陷阱。本文记录这套进度模型的完整设计：分层解耦的进度模型、嵌套区间的栈操作、以及为什么跨线程时「推」要改成「拉」。

## 它解决什么问题

多阶段算法（如「读取 → 多文件循环 → 每文件多行 → 每行多步处理 → 保存」）需要一个进度条来告知用户「进行到哪一步了」。这个进度条需要满足几个条件：

- **UI 不能卡**：主线程必须保持响应，用户应该能随时点取消，或者切换到别的窗口。
- **进度要准确**：不能只在每份文件开始时跳一下，10 秒的算法步骤应该在进度条上平滑走 10 秒。
- **权重不均分**：不同步骤耗时差异很大（预处理 0.1 秒，DBSCAN 5 秒，画图 3 秒），均分会让进度条在快步骤上飞过、在慢步骤上看起来像卡死。
- **层级解耦**：进度的代码分散在多个文件（读取层、算法层、UI 层），每层不应该知道「我现在到底应该报 37% 还是 41%」这种全局位置信息，否则每改一次流程所有层都要跟着改。

没有这套模型之前，通常的做法是每层自己算一个百分比，主线程只负责画出来。这种做法在简单场景能跑，但权重不均时进度条会「跳」，层级耦合时加一个步骤要改三处代码，调试时不知道某个 50% 到底对应哪一步。

## 核心概念

### UI 线程（主线程）与工作线程

PySide6/Qt 的界面必须在主线程操作。主线程有一个**事件循环**，负责接收鼠标点击、定时器、跨线程信号等事件，然后分发给相应的槽函数。如果在主线程里跑一个 10 秒的 numpy 计算，事件循环会被阻塞 10 秒，期间界面不会重绘、不会响应点击，看起来就像卡死。

**QThread** 是 Qt 的跨线程机制。把它子类化、把算法扔进 `run()` 方法，再 `start()`，算法就会在另一个操作系统线程里跑，主线程的事件循环不受影响。

### 信号与槽（Signal/Slot）与 Queued Connection

跨线程时信号不会直接调用槽，而是**投递到接收方的事件队列里排队**。这意味着工作线程发了 10 条信号，主线程要逐个处理完才能处理下一条。这是「推」模式性能问题的根因。

### 嵌套进度区间

进度条只有一根（0%~100%），但进度的代码分散在多层。需要一种机制，让每层只声明自己这层的「相对权重」，系统负责把它映射到全局位置。「嵌套区间」就是这种机制的核心数据结构。

## 进度模型的结构与关键设计

### 三层分离：模型层、线程层、UI 层

进度模型本身不依赖 Qt，这样算法层可以保持纯净，被 CLI 或测试直接调用。

```py
# process/progress.py（纯 Python）
class Progress:
    def __init__(self, emit):
        self._emit = emit      # callback: (fraction: float, text: str|None) -> None

    def plan(self, weights): pass
    def zoom(self, key, text=None): pass
    def step(self, key): pass
```

```py
# ui/deal_worker.py（QThread 子类）
class DealWorker(QThread):
    def run(self):
        read_input()
        progress = Progress(self._record_progress)   # 纯 Python Progress 实例
        for file in files:
            process_file(progress)
        save_output()
```

```py
# ui/deal_tab.py（UI 层）
class DealTab(QWidget):
    def _run(self):
        self._worker = DealWorker()
        self._worker.finished_ok.connect(self._on_success)
        self._worker.start()   # 微秒级，UI 瞬间响应
```

这样分层之后，算法层 (`process_file`) 不知道也不需要知道 Qt，进度模型 (`Progress`) 不需要知道自己在哪个线程里，只有 UI 层处理信号和界面刷新。

### 嵌套区间建模：栈结构与三条规则

进度模型的核心是**一个栈**，栈里的每个元素是一个区间段 (`_Segment`)。整个进度条的位置由栈顶决定，全局位置由整个栈叠加决定。

```py
class _Segment:
    __slots__ = ('lo', 'hi', 'weights', 'total', 'done', 'pending')
```

`lo`/`hi` 是这个段在全局 0~1 尺度上的绝对位置（常量，压栈时刻算好），其余都是相对量。这三条规则是理解整个模型的关键：

**规则一**：只有 `lo`/`hi` 是全局绝对坐标，`weights`、`total`、`done` 全是相对于当前段的。

**规则二**：`_pos(done)` 是唯一的映射函数，把段内的相对进度转换成全局位置：

```py
def _pos(self, done):
    seg = self._seg
    return seg.lo + (seg.hi - seg.lo) * min(done / seg.total, 1.0)
```

**规则三**：只有栈顶是「当前工作面」。`self._seg` 就是 `self._stack[-1]`，所有操作都针对栈顶。

### 三个操作对栈的影响

`zoom(key)` 是唯一改变栈深度的操作。它用 `@contextmanager` 实现，天然支持 `with` 语句：

```py
@contextmanager
def zoom(self, key, text=None):
    self._commit()
    parent = self._seg
    weight = parent.weights.get(key, 0.0)

    # 子段的全局区间 = [父段当前done位置, 父段当前done+权重位置]
    lo = self._pos(parent.done)
    hi = self._pos(parent.done + weight)

    self._stack.append(_Segment(lo, hi))
    try:
        yield self
    finally:
        self._stack.pop()
        parent.done += weight          # 整块结算，不管内部走了几步
        parent.pending = None
        self._fire(self._pos(parent.done), None)
```

关键洞察是：**子段不知道父段存在**。压栈时刻把父段的 `done + weight` 换算成全局位置存进 `lo`/`hi`，之后子段内部只操作自己的 `done`，退出时把 `weight` 加回父段的 `done` 就行。这实现了完全的解耦。

`plan(weights)` 只改栈顶的权重表和 `total`，不碰栈深度。`step(key)` 只改栈顶的 `pending`，标记「这一步刚开始，还没跑完」。

### `pending` 延迟结算：标签显示「正在跑」而不是「刚跑完」

`step(key)` 表示「开始跑 key」，此时还没跑完，不能计入 `done`，所以先挂到 `pending`。等下一个 `step` 或 `zoom` 来了才 `_commit`：

```py
def step(self, key):
    self._commit()
    self._seg.pending = key
    self._fire(self._pos(self._seg.done), key)
```

好处有两个：

1. **标签实时显示「正在跑 dbscan」**，而不是等它跑完才跳到下一个步骤名。进度条位置和步骤名始终同步。
2. **被条件守卫跳过的步骤不会卡住进度条**。如果某行因为缺字段被跳过，`zoom` 不会进入，`step` 不会被调，权重永远不会 `commit`。但退出时父段是整块 `+weight`，所以位置照样落对，只是中间少几个台阶。

### 权重按预估耗时，不均分

均分的话进度条会在便宜的步骤上飞过、在贵的步骤上僵住十几秒。改成按预估耗时配权重后，进度条节奏和用户感知一致：

```text
根（1 个文件）： {convert: 0, run: 100}     # npz 输入时 convert 跳过
文件（run）： {load: 22, process: 78}      # load 读磁盘快，process 包含算法
处理（process）：各行等权
单行（每行）： {prep: 1, track_below: 1, track: 1, output: 2}  # output 要落盘，多给权重
```

改这些数只影响进度条节奏，不影响任何计算结果。

### 完整追踪示例

用一次具体执行（1 个文件、2 行、每行权重 `prep1 + track_below1 + track1 + output2 = 5`）来追踪栈的变化：

```text
初始: 栈=[A(0,1)]，A.plan({0:1, 1:1})，0 权重的 convert 被过滤

zoom(0) → 栈=[A, B(0.00,1.00)]，B.plan({run:100})

zoom(run) → 栈=[A, B, C(0.00,1.00)]，C.plan({load:22, process:78})

  step(load) → 报 _pos(0)=0.00，标签 "load"，进度 0%

zoom(process): _commit() → C.done=22

  → 栈=[A, B, C, D(0.22,1.00)]，D.plan({0:1, 1:1})

zoom(0) → 栈=[A,B,C,D,E(0.22,0.61)]，E.plan(行权重)，total=5

  step(prep)        → E.done=0， 报 0.22 + 0.39*(0/5) = 0.220，进度 22%
  step(track_below) → E.done=1， 报 0.22 + 0.39*(1/5) = 0.298，进度 30%
  step(track)       → E.done=2， 报 0.22 + 0.39*(2/5) = 0.376，进度 38%
  step(output)      → E.done=4， 报 0.22 + 0.39*(4/5) = 0.532，进度 53%

  退出 zoom(0) → 弹 E，D.done=1.0，报 _pos(1.0) = 0.610，进度 61%
  （注意 output 的 pending 从未被 commit，E.done 停在 4/5，
    但父段是整块 +1.0，所以位置照样对 —— 这就是「跳过不卡住」）

zoom(1) → E'(0.61,1.00) 同样四步，进度 61%→63%→71%→84%

层层收尾：退 process → C.done=100 报 1.00；退 run；退文件 → 100%
```

这个追踪表展示了栈操作的完整语义，也是给初学者讲清楚「压栈时区间就固定了」最直观的方式。

## 关键设计取舍：推改拉

### 问题的发现：日志显示 1 秒，界面过了 2 秒

实测数据（推模式，64 行 × 8000 点）：

| 阶段 | 时间 |
| --- | --- |
| 线程启动延迟 | 2.0 ms |
| 算法本体 | 13503 ms |
| **回传等待** | **979 ms** |
| 画图 | 4524 ms |

「回传等待」的定义是：工作线程 `run()` 结束的时刻 → 主线程真正执行 `finished_ok` 槽的时刻。算法本身只用了 13.5 秒，但用户要等将近 1 秒才看到「完成」回调。

根因不是单条信号贵，而是**UI 刷新次数和算法步骤数绑死了**。本例发了 321 条 progress 信号 + 67 条 log 信号，每条都会触发一次事件处理。`finished_ok` 被排在队列最后面，主线程得把前面的全啃完才轮到它。

### 为什么进度上报不能高频用「推」

跨线程的信号是**排队投递**的。工作线程发了信号 A、B、C，主线程的事件循环会按顺序处理 A、B、C，然后才处理 D。进度条每更新一次，主线程要做：

1. 取参数、拼字符串
2. `QProgressBar.setValue`（同步 repaint，约 0.85 ms/次）
3. `QLabel.setText`（约 0.01 ms/次）
4. 可能触发布局重排

行数一多、方法一叠，信号量线性涨。攒批和判重只能把系数调小，治不了根。

### 解法：改成「拉」

把「工作线程主动发」改成「主线程定时来取」。工作线程只写状态槽，主线程用 QTimer 轮询：

```py
# 工作线程
def _record_progress(self, fraction, text):
    with self._lock:
        self._pending_progress = (int(round(fraction * 100)), text)

def take_updates(self):
    with self._lock:
        progress, self._pending_progress = self._pending_progress, None
        return progress
```

```py
# 主线程（QTimer 每 100 ms 轮询一次）
def _poll(self):
    progress = self._worker.take_updates()
    if progress is not None:
        pct, text = progress
        if pct != self._last_pct:
            self.progress_bar.setValue(pct)
            self._last_pct = pct
        if text != self._last_text:
            self.status_label.setText(text)
            self._last_text = text
```

改完之后，UI 刷新次数 = 任务时长 ÷ 轮询间隔（100 ms），与行数、步骤数、数据量全部无关。理论上「回传等待」应该从 979 ms 降到几十毫秒（仅轮询间隔），但这点未实测，成文时需标注。

### 日志与进度分开处理

日志要保证不丢（审计材料），进度只保留最新一条（UI 上只有一个「当前阶段」的位置）：

```py
def _log(self, msg):
    with self._lock:
        self._pending_logs.append(str(msg))

def take_updates(self):
    with self._lock:
        logs, self._pending_logs = self._pending_logs, []
        progress, self._pending_progress = self._pending_progress, None
    return logs, progress
```

主线程取走后一次性 `'\n'.join(logs)` 追加到 QTextEdit，只做一次文档布局。逐行 `append` 且每行 `ensureCursorVisible()` 会每次强制重排。

## 性能相关的小细节

### 单调性钳制

权重是估的，某步比预估快时收尾位置可能倒退，进度条倒退会让人以为出错：

```py
def _fire(self, fraction, text):
    if fraction < self._last and not text:
        return
    self._last = max(self._last, fraction)
```

`text` 不为空时允许倒退（切换步骤名），否则只允许前进。

### `NullProgress` 与 `as_progress`

算法层统一用 `progress = as_progress(progress)` 处理「外部没传进度对象」的情况：

```py
class NullProgress:
    def plan(self, *args): pass
    def zoom(self, *args, **kwargs): yield from ()
    def step(self, *args): pass

NULL_PROGRESS = NullProgress()

def as_progress(prog):
    return prog if prog is not None else NULL_PROGRESS
```

这样算法代码里不用写 `if progress is not None`，零开销。

### `__slots__` 与锁

`_Segment` 用 `__slots__` 节省内存，`Progress` 的状态更新用 `threading.Lock`（GIL 下开销可忽略）保护。

### 耗时汇总要留残差项

原来汇总行只列「参数 + 读取 + 算法 + 组装 + 保存 + 画图」，从未声称等于墙钟时间，差额（回传等待）就藏在里面。改成：

```text
参数: 12ms | 读取: 34ms | 算法: 13503ms | 组装: 56ms | 保存: 78ms | 画图: 4524ms | 其他: 1046ms
```

保证各项之和恒等于全流程，这个套路值得单独强调。

## 踩坑记录

- `QThread` 对象必须挂在实例属性上（`self._worker`），用局部变量会被 GC，线程还没跑完就被析构。
- 收尾逻辑挂 `QThread.finished` 而不是自定义的成功信号：成功、失败、取消三条路都会走到，才不会漏掉解锁按钮和收进度条。
- 线程里的 `started_at`/`finished_at` 由工作线程写、主线程读，但主线程读的时候线程已经结束了（`finished_ok` 在那之后），没有竞争，不用加锁。
- 退出前要等线程收尾（置取消位 + `wait(timeout)`），否则 Qt 会报 `QThread destroyed while still running`。
- 画图必须留在主线程，matplotlib 的 Figure/canvas 不是线程安全的。

## 小结

这套进度模型的核心价值是**解耦**：进度模型不依赖 Qt，算法层不感知层级结构，UI 层不关心进度怎么算。嵌套区间把「每层只知道自己的相对权重」这件事用栈结构实现得很紧凑。推改拉的教训是跨线程时高频信号会让 UI 刷新次数和计算步骤数绑定，轮询从根本上切断了这个耦合。
