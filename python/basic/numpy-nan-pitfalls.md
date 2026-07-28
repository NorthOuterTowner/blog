# 用 NaN 表示缺失之后，numpy 聚合函数的连带坑

在做数据清洗时选了一个看起来很省事的设计：被剔除的采样点不删除数组元素，而是写成 NaN。这样各列长度不变，列式二进制存储（npz 里的二维数组）不会因为行长不齐而存不进去，matplotlib 画散点也会自动跳过 NaN，绘图代码一行都不用改。

看着完美，跑起来炸了：画图时报 `Axis limits cannot be NaN or Inf`。查下去发现根因和绘图无关，是 `np.argmax` 不是 nan-aware。本文记录这个坑的排查过程，以及"用 NaN 表示缺失"这个设计会连带影响哪些调用。

## 报错现场

```shell
ValueError: Axis limits cannot be NaN or Inf
  File "...\plotting.py", line 413, in plot_peak_region
    axes[0].set_xlim(peak_x + x_lo, peak_x + x_hi)
```

第一反应是"matplotlib 处理不了 NaN"，于是去翻绘图代码，想在画之前把 NaN 过滤掉。方向错了。

## 根因：np.argmax 遇到 NaN 会停在那里

`np.argmax` **不做 NaN 屏蔽**。NaN 参与任何比较都返回 False，所以在"当前元素是否大于已知最大值"这个判断上，NaN 一旦成为已知最大值，后面的元素就再也赢不了它。实际表现是：数组里第一个 NaN 的下标被返回。

```py
>>> import numpy as np
>>> a = np.array([1.0, 5.0, np.nan, 9.0])
>>> np.argmax(a)
2
>>> a[np.argmax(a)]
nan
```

实测那份数据 8192 个点里有 5050 个 NaN，`np.argmax` 返回 142，正是第一个被剔除点的位置。于是 `peak_x = x[idx]` 拿到的是有效值但位置完全错，`peak_y` 是 NaN，喂给 `set_xlim` 才抛异常。

关键认知：**画图本身对 NaN 免疫**，`plot` 和 `scatter` 会跳过它们。炸的是"由聚合函数算出来、再喂给 `set_xlim` / `set_ylim` 的标量"。所以排查方向应该是往上游找 argmax / max / min，而不是在绘图调用附近打转。

## 解法：包一层 nan 安全的 helper

直接换成 `np.nanargmax` 还不够——它在全 NaN 的输入上会抛 `All-NaN slice encountered`。这种退化输入是真会出现的（某条曲线被整条剔掉）：

```py
import numpy as np


def peak_index(values):
    values = np.asarray(values, dtype=float)
    if values.size == 0 or not np.any(np.isfinite(values)):
        return None                      # 调用方跳过这条曲线
    return int(np.nanargmax(values))
```

`dtype=float` 不能省。从 JSON 或者 Python 列表转过来的数组可能是 object dtype，`np.nanargmax` 在 object 数组上不做 NaN 屏蔽，等于白包一层：

```py
>>> a = np.array([1.0, np.nan, 9.0], dtype=object)
>>> np.nanargmax(a)
1
>>> np.nanargmax(np.asarray(a, dtype=float))
2
```

返回 `None` 而不是抛异常，是让调用方决定怎么处理——这里的语义是"这条曲线没有可用数据，跳过不画"，不是错误。

## 需要成对替换的一组函数

选了 NaN 表示缺失，整条下游链路的聚合调用都得审一遍：

| 原函数 | nan 安全版本 | 备注 |
| --- | --- | --- |
| `np.argmax` / `np.argmin` | `np.nanargmax` / `np.nanargmin` | 全 NaN 时抛异常，要先判断 |
| `np.max` / `np.min` | `np.nanmax` / `np.nanmin` | 全 NaN 时返回 NaN 并给 RuntimeWarning |
| `np.mean` / `np.std` | `np.nanmean` / `np.nanstd` | 同上 |
| `np.median` | `np.nanmedian` | 同上 |
| `np.sum` | `np.nansum` | 全 NaN 返回 0，注意这个语义差异 |
| `np.ptp` | 手写 `nanmax - nanmin` | 没有 `np.nanptp` |
| 比较 / 排序 | 先 `np.isfinite` 过滤 | `np.sort` 会把 NaN 排到末尾 |

另外注意 `nan*` 系列在全 NaN 时的行为并不统一：`nansum` 返回 0，`nanmax` 返回 NaN 加警告，`nanargmax` 直接抛异常。凡是可能收到全 NaN 输入的地方，都得显式处理退化情况，不能指望统一语义。

## 验证要专门覆盖退化输入

修完之后跑正常清理过的文件是不够的，那些文件里 NaN 是零散分布的，走不到退化分支。要专门构造：

- 一行全 NaN 的数据：确认只是跳过不画，不抛异常。
- 一行完全没有 NaN 的数据：确认没有因为加了 helper 反而改变原有行为。
- 首元素就是 NaN 的数据：这是最容易触发 `argmax` 错误的形态。

第一种最重要。它是这个设计必然会产生的输入，却几乎不会在随手测试里出现。

## 小结

"用 NaN 表示缺失"是个好设计，代价是它把缺失语义推给了整条计算链路。选它的时候要一并接受：所有聚合调用都要审、所有退化情况都要显式处理。

排查这类问题的思路是**顺着数据流往上找第一个把数组变成标量的地方**。NaN 在数组里传播是安静的，只在收缩成标量、或者被交给不接受 NaN 的接口时才爆出来，爆的位置离病根往往很远。
