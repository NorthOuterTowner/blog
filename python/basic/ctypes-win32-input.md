# 用 ctypes 调 Win32 SendInput 控制鼠标

需要写一个脚本让 Windows 保持"有人在用"的状态——具体点说，是让系统的空闲计时器不要走到锁屏那一步。最先想到的是 `SetCursorPos`，把鼠标指针挪一挪。跑起来发现完全没用，该锁还是锁。

查下去才知道 `SetCursorPos` 不算输入事件，必须用 `SendInput`。而 `SendInput` 的签名带联合体和归一化坐标，用 ctypes 声明要注意几个地方，多显示器和高 DPI 下还有额外的换算。本文记录这套调用的完整写法和踩过的坑，环境是 Windows 11 + Python 3.11 标准库 ctypes，不依赖 pywin32。

## SetCursorPos 为什么不行

`SetCursorPos` 只是修改指针位置，属于"改状态"，不产生输入事件。系统的空闲判断走的是 `GetLastInputInfo`，它读的是最后一次**真实输入事件**的时间戳。指针位置变了但没有输入事件，空闲计时器照走。

想验证这一点很容易：

```py
import ctypes, time
from ctypes import wintypes

user32 = ctypes.windll.user32

class LASTINPUTINFO(ctypes.Structure):
    _fields_ = [('cbSize', wintypes.UINT), ('dwTime', wintypes.DWORD)]

def idle_seconds():
    info = LASTINPUTINFO()
    info.cbSize = ctypes.sizeof(info)
    user32.GetLastInputInfo(ctypes.byref(info))
    return (user32.GetTickCount() - info.dwTime) / 1000.0
```

调 `SetCursorPos` 之后 `idle_seconds()` 继续增长；调 `SendInput` 之后立刻归零。

## 结构体声明：关键是匿名 union

`INPUT` 结构在 C 里是一个 tagged union：一个 `type` 字段，加一个包含 `MOUSEINPUT` / `KEYBDINPUT` / `HARDWAREINPUT` 的匿名联合。ctypes 里要用 `_anonymous_` 才能还原这个写法：

```py
import ctypes
from ctypes import wintypes

user32 = ctypes.windll.user32

INPUT_MOUSE = 0
MOUSEEVENTF_MOVE = 0x0001
MOUSEEVENTF_ABSOLUTE = 0x8000
MOUSEEVENTF_VIRTUALDESK = 0x4000


class MOUSEINPUT(ctypes.Structure):
    _fields_ = [('dx', wintypes.LONG),
                ('dy', wintypes.LONG),
                ('mouseData', wintypes.DWORD),
                ('dwFlags', wintypes.DWORD),
                ('time', wintypes.DWORD),
                ('dwExtraInfo', ctypes.POINTER(wintypes.ULONG))]


class INPUT(ctypes.Structure):
    class _U(ctypes.Union):
        _fields_ = [('mi', MOUSEINPUT)]

    _anonymous_ = ('u',)          # 关键：否则得写 inp.u.mi
    _fields_ = [('type', wintypes.DWORD), ('u', _U)]


def send_mouse(dx, dy, flags):
    inp = INPUT(type=INPUT_MOUSE,
                mi=MOUSEINPUT(dx, dy, 0, flags, 0, None))
    user32.SendInput(1, ctypes.byref(inp), ctypes.sizeof(inp))
```

`_anonymous_ = ('u',)` 把联合体成员提升到外层，于是可以直接 `INPUT(type=..., mi=...)`，也可以 `inp.mi.dx` 访问。少了这一行就得写成 `inp.u.mi.dx`，而且构造函数里传 `mi=` 会报错。

`dwExtraInfo` 声明成指针类型是照抄 C 头文件（那里是 `ULONG_PTR`）。传 `None` 就行，也可以塞一个自定义的标记值，用来在钩子里区分"这个事件是我自己发的"。

## 绝对移动的坐标不是像素

这是最容易出错的地方。带 `MOUSEEVENTF_ABSOLUTE` 时，`dx` / `dy` 不是像素，而是 **0..65535 的归一化坐标**，而且默认只映射到主显示器。

