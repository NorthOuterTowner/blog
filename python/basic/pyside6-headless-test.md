# 给 PySide6 界面写无头 smoke test

给一个 PySide6 桌面工具加了一个新的处理 Tab：左边一堆参数控件，右边嵌一个 matplotlib 画布，点按钮跑一遍数据处理然后出图。改一次就手动启动程序、点几下、看图，循环几十次之后受不了了——想要一个能在命令行里跑完、直接告诉我"Tab 能构造、控件都生成了、业务流程跑通了、图长这样"的脚本。

Qt 在没有显示器的环境下默认起不来，但它自带一个 offscreen 平台插件，正好能满足这个需求。本文记录这套无头 smoke test 的写法，包括必须注意的 import 顺序、两条会误导人的警告，以及配合它的"参数注册表"设计。

## 核心：offscreen 平台 + 直接调槽函数

```py
import os
os.environ.setdefault('QT_QPA_PLATFORM', 'offscreen')   # 必须在 import QtWidgets 之前

from PySide6.QtWidgets import QApplication
from myapp.ui.deal_tab import DealTab

app = QApplication([])

tab = DealTab(None)
tab.file_list.addItem(r'D:\data\sample.npz')
tab._run(save=False)                    # 直接调槽函数，不用模拟点击

log = tab.log_edit.toPlainText()
assert '处理完成' in log, log
print(log)

tab.plot_widget.figure.savefig('out.png')   # 嵌入的 FigureCanvas 照样能存图
print('saved out.png')
```

三个要点：

**`QT_QPA_PLATFORM` 必须在 import QtWidgets 之前设。** Qt 在导入时就会决定用哪个平台插件，之后再改环境变量没有任何效果。用 `setdefault` 而不是直接赋值，这样在有显示器的机器上想看真窗口时，可以从外面传 `QT_QPA_PLATFORM=windows` 覆盖掉。

**直接调槽函数，不要模拟点击。** `QTest.mouseClick` 那套需要事件循环转起来，而且定位控件很啰嗦。smoke test 关心的是"业务逻辑能不能跑通"，按钮和槽函数的连线只要在构造时 `clicked.connect(self._run)` 写对了就不用测。直接调 `tab._run()` 更直接，出错时栈也干净。

**嵌入式画布可以直接 savefig。** matplotlib 的 `FigureCanvasQTAgg` 持有的还是一个普通的 `Figure` 对象，跟界面渲染没关系。offscreen 模式下窗口不显示，但 figure 里的 artist 都已经画好了，`savefig` 出来的图和真跑时一模一样。这是这套测试最有价值的部分——图能存下来，就能人眼过一遍或者做像素对比。

## 两条会误导人的输出

### QFontDatabase 找不到字体目录

```shell
QFontDatabase: Cannot find font directory ...\PySide6\lib\fonts.
Note that Qt no longer ships fonts.
```

这是**警告，不是错误**。offscreen 平台想找一份内置字体给文本测量用，找不到就退回系统字体。脚本照样跑完，断言照样执行，图也正常出。别顺着这条信息去装字体或者改 Qt 安装目录，那是白费功夫。

### PowerShell 里看起来"崩了"

在 PowerShell 里跑上面的脚本，很可能只看到那条字体警告，看不到自己 print 的日志，容易误以为脚本崩在开头了。原因是 stdout 被缓冲、stderr 没有，两者到达终端的时机不同。

加 `-u`：

```shell
python -u smoke_test.py
```

这个坑在任何"边跑边看输出"的场景都会遇到，不限于 Qt。习惯上给所有诊断脚本都加 `-u`。

## 配套设计：参数面板用注册表驱动

无头测试要能断言"参数都读对了"，前提是参数读取有一个统一的入口。手写控件的写法做不到这点——每个参数都散在 `self.xxx_edit.text()` 里，测试得逐个访问控件。

改成注册表驱动：每个参数声明成一条数据，UI 按声明生成控件。

```py
PARAM_SPEC = [
    {'name': 'n_bins',  'label': '分箱数',   'type': 'int',   'default': 64,
     'tip': '按 x 分箱的数量，太少趋势线不贴合'},
    {'name': 'k',       'label': 'MAD 倍数', 'type': 'float', 'default': 4.0},
    {'name': 'method',  'label': '方法',     'type': 'choice',
     'choices': ['mad', 'dbscan', 'ridge'], 'default': 'mad'},
    {'name': 'save',    'label': '保存结果', 'type': 'bool',  'default': True},
]
```

建控件时按 type 分派：

```py
for spec in PARAM_SPEC:
    if spec['type'] == 'choice':
        w = QComboBox()
        w.addItems(spec['choices'])
        w.setCurrentText(str(spec['default']))
    elif spec['type'] == 'bool':
        w = QCheckBox()
        w.setChecked(spec['default'])
    else:
        w = QLineEdit(str(spec['default']))
    if spec.get('tip'):
        w.setToolTip(spec['tip'])
    self._widgets[spec['name']] = w
```

读取时按 type 转换，转失败回落默认值并打日志（用户在输入框里打错字不该让整个流程崩掉）：

```py
def _collect_params(self):
    out = {}
    for spec in PARAM_SPEC:
        w = self._widgets[spec['name']]
        raw = w.isChecked() if spec['type'] == 'bool' else (
            w.currentText() if spec['type'] == 'choice' else w.text())
        try:
            if spec['type'] == 'int':
                out[spec['name']] = int(raw)
            elif spec['type'] == 'float':
                out[spec['name']] = float(raw)
            else:
                out[spec['name']] = raw
        except ValueError:
            self._log(f"{spec['label']} 的值 {raw!r} 无法解析，使用默认值 {spec['default']}")
            out[spec['name']] = spec['default']
    return out
```

好处有两层：

- 新增一种处理方式只改注册表，UI 代码一行不动。
- 测试里一句 `tab._collect_params()` 就能拿到全部参数做断言，不用逐个摸控件：

  ```py
  params = tab._collect_params()
  assert params['n_bins'] == 64
  assert params['method'] == 'mad'
  ```

顺带还把 tooltip 也统一了——`tip` 字段写在声明里，不会出现"有的控件有提示有的没有"。

## 小结

这套测试不追求覆盖率，只回答一个问题：**改完之后界面还能构造、流程还能跑完、图还能出来吗**。跑一次几秒钟，比手动点几十次省下来的时间多得多。

前提是业务逻辑不要写在信号槽的 lambda 里、参数不要散在各个控件里。可测性的约束反过来会改善代码结构，注册表驱动的参数面板就是被测试需求逼出来的，最后发现它本身也比手写控件好用。
