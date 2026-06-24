# 使用python装饰器

## 基础装饰器

```py
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        time.sleep(5)
        end = time.time()
        print(f"duration:{end-start}")
        return result
    return wrapper

@timer
def hello():
    print("hello world")

hello()
```
输出结果：
```shell
(base) PS C:\Users\86189\Desktop\mianjing\mianjing\python> python ex.py
hello world
duration:5.001402854919434
```

## 带有参数的装饰器

添加一层装饰器来实现参数的传入。
```py
def counter(times):
    def para(func):
        def wrapper(*args, **kwargs):
            for i in range(times):
                print(f"call: {i} times")
                result = func(*args, **kwargs)
            return result
        return wrapper
    return para

@counter(3)
def hello():
    print("hello world")

hello()
```

运行结果如下:
```shell
(base) PS C:\Users\86189\Desktop\mianjing\mianjing\python> python ex.py
call: 0 times
hello world
call: 1 times
hello world
call: 2 times
hello world
```