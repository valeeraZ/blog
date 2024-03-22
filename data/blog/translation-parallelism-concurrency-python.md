---
title: '翻译：Python的并发与并行'
date: '2024-03-20'
lastmod: '2024-03-20'
tags: ['Python']
draft: false
images: ['/static/images/fastapi_testing.png']
layout: PostLayout
summary: 'Python的线程，异步，多进程'
---

-- 译者：[Wenzhuo Zhao](https://www.linkedin.com/in/wenzhuo-zhao-cs-dev/), 原文链接：[Speeding Up Python with Concurrency, Parallelism, and asyncio](https://testdriven.io/blog/concurrency-parallelism-asyncio/)。代码链接：[https://github.com/based-jace/concurrency-parallelism-and-asyncio/tree/master](https://github.com/based-jace/concurrency-parallelism-and-asyncio/tree/master)

并发和并行是什么，它们如何应用于 Python？  
应用程序变慢的原因有很多。有时这是由于算法设计不佳或选择错误的数据结构。然而，有时候这是由于我们无法控制的力量，比如硬件限制或网络的怪癖。这就是并发和并行所适用的地方。它们允许你的程序同时做多个事情，要么在同一时间内进行，要么尽可能少地浪费等待繁忙任务所需的时间。  
无论你正在处理外部网络资源、从多个文件中读取和写入数据，还是需要使用计算密集型函数以不同参数多次运行，在本文中都将帮助你最大化代码效率和速度。  
首先，我们将深入了解并发和并行以及它们如何与 Python 领域中使用标准库（例如线程、进程和异步 IO）相结合。本文最后一部分将比较 Python 对 async/await 实现方式与其他语言对其实现方式之间的差异。  
您可以在[GitHub 上 concurrency-parallelism-and-asyncio](https://github.com/based-jace/concurrency-parallelism-and-asyncio)的代码库中找到本文中所有示例代码。

## 目标

通过本文的阅读，您应该能够回答以下问题：

- 什么是并发？(concurrency)
- 什么是线程？(thread)
- 当某物是非阻塞时(non-blocking)，这意味着什么？
- 什么是事件循环？(event loop)
- 什么是回调函数？(callback function)
- 为何 asyncio 方法总比 threading 方法快一点？
- 在何时应使用线程，在何时应使用 asyncio ？
- 什么是并行性？
- 并发和并行性之间有何区别?
- 是否可以将 asyncio 与多进程结合使用?
- 在何时应使用多进程而不是 asyncio 或 threading ?
- 多进程、asyncio 和 concurrency.futures 之间有何区别?
- 如何用 pytest 测试 asyncio？
  译者注：本文将分为多个部分进行翻译，每个部分都会回答上述问题中的一部分。

## 并发

并发是什么？  
对于并发的有效定义是“能够同时执行多个任务”。然而，这有点误导人，因为这些任务可能实际上不会在完全相同的时间内执行。相反，一个进程可能会启动，然后一旦它正在等待特定指令完成时切换到新的任务，只有当它不再等待时才回来。一旦一个任务完成了，它又切换到未完成的任务直到所有任务都被执行完毕。任务异步开始、异步执行和异步结束。

如果这对你来说很困惑，我们可以用一个类比来思考：假设你想要制作一份三明治。首先，你会把培根放入中小火的平底锅中煎熟。当培根在煮的时候，你可以拿出番茄和生菜开始准备（清洗和切割）它们。与此同时，你不断地检查并偶尔翻转培根。
到了这个阶段，你已经开始了一个任务，并且在此期间又开始并完成了其他两个任务，而所有这些都是在等待第一个任务完成的同时进行的。

最终，你把面包放进了烤面包机。当它在烤的时候，你继续检查培根。当一块块完成后，你把它们拿出来放在盘子上。一旦你的面包烤好了，你可以涂上自己喜欢的三明治酱料，然后开始叠加番茄、生菜，最后是熟透的培根。只有当所有东西都做好、准备好并且叠加好之后，才能将最后一片吐司放在三明治上，并切开（可选），然后吃掉。

由于需要同时执行多个任务，在制作三明治时本质上是一个并发过程，即使您没有完全专注于每个任务。为了方便起见，在接下来的部分中我们将简称这种形式为"并发"（Concurrency）。我们稍后会对此进行区分。

因此，并发非常适用于 I/O 密集型进程——涉及等待网络请求或文件读写操作的任务。

在 Python 中，有几种不同的方法可以实现并发处理。首先我们来看看线程库（threading library）。

在本节中的示例中，我们将构建一个小型的 Python 程序，从 Binary Jazz 的 Genrenator API 随机获取五个音乐流派的名字，将每个流派的名字单词打印到屏幕上，并将每个流派放入单独的文件中。

要在 Python 中使用线程，您唯一需要导入的是 threading 模块。但是为了这个示例，我还导入了 urllib 模块以处理 HTTP 请求，time 模块来确定函数完成所需的时间，并且导入 json 模块以便于将从 Genrenator API 返回的 JSON 数据进行转换。

```python
def write_genre(file_name):
    """
    Uses genrenator from binaryjazz.us to write a random genre to the
    name of the given file
    """

    req = Request("https://binaryjazz.us/wp-json/genrenator/v1/genre/", headers={"User-Agent": "Mozilla/5.0"})
    genre = json.load(urlopen(req))

    with open(file_name, "w") as new_file:
        print(f"Writing '{genre}' to '{file_name}'...")
        new_file.write(genre)
```

我们真正感兴趣的是下一部分，也就是实际使用线程的地方：

```python
threads = []

for i in range(5):
    thread = threading.Thread(
        target=write_genre,
        args=[f"./threading/new_file{i}.txt"]
    )
    thread.start()
    threads.append(thread)

for thread in threads:
    thread.join()
```

我们首先从一个列表开始。然后，我们继续迭代五次，每次创建一个新的线程。接下来，我们启动每个线程，将其附加到我们的`threads`列表中，然后最后再次迭代我们的列表以加入每个线程。

解释：在 Python 中创建线程很容易。

要创建一个新的线程，请使用`threading.Thread()`。您可以传递`kwarg`（关键字参数）target，并为其赋予您想要在该线程上运行的函数值。但是只传递函数名而不是它的值（也就是说，在这种情况下写`write_genre`而不是`write_genre()`）。要传递参数，请传入`kwargs`（它接受包含您 kwargs 的字典）或者`args`（它接受包含您 args 的可迭代对象——在本例中为列表）。

然而，创建一个线程并不等同于启动一个线程。要启动你的线程，请使用`{你的线程名称}.start()`。启动一个线程意味着“开始执行”。

最后，用`thread.join()`合并各个线程时，我们所做的只是确保该线程在继续执行代码之前已经完成了任务。

## 线程

线程是一种让计算机将单个进程/程序分解为许多轻量级片段并并行执行的方式。有点令人困惑的是，Python 的标准线程实现由于全局解释器锁（GIL）的存在，限制了线程只能一次执行一个。GIL 是必要的，因为 CPython（Python 的默认实现）的内存管理不是线程安全的。由于这个限制，在 Python 中进行线程操作是并发但不是并行的。为了绕过这个问题，Python 提供了一个独立的 multiprocessing 模块，它没有受到 GIL 的限制，并可以启动单独的进程来实现代码的并行执行。使用 multiprocessing 模块与使用 threading 模块几乎完全相同。

下面是不使用线程，而是使用 for 循环迭代 5 次运行`write_genre`函数的示例：

```python
print("Starting...")
start = time.time()

for i in range(5):
    write_genre(f"./sync/new_file{i}.txt")

end = time.time()
print(f"Time to complete synchronous read/writes: {round(end - start, 2)} seconds")
```

我们使用`time`库来计算时间，得到的输出是：

```
Starting...
Writing "binary indoremix" to "./sync/new_file0.txt"...
Writing "slavic aggro polka fusion" to "./sync/new_file1.txt"...
Writing "israeli new wave" to "./sync/new_file2.txt"...
Writing "byzantine motown" to "./sync/new_file3.txt"...
Writing "dutch hate industrialtune" to "./sync/new_file4.txt"...
Time to complete synchronous read/writes: 1.42 seconds
```

现在让我们运行使用线程的版本：

```
Starting...
Writing "college k-dubstep" to "./threading/new_file2.txt"...
Writing "swiss dirt" to "./threading/new_file0.txt"...
Writing "bop idol alternative" to "./threading/new_file4.txt"...
Writing "ethertrio" to "./threading/new_file1.txt"...
Writing "beach aust shanty français" to "./threading/new_file3.txt"...
Time to complete threading read/writes: 0.77 seconds
```

您可能注意到的第一件事是文件的输入未按顺序完成，而是按照顺序 2 - 0 - 4 - 1 - 3。这是因为线程的异步性质：当一个函数等待时，另一个函数开始，依此类推。因为我们能够在等待其他任务完成的同时继续执行任务（由于网络或文件 I/O 操作），因此您可能还注意到我们将时间大约减少了一半：0.77 秒。虽然现在看起来可能不是很多，但很容易想象构建一个需要向文件写入更多数据或与更复杂的 Web 服务交互的 Web 应用程序的真实情况。

## Asyncio 异步

让我们来看一个使用 asyncio 的例子。对于这种方法，我们将使用 pip 安装 aiohttp。这将允许我们进行非阻塞请求并使用即将介绍的 async/await 语法接收响应。它还具有一个函数的额外好处，可以在不需要导入 json 库的情况下转换 JSON 响应。我们还会安装和导入 aiofiles，它允许非阻塞文件操作。除了 aiohttp 和 aiofiles 之外，还要导入 Python 标准库中附带的 asyncio 模块。

_“非阻塞”意味着程序将允许其他线程在等待时继续运行。这与“阻塞”代码相反，“阻塞”代码完全停止程序的执行。正常的同步 I/O 操作会受到此限制。_

让我们看一下 asyncio 示例中的 write_genre 函数的异步版本：

```python
async def write_genre(file_name):
    """
    Uses genrenator from binaryjazz.us to write a random genre to the
    name of the given file
    """

    async with aiohttp.ClientSession() as session:
        async with session.get("https://binaryjazz.us/wp-json/genrenator/v1/genre/") as response:
            genre = await response.json()

    async with aiofiles.open(file_name, "w") as new_file:
        print(f'Writing "{genre}" to "{file_name}"...')
        await new_file.write(genre)
```

对于那些不熟悉许多其他现代语言中的`async/await`语法的人来说，`async`声明了一个函数、`for`循环或`with`语句必须以异步方式使用。要调用一个`async`函数，你必须从另一个`async`函数中使用`await`关键字，或者直接从事件循环中调用`create_task()`，可以通过 `asyncio.get_event_loop()` 获取 -- 即 `loop = asyncio.get_event_loop()`。

此外：

- `async with` 允许等待异步响应和文件操作。
- `async for` （此处未使用）迭代异步流。

## The Event Loop 事件循环

事件循环是异步编程固有的构造，允许异步执行任务。当您阅读本文时，我可以有把握地假设您可能不太熟悉这个概念。但是，即使您从未编写过异步应用程序，每次使用计算机时您都会有事件循环的经验。无论您的计算机是在监听键盘输入、玩在线多人游戏，还是在后台复制文件时浏览 Reddit，事件循环都是让一切顺利高效运行的驱动力。从本质上讲，事件循环是一个等待触发器的过程，然后在满足这些触发器后执行特定的（编程的）操作。它们通常返回某种“promise”（JavaScript 语法）或“future”（Python 语法）来表示任务已添加。一旦任务完成，promise 或 future 返回一个从被调用函数传回的值（假设该函数确实返回一个值）。

> 执行一个函数以响应另一个函数的想法称为“回调”。

我们使用`async with`来异步打开客户端会话。`aiohttp.ClientSession()`类允许我们发出 HTTP 请求并保持与源的连接，而不阻塞代码的执行。然后，我们向 Genrenator API 发出异步请求，并等待 JSON 响应（一个随机音乐流派）。在下一行中，我们再次使用`async with`和`aiofiles`库异步打开一个新文件以将新的流派写入其中。我们先打印流派，然后将其写入文件中。

与常规 Python 脚本不同，使用 asyncio 编程几乎强制\*使用某种“main”函数。这是因为您需要使用“async”关键字才能使用“await”语法，而“await”语法是实际运行其他异步函数的唯一方法。

```python
async def main():
    tasks = []

    for i in range(5):
        tasks.append(write_genre(f"./async/new_file{i}.txt"))

    await asyncio.gather(*tasks)

if __name__ == "__main__":
    asyncio.run(main())
```

正如您所看到的，我们用“async”声明了它。然后我们创建一个名为“tasks”的空列表来容纳我们的异步任务（调用 Genrenator 和文件 I/O）。我们将任务添加到列表中，但它们实际上尚未运行。在我们使用 `await asyncio.gather(*tasks)` 安排调用之前，这些调用实际上不会进行。这将运行列表中的所有任务并等待它们完成，然后再继续程序的其余部分。最后，我们使用 `asyncio.run(main())` 来运行我们的“main”函数。 `.run()` 函数是我们程序的入口点，通常每个进程只应调用一次。

> 对于那些不熟悉的人来说，任务前面的 \* 称为“参数解包”。正如听起来的那样，它将我们的列表解压缩为函数的一系列参数。在本例中，我们的函数是 `asyncio.gather()`。

```
Writing "albuquerque fiddlehaus" to "./async/new_file1.txt"...
Writing "euroreggaebop" to "./async/new_file2.txt"...
Writing "shoedisco" to "./async/new_file0.txt"...
Writing "russiagaze" to "./async/new_file4.txt"...
Writing "alternative xylophone" to "./async/new_file3.txt"...
Time to complete asyncio read/writes: 0.71 seconds
```

我们看到它甚至更快。而且，一般来说，asyncio 方法总是比线程方法稍微快一点。这是因为当我们使用"await"语法时，实际上告诉程序“等一下，我马上回来”，但我们的程序会记录下完成任务所需的时间。一旦完成了，我们的程序就会知道，并在能够继续之后立即恢复执行。Python 中的线程允许异步操作，但如果有尚未准备好的线程可以继续运行，则理论上我们的程序可能会跳过不同的线程并浪费时间。

那么什么时候应该使用线程，什么时候应该使用 asyncio？

当您编写新代码时，请使用 asyncio。如果您需要与较旧的库或不支持 asyncio 的库进行交互，那么使用线程可能会更好。

## 使用 pytest 测试 asyncio

使用 `pytest` 测试异步函数与测试同步函数一样简单。只需使用 pip 安装 `pytest-asyncio` 包，使用 `async` 关键字标记您的测试，然后应用一个装饰器，让 pytest 知道它是异步的： `@pytest.mark.asyncio` 。让我们看一个例子。

```python
import asyncio

async def say_hello(name: str):
    """ Sleeps for two seconds, then prints 'Hello, {{ name }}!' """
    try:
        if type(name) != str:
            raise TypeError("'name' must be a string")
        if name == "":
            raise ValueError("'name' cannot be empty")
    except (TypeError, ValueError):
        raise

    print("Sleeping...")
    await asyncio.sleep(2)
    print(f"Hello, {name}!")
```

> asyncio.sleep() 和 time.sleep() 之间的区别在于 asyncio.sleep() 是非阻塞的。

现在我们用 pytest 来测试一下。

```python
import pytest # Note: pytest-asyncio does not require a separate import

from hello_asyncio import say_hello

@pytest.mark.parametrize("name", [
    "Robert Paulson",
    "Seven of Nine",
    "x Æ a-12"
])
@pytest.mark.asyncio
async def test_say_hello(name):
    await say_hello(name)
```

- @pytest.mark.asyncio 装饰器让 pytest 异步工作
- 我们`await`处理我们的异步函数，就像我们在测试之外运行它一样

接下来我们将编写一些输入错误的测试，让我们创建一个名为 TestSayHelloThrowsExceptions 的类：

```python

class TestSayHelloThrowsExceptions:
    @pytest.mark.parametrize("name", [
        "",
    ])
    @pytest.mark.asyncio
    async def test_say_hello_value_error(self, name):
        with pytest.raises(ValueError):
            await say_hello(name)

    @pytest.mark.parametrize("name", [
        19,
        {"name", "Diane"},
        []
    ])
    @pytest.mark.asyncio
    async def test_say_hello_type_error(self, name):
        with pytest.raises(TypeError):
            await say_hello(name)
```

运行测试：

```
pytest -v
...
collected 7 items

test_hello_asyncio.py::test_say_hello[Robert Paulson] PASSED                                    [ 14%]
test_hello_asyncio.py::test_say_hello[Seven of Nine] PASSED                                     [ 28%]
test_hello_asyncio.py::test_say_hello[x \xc6 a-12] PASSED                                       [ 42%]
test_hello_asyncio.py::TestSayHelloThrowsExceptions::test_say_hello_value_error[] PASSED        [ 57%]
test_hello_asyncio.py::TestSayHelloThrowsExceptions::test_say_hello_type_error[19] PASSED       [ 71%]
test_hello_asyncio.py::TestSayHelloThrowsExceptions::test_say_hello_type_error[name1] PASSED    [ 85%]
test_hello_asyncio.py::TestSayHelloThrowsExceptions::test_say_hello_type_error[name2] PASSED    [100%]
```

## 不使用 pytest-asyncio

除了 pytest-asyncio 之外，您可以创建一个 pytest fixture 来生成一个异步事件循环：

```python
import asyncio
import pytest

from hello_asyncio import say_hello


@pytest.fixture
def event_loop():
    loop = asyncio.get_event_loop()
    yield loop
```

然后，您可以像正常的同步测试一样创建测试，而不使用 async / await 语法：

```python
@pytest.mark.parametrize("name", [
    "Robert Paulson",
    "Seven of Nine",
    "x Æ a-12"
])
def test_say_hello(event_loop, name):
    event_loop.run_until_complete(say_hello(name))


class TestSayHelloThrowsExceptions:
    @pytest.mark.parametrize("name", [
        "",
    ])
    def test_say_hello_value_error(self, event_loop, name):
        with pytest.raises(ValueError):
            event_loop.run_until_complete(say_hello(name))

    @pytest.mark.parametrize("name", [
        19,
        {"name", "Diane"},
        []
    ])
    def test_say_hello_type_error(self, event_loop, name):
        with pytest.raises(TypeError):
            event_loop.run_until_complete(say_hello(name))
```

## Parallelism 并行性

并行性与并发性密切相关。事实上，并行性是并发性的一个子集：并发进程同时执行多个任务，无论它们是否被转移全部注意力，而并行进程在物理上同时执行多个任务。一个很好的例子是一边开车，一边听音乐，一边吃我们在上一节中制作的三明治。

现在让我们来看一下如何在 Python 中实现这个。我们可以使用 multiprocessing 库，但是让我们改用 concurrent.futures 库--我们不需要手动管理进程数量。多进程最大的好处是在执行多个 CPU 密集型任务时，举个例子，让我们计算从 1000000 到 1000016 之间的平方数。

```python
import concurrent.futures
import time


if __name__ == "__main__":
    pow_list = [i for i in range(1000000, 1000016)]

    print("Starting...")
    start = time.time()

    with concurrent.futures.ProcessPoolExecutor() as executor:
        futures = [executor.submit(pow, i, i) for i in pow_list]

    for f in concurrent.futures.as_completed(futures):
        print("okay")

    end = time.time()
    print(f"Time to complete: {round(end - start, 2)}")
```

看一下我们的 main 函数，我们创建一个从 1000000 到 1000016 的列表，我们使用`concurrent.futures`打开一个`ProcessPoolExecutor`，并且我们使用列表推导和`ProcessPoolExecutor().submit()`来开始执行我们的进程并将它们放入一个名为"futures"的列表中。

> 如果我们想使用线程，也可以使用`ThreadPoolExecutor()` -- `concurrent.futures`非常灵活。

这就是异步性的体现：「results」列表实际上并不包含运行函数的结果。相反，它包含了类似于 JavaScript 中「promises」概念的「futures」。为了让程序能够继续运行，我们会得到代表值占位符的这些 futures。如果我们尝试打印 future，则根据其是否已经完成运行，我们将得到一个状态为「pending」或者「finished」的返回值。一旦它完成了，我们可以使用 `var.result()` 来获取返回值（假设有返回值）。在本例中，我们的 var 将是 「result」。

下面是一个不使用并行的同步版本：

```python
print("Starting...")
start = time.time()

for i in range(1000000, 1000016):
    pow(i, i)
    print("okay")

end = time.time()
print(f"Time to complete: {round(end - start, 2)}")
```

比较一下两者的运行时间，你会发现并行的版本更快：同步版本需要 54.64 秒，而并行版本只需要 6.24 秒。

那么如果我们使用线程来代替会发生什么呢？可以猜到——这不会比同步执行快多少。事实上，它可能会更慢，因为它仍然需要一点时间和精力来启动新线程。我们用 `ThreadPoolExecutor()` 替换 `ProcessPoolExecutor()`，需要 53.83 秒。

正如我之前提到的，线程允许您的应用程序在其他任务等待时专注于新任务。在这种情况下，我们的系统从不闲着。另一方面，多进程会启动全新的服务，通常在单独的 CPU 核心上运行，并准备与脚本中正在进行的任何其他操作完全同步地执行您要求它做的事情。这就是为什么多进程版本大约只需要 1/9 左右时间是有道理的——因为我的 CPU 有 8 个核心。

## 译者的注解和总结

- 线程 threading 用于**并发**执行任务
- 多进程 multiprocessing 用于**并行**执行任务
- asyncio 用于与 Python 解释器管理的**协程并发**运行任务

用表格来比较：

| 库                 | 类/方法             | 类型         |
| ------------------ | ------------------- | ------------ |
| threading          | thread              | 并发         |
| concurrent.futures | ThreadPoolExecutor  | 并发         |
| asyncio            | gather              | 通过协程并发 |
| multiprocessing    | Pool                | 并行         |
| concurrent.futures | ProcessPoolExecutor | 并行         |

### CPU-bound 和 I/O-bound

受 CPU 限制的任务是 CPU 密集型的。例如，数学计算受 CPU 限制，因为计算能力随着计算机处理器数量的增加而增加。并行性适用于 CPU 密集型任务。理论上，如果一个任务被划分为 n 个子任务，那么这 n 个子任务中的每一个都可以并行运行，从而有效地将时间减少到原来非并行任务的 1/n。CPU 密集型任务的最佳示例是数据科学。数据科学家处理大量数据。对于数据预处理，他们可以将数据分成多个批次并并行运行，从而有效减少总处理时间。增加核心数量可以加快处理速度。

Web 抓取受 IO 限制。因为该任务对 CPU 的影响很小，因为大部分时间都花在网络读写上。其他常见的 IO 密集型任务包括数据库调用以及向磁盘读取和写入文件。 Web 应用程序（例如 Django 和 Flask）是 IO 密集型应用程序。对于 IO 密集型任务来说，并发是首选，因为您可以在获取 IO 资源时执行其他操作。

## 例子：将 asyncio 和多进程结合使用

_如果我需要将大量 I/O 操作与繁重的计算结合起来怎么办？_

假设您需要抓取 100 个网页，然后需要将网页内容保存在文件中，我们可以通过让每个进程抓取一小部分页面来分离每个计算机核心的计算能力。

我们可以用 asyncio 和 beautifulsoup 来抓取网页，然后用多进程来保存文件。

```python
import asyncio                         # Gives us async/await
import concurrent.futures              # Allows creating new processes
import time
from math import floor                 # Helps divide up our requests evenly across our CPU cores
from multiprocessing import cpu_count  # Returns our number of CPU cores

import aiofiles                        # For asynchronously performing file I/O operations
import aiohttp                         # For asynchronously making HTTP requests
from bs4 import BeautifulSoup          # For easy webpage scraping
```

首先，我们将创建一个异步函数，向维基百科发出请求抓取随机页面。使用 BeautifulSoup 抓取每个页面的标题，然后将其写入到给定的文件中，用制表符分隔每个标题。该函数将采用两个参数：num_pages - 请求和抓取标题的页数，output_file - 将标题附加到的文件

```python
async def get_and_scrape_pages(num_pages: int, output_file: str):
    """
    Makes {{ num_pages }} requests to Wikipedia to receive {{ num_pages }} random
    articles, then scrapes each page for its title and appends it to {{ output_file }},
    separating each title with a tab: "\\t"

    #### Arguments
    ---
    num_pages: int -
        Number of random Wikipedia pages to request and scrape

    output_file: str -
        File to append titles to
    """
    async with \
    aiohttp.ClientSession() as client, \
    aiofiles.open(output_file, "a+", encoding="utf-8") as f:

        for _ in range(num_pages):
            async with client.get("https://en.wikipedia.org/wiki/Special:Random") as response:
                if response.status > 399:
                    # I was getting a 429 Too Many Requests at a higher volume of requests
                    response.raise_for_status()

                page = await response.text()
                soup = BeautifulSoup(page, features="html.parser")
                title = soup.find("h1").text

                await f.write(title + "\t")

        await f.write("\n")
```

我们异步地打开`ClientSession`和`aiofiles.open()`，然后迭代`num_pages`次，每次都请求一个维基百科的随机页面。我们使用`BeautifulSoup`从页面中提取标题，然后将其写入文件。最后，我们将标题写入文件。

接下来写一个同步的函数调用这个异步的 get_and_scrape_pages 函数。

```python
def start_scraping(num_pages: int, output_file: str, i: int):
    """ Starts an async process for requesting and scraping Wikipedia pages """
    print(f"Process {i} starting...")
    asyncio.run(get_and_scrape_pages(num_pages, output_file))
    print(f"Process {i} finished.")
```

写一个 main 函数，我们将使用`concurrent.futures`来创建多个进程，每个进程都会调用`start_scraping`函数。

```python

def main():
    NUM_PAGES = 100 # Number of pages to scrape altogether 总共要抓取的页面数
    NUM_CORES = cpu_count() # Our number of CPU cores (including logical cores) 我们的CPU核心数量（包括逻辑核心）
    OUTPUT_FILE = "./wiki_titles.tsv" # File to append our scraped titles to 把我们抓取的标题写入到这个文件里
    PAGES_PER_CORE = floor(NUM_PAGES / NUM_CORES) # 每个cpu核心要抓取的页面数
    PAGES_FOR_FINAL_CORE = PAGES_PER_CORE + NUM_PAGES % PAGES_PER_CORE # 最后一个cpu核心要抓取的页面数（就是剩下的页面数，因为不能整除）

    futures = []

    with concurrent.futures.ProcessPoolExecutor(NUM_CORES) as executor:
        # 每个核心都要处理一部分页面
        for i in range(NUM_CORES - 1):
            new_future = executor.submit(
                start_scraping, # Function to perform
                # v Arguments v
                num_pages=PAGES_PER_CORE,
                output_file=OUTPUT_FILE,
                i=i
            )
            futures.append(new_future)

        # 单独处理最后一个核心
        futures.append(
            executor.submit(
                start_scraping,
                num_pages=PAGES_FOR_FINAL_CORE,
                output_file=OUTPUT_FILE,
                i=NUM_CORES - 1
            )
        )

    concurrent.futures.wait(futures)
```

在 main 函数中，我们使用`ProcessPoolExecutor`创建了一个进程池，每个进程都会调用`start_scraping`函数。我们将`NUM_PAGES`分成`NUM_CORES`份，每个进程都会处理一部分页面。最后一个进程将处理剩下的页面。

最后让我们运行 main 函数。

```python
if __name__ == "__main__":
    start = time.time()
    main()
    print(f"Time to complete: {round(time.time() - start, 2)} seconds.")
```

使用我的 8 核 CPU 运行该程序后（以及基准测试代码）:[代码链接](https://github.com/based-jace/concurrency-parallelism-and-asyncio/tree/master/code_examples/asyncio_and_multiprocessing)

- 如上的版本（asyncio+concurrent.futures）：5.65 秒
- 不用 asyncio，只用 concurrent.futures：8.87 秒
- 只用 asyncio，不用 concurrent.futures：47.92 秒
- 只用同步代码：88.86 秒

## 总结: 何时使用多进程、何时使用异步或线程

- 当您需要进行大量繁重计算并且可以将它们分开时，请使用多处理。
- 当您执行 I/O 操作（与外部资源通信或从文件读取/写入文件）时，请使用 asyncio 或线程。
- 多处理和 asyncio 可以一起使用，但有一个好办法是在线程/使用 asyncio 之前分叉一个进程，而不是相反 - 与进程相比，线程相对便宜。

## 附录：其他语言的异步编程

### Go

Go 使用“goroutines”和“channels”，而不是我们之前介绍的所有语言固有的传统 async / await 语法。你可以将 channel 视为类似于 Python 的 future。在 Go 中，可以将 channel 作为参数发送给函数，然后使用 `go` 并发运行该函数。当需要确保函数完成时，可以使用 `<-` 语法，可以将其视为更常见的 `await` 语法。如果你的 goroutine（你的异步运行的函数）有返回值，则可以通过这种方式返回值。

```go
package main

import "fmt"

// "chan" makes the return value a string channel instead of a string
func returnHello(result chan string){
    // Gives our channel a value
    result <- "hello world"
}

func main() {
    // Creates a string channel
    result := make(chan string)

    // Starts execution of our goroutine
    go returnHello(result)

    // Awaits and prints our string
    fmt.Println(<- result)
}
```
