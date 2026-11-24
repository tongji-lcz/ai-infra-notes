## 一.前置基础

### 1.核心理念

明确一个观点，所有的优化都是在计算、通信和显存的不可能三角中做取舍，学习时候始终要记住，这个技术牺牲了什么，换取了什么

### 2.编程语言基础

#### 2.1 三层语言

编排层Python、Shell，系统层C/C++，设备层CUDA C++，Triton，AI infra工程师最重要的语言能力是沿调用链跨层追踪问题

#### 2.2 常见命令工具箱

```bash
python --version       # 正在使用哪个 Python
which python           # 这个可执行文件来自哪里
python -m pip --version

g++ --version          # C++ 编译器版本
cmake --version        # 构建系统版本

uname -a               # 内核与系统信息
ps aux                 # 有哪些进程
free -h                # 主机内存使用
df -h                   # 文件系统容量

nvidia-smi             # 驱动可见的 GPU 与进程
nvcc --version         # CUDA Toolkit 中的编译器版本
```

#### 2.3 python

1.Python 变量更准确的说法是“**名字绑定到对象**”，写函数时自问一句：我这个函数是“改造了传进来的东西”，还是“做了个新东西返回出去”？这两者的语义完全不同，接口设计时要想清楚

2.常见不可变对象包括 `int`、`float`、`bool`、`str`、`bytes` 和元素均不可变时的 `tuple`；常见可变对象包括 `list`、`dict`、`set` 和大多数自定义实例。

3.|表示联合类型，int|string表达的是，int和string类型都可以

4.

```python
# 错误：同一个列表会被多次调用共享
def collect_bad(x: int, result: list[int] = []) -> list[int]:
    result.append(x)
    return result

collect_bad(1) -> [1]        id=1459917953216
collect_bad(2) -> [1, 2]     id=1459917953216
collect_bad(3) -> [1, 2, 3]  id=1459917953216     ← 三次是同一个列表！

#这里的严重问题在于局部变量 result 每次调用都是新创建的，并非同一个变量，但每次它都绑定到同一个列表对象（存在 __defaults__ 里的那个），累加发生在那个对象里，不是发生在变量里

# 正确：用 None 表示“未提供”
def collect(x: int, result: list[int] | None = None) -> list[int]:
    if result is None:
        result = []
    result.append(x)
    return result

```

这部分代码里需要注意的是，if result is None和if not result可完全不是一个东西，后者只是判断了，result为不为真，例如[]是false但是不是None

5.深拷贝和浅拷贝

```python
import copy

src = [[1], [2]]
shallow = src.copy()           # 只复制最外层列表
deep = copy.deepcopy(src)      # 递归复制内部对象

shallow[0].append(9)
print(src)                     # [[1, 9], [2]]
print(deep)                    # [[1], [2]]

shallow = src.copy()      # 新外层 S，但 S[0] 还是 B，S[1] 还是 C
deep = copy.deepcopy(src) # 新外层 + 新内层 B'、C'
```

6.Python 按 LEGB 顺序查找名字：Local（当前函数内部，函数参数、函数里赋值的变量）→ Enclosing（外层函数，闭包，内层函数看到外层函数的变量）→ Global（模块）→ Builtins（内置）。

```python
x = 1
def f():
    print(x)     # 报 UnboundLocalError！
    x = 2        # 因为函数里出现了对 x 的赋值，x 在整个函数内都被判定为 local
    
#核心思想，函数里只要给某个名字赋过值，它在整个函数里就是局部变量，和"先读到未赋值的局部变量"冲突就报错
```

在 Python 里，函数本身也是一个"值"，可以赋值给变量、放进列表、被返回；只有写出名字(...) 时才真正执行它。 

```python
from collections.abc import Callable

def make_multiplier(scale: float) -> Callable[[float], float]:
    def multiply(x: float) -> float:
        return x * scale
    return multiply

double = make_multiplier(2.0)
print(double(3.0))  # 6.0
#double就是一个函数对象，类型是function
#可以理解成：make_multiplier(2.0) 造出了一台"乘以 2"的机器，double 是这台机器的名字；double(3.0) 才是把 3.0 塞进机器里运行。再调用一次 make_multiplier(10) 会造出另一台"乘以 10"的机器，两台互不影响，各自记着自己的 scale。

-----------------------------------------------------------------------------

# 三个函数最终都读取同一个 i，结果都是 2
bad = [lambda: i for i in range(3)]

# 用默认参数在每轮定义时保存当前值
good = [lambda i=i: i for i in range(3)]
print([fn() for fn in good])  # [0, 1, 2]

```

lambda是匿名函数的简写，格式是lambda参数: 表达式，这里bad是一个三个函数对象的list列表
