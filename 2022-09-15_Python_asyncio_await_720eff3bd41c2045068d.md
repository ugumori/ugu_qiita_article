<!--
title:   asyncioのawaitとcoroutine実行を理解する
tags:    Python,asyncio,await
id:      720eff3bd41c2045068d
private: false
-->
awaitはcoroutineの処理が完了するまで待つ。
複数のcoroutineを並列に走らせるにはasyncio.gatherを使うのだが、複数のcoroutineを並べてawaitするとどうなるのか。


まずはgather。
`python main.py
import asyncio
import time

async def func(sleep):
    print(f"{int(time.time())} func start sleep:{sleep}")
    await asyncio.sleep(sleep)
    print(f"{int(time.time())} func end sleep:{sleep}")

async def main():
    print(f"{int(time.time())} main start")
    sleep_task1 = asyncio.create_task(func(sleep=1))
    sleep_task2 = asyncio.create_task(func(sleep=2))
    await asyncio.gather(sleep_task1, sleep_task2)
    print(f"{int(time.time())} main end")

asyncio.run(main())
`

```output.log
1663209464 main start
1663209464 func start sleep:1
1663209464 func start sleep:2
1663209465 func end sleep:1
1663209466 func end sleep:2
1663209466 main end
```

全てのcoroutineが並列に実行され、完了するまで（つまり`sleep_task2`が完了するまで）待っている.

ではawaitを並べてみる。mainを書き換える.

```python
async def main():
    print(f"{int(time.time())} main start")
    sleep_task1 = asyncio.create_task(func(sleep=1))
    sleep_task2 = asyncio.create_task(func(sleep=2))
    await sleep_task1
    await sleep_task2
    print(f"{int(time.time())} main end")
```

```output.log
1663209617 main start
1663209617 func start sleep:1
1663209617 func start sleep:2
1663209618 func end sleep:1
1663209619 func end sleep:2
1663209619 main end
```
なんとなく、sleep_task1の完了を待ってからsleep_task2を開始するのかなと思ってたが、gatherを使った場合と変わらない。並列に開始し、sleep_task2が完了するまで待っているようだ。

ではこうするとどうなるのか

```python
async def main():
    print(f"{int(time.time())} main start")
    sleep_task1 = asyncio.create_task(func(sleep=1))
    await sleep_task1 # create_taskの間に挟む.
    sleep_task2 = asyncio.create_task(func(sleep=2))
    await sleep_task2
    print(f"{int(time.time())} main end")
```

```output.log
1663209991 main start
1663209991 func start sleep:1
1663209992 func end sleep:1
1663209992 func start sleep:2
1663209994 func end sleep:2
1663209994 main end
```
この場合はsleep_task1の完了を待ってからsleep_task2を開始し、完了を待っている.
awaitは、**awaitを実行した段階でasyncio.create_taskによりスケジュールされているcoroutineのみを実行開始し、完了を待つようだ。**

では2つのcoroutineを`asyncio.create_task`でスケジュールするが、awaitを1つしかしない場合どうなるのか.

```python
async def main():
    print(f"{int(time.time())} main start")
    sleep_task1 = asyncio.create_task(func(sleep=1))
    sleep_task2 = asyncio.create_task(func(sleep=2))
    await sleep_task1
    # await sleep_task2 # sleep_task2はawaitしない.
    print(f"{int(time.time())} main end")
```

```output.log
1663210406 main start
1663210406 func start sleep:1
1663210406 func start sleep:2
1663210407 func end sleep:1
1663210407 main end
```

こちらはなんとなくsleep_task2は実行されないのではと思ってしまうが、`sleep_task1`と`sleep_task2`は並列に開始されている。だが、awaitしていないsleep_task2は完了しないままプロセスが終了している.

`await`を実行すると処理が走るので、awaitは「指定されたcoroutineを実行し、完了まで待つ」という認識を持ってしまうがそうではなく、awaitはあくまで「指定されたcoroutineの完了を待つ」。
**実行されるcoroutineはawaitするまでにスケジュールしたcoroutine**である。
そしてawaitしていないcoroutineは実行されても当然完了を待たない。


## まとめ
- awaitを実行した時、それまでにcreate_taskでスケジュールされたcoroutineが実行される。
- awaitはあくまで待つsynctaxである。開始する処理を指定するものではない。