# 如何通过API进行终端的调用

## 完成项目的初始化

这里假设你已经存在`app.py`的主结构，我们通过一个目录下的`routes.py`完成全部api的编写。
首先定义基本的结构：
```py
# routes.py
from flask import blueprint, jsonify, request

cmd_bp = Blueprint("cmd_bp",__name__)

@cmd_bp.route("/send", methods=["POST"])
def send_cmd():
    data = request.get_json()
    command = data.get("command", "bash")
```

## 直接操作CLI

之后开始进入正题，为了进行CLI的调用，我们需要通过API写入终端，再从终端中读取信息，这里我们使用`subProcess.run()`实现。
这个方法启动一个子进程，其中会进行命令或者文件的执行，当选择`shell=True`，他就可以执行shell命令。在默认`sshell=False`的情况下，只能执行可执行文件，如`.bat`,`.exe`之类的。
因此如果不设置`shell=True`，当你试图执行`ls`的时候，他就会在系统当前目录下寻找名为`ls`的可执行文件，也就得不到想要的效果了。
```py
import select
import subprocess

try:
    result = subprocess.run(
        command,
        shell=True,
        capture_output=True,
        text=True
    )

    return jsonify({
        "stdout": result.stdout,    # 标准输出
        "stderr": result.stderr,    # 错误输出
    }), 200

except Exception as e:
    return jsonify({
        "stderr": str(e)
    }), 500
```

## 打开模拟TTY

通过这样的子进程进行命令的执行，即可快速获取CLI返回的结果。但在特殊的情况下，有些命令，系统类的如`sudo`，`vim`，应用类的则数不胜数，都需要必须在真实终端的情况下，才会真正返回结果。他们会检测自己是否运行在 **真实终端（TTY）** 中。
如果你只用 `subprocess.PIPE`，程序会发现 `stdout` 是一个“管道”而不是“屏幕”，从而拒绝执行或改变行为（比如不再输出彩色文字，或直接报错 `standard in must be a tty`）。
在Linux环境下，我们可以使用`openpty()`来生成虚拟模拟终端。
```py
import os
import pty

master_fd, slave_fd = pty.openpty()

try:
    process = subprocess.Popen(
        command,
        shell=True,
        stdin=slave_fd,
        stdout=slave_fd,
        stderr=slave_fd,
        close_fds=True,
        text=True
    )

    os.close(slave_fd)
    output = ""
    while True:
        r, _, _ = select.select([master_fd], [], [], 0.1)
        if master_fd in r:
            data = os.read(master_fd, 1024).decode("utf-8")
            if not data:
                break
            output += data

    return jsonify({
        "output": output,
        "status": "completed"
    }), 200

except Exception as e:
    return jsonify({"error": str(e)}), 500
finally:
    os.close(master_fd)
```

当然，只是通过一个`slave_fd`来进行虚拟终端的模拟可能过于潦草，对于一些对于终端侦测比较严格的应用来说，参数的缺失可能导致其拒绝答复，所以想要让终端更像真实的终端，可以帮他进行参数的设置。
其中，`path`中的内容才能使得终端的命令正常执行，因此为了让终端能够找到所有你在真实终端执行的命令，就必须将所有环境变量都传入模拟终端，因此我们使用`os.environ.copy()`，同时为其附上终端UI的参数。

```py
set_winsize(slave_fd, 40, 100)

env = os.environ.copy()
env["TERM"] = "xterm-256color"
env["COLUMNS"] = "80"
env["LINES"] = "24"

process = subprocess.Popen(
        command,
        shell=True,
        stdin=slave_fd,
        stdout=slave_fd,
        stderr=slave_fd,
        close_fds=True,
        text=True
    )
```

这样就会使得终端的参数更加完整，从而让各种终端命令通过`python`正常执行。

## 流式输出

有一些CLI命令可能启动了某些应用，尤其是可能启动了LLM或者agent，他们需要大量的思考时间和流式的输出。这种情况下我们就需要进行流式的返回，通过编写一个stream函数，将获取的`data`从叠加变为`yield`，从而动态返回LLM生成的内容。
(此处使用SSE规范便于前端获取数据进行提取、加工和展示)

```py
def stream(command):
    try:
        while True:
            r, _, _ = select.select([master_fd], [], [], 0.1)
            if master_fd in r:
                try:
                    data = os.read(master_fd, 1024)
                    if not data:
                        break
                    yield f"data: {json.dumps(data.decode("utf-8"))}\n\n"
                except OSError:
                    break
            if process.poll() is not None:
                break
            
    finally:
        os.close(master_fd)
        if process.poll() is None:
            process.terminate()
```

其中`process.poll()`返回进程是否仍然在运行，如果在运行则会返回`None`，通过这种情况防止其未发出关闭信号但已经停止运行的情况。

```py
return Response(stream(command), mimetype="text/event-stream")
```

完整代码如下：

```py
from flask import blueprint, jsonify, request
import os
import pty

cmd_bp = Blueprint("cmd_bp",__name__)

@cmd_bp.route("/send", methods=["POST"])
def send_cmd():
    data = request.get_json()
    command = data.get("command", "bash")

    master_fd, slave_fd = pty.openpty()

    set_winsize(slave_fd, 40, 100)
    env = os.environ.copy()
    env["TERM"] = "xterm-256color"
    env["COLUMNS"] = "80"
    env["LINES"] = "24"

    process = subprocess.Popen(
            command,
            shell=True,
            stdin=slave_fd,
            stdout=slave_fd,
            stderr=slave_fd,
            close_fds=True
        )

    os.close(slave_fd)

    def stream(command):
        try:
            while True:
                r, _, _ = select.select([master_fd], [], [], 0.1)
                if master_fd in r:
                    try:
                        data = os.read(master_fd, 1024)
                        if not data:
                            break
                        d_data = json.dumps(data.decode('utf-8'))
                        yield f"data: {d_data}\n\n"
                    except OSError:
                        break
                if process.poll() is not None:
                    break
                
        finally:
            os.close(master_fd)
            if process.poll() is None:
                process.terminate()

    return Response(stream(command), mimetype="text/event-stream")
```