多显示器必须加 `MOUSEEVENTF_VIRTUALDESK`，并且按**虚拟桌面**的尺寸换算：

```py
SM_XVIRTUALSCREEN, SM_YVIRTUALSCREEN = 76, 77
SM_CXVIRTUALSCREEN, SM_CYVIRTUALSCREEN = 78, 79


def virtual_desktop():
    g = user32.GetSystemMetrics
    return (g(SM_XVIRTUALSCREEN), g(SM_YVIRTUALSCREEN),
            g(SM_CXVIRTUALSCREEN), g(SM_CYVIRTUALSCREEN))


def move_to(x, y):
    left, top, width, height = virtual_desktop()
    nx = int(round((x - left) * 65535 / (width - 1)))
    ny = int(round((y - top) * 65535 / (height - 1)))
    flags = MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE | MOUSEEVENTF_VIRTUALDESK
    send_mouse(nx, ny, flags)
```

几个细节：

- 指标编号 **76~79** 是虚拟屏幕，不是 0/1。`GetSystemMetrics(0)` / `(1)` 是主屏的宽高，用它换算在副屏上会整体偏。
- 分母是 `width - 1` 而不是 `width`。65535 要对应最右边那一列像素。
- 虚拟桌面的原点可以是负数（副屏在主屏左边时 `left < 0`），所以要先减 `left` / `top` 平移到 0 起点。

## 高 DPI 缩放要先声明 DPI 感知

系统缩放不是 100% 时，进程如果不是 DPI 感知的，Windows 会给它一套虚拟化的坐标，读到的屏幕尺寸和实际像素对不上，移动结果整体偏。启动时先声明一次：

```py
user32.SetProcessDPIAware()
```

这个调用必须在读取任何屏幕指标之前，而且一个进程只生效一次。更新的 API 是 `SetProcessDpiAwarenessContext`，但要处理不同 Windows 版本的可用性，脚本场景 `SetProcessDPIAware()` 够用。

## 配套的两个小工具

**读当前位置**：

```py
class POINT(ctypes.Structure):
    _fields_ = [('x', wintypes.LONG), ('y', wintypes.LONG)]


def cursor_pos():
    pt = POINT()
    user32.GetCursorPos(ctypes.byref(pt))
    return pt.x, pt.y
```

**非阻塞读按键**，给循环脚本做退出键：

```py
def esc_pressed():
    return bool(user32.GetAsyncKeyState(0x1B) & 0x8000)
```

`GetAsyncKeyState` 的高位表示"当前是否按下"，低位表示"上次调用以来是否按过"。做退出键用高位，不需要事件循环，在 `while` 里每轮查一次就行。

## 实测：归一化换算没有累计漂移

一开始担心 0..65535 的整数换算会有舍入误差，长时间运行会让实际位置慢慢偏离指令位置。跑了个验证：脚本连续移动 3 秒，记下最后一次指令的目标坐标，然后读 `GetCursorPos`：

```shell
$ python -u mouse_test.py
last command : (1487, 622)
GetCursorPos : (1487, 622)
match: True
```

完全相等。原因是每次都用绝对坐标，误差不会累加——`round` 造成的偏差最多 1 像素，而且下一次移动会重新从目标坐标算，不继承上一次的偏差。

这个性质有个额外用途：可以用来**检测真人是否插手**。把每次指令的目标坐标记下来，下一步动作前先读实际位置，两者偏差超过阈值就说明期间有人动了鼠标，脚本可以据此退出，避免和使用者抢指针。

```py
expected = (nx_px, ny_px)
actual = cursor_pos()
if abs(actual[0] - expected[0]) > 4 or abs(actual[1] - expected[1]) > 4:
    print('检测到人工干预，退出')
    break
```

## 小结

`SendInput` 相比 `SetCursorPos` 的差别不只是"更真实"，而是语义完全不同：一个是改状态，一个是产生事件。凡是涉及"让系统认为有输入"的需求，都只能走后者。

ctypes 这边的三个坎：匿名 union 的 `_anonymous_`、绝对坐标是 0..65535 而非像素、多显示器和高 DPI 各自需要额外一步。都不难，但错了之后表现是"指针跑到别的地方去了"，不容易一眼看出是哪一环。
