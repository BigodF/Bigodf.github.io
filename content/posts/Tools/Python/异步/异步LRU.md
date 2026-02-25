---
title: python异步LRU
date: '2025-11-28T15:24:58'
lastmod: '2025-11-28T17:07:35'
author:
- Bigodf
tags:
- Tools
- Python
- 异步
- LRU
description: python异步LRU
summary: ''
weight: null
slug: ''
draft: false
comments: true
showToc: true
TocOpen: true
autonumbering: true
hidemeta: false
disableShare: true
searchHidden: false
showbreadcrumbs: true
mermaid: true
cover:
  image: ''
  caption: ''
  alt: ''
  relative: false


---

本文介绍一个第三方库async-lru，安装方式如下
```bash
pip install async-lru
```

功能：
1. 作为装饰器，给函数计算结果加上 LRU 缓存。
2. 支持参数
	1. 最大缓存个数。
	2. key 的过期时间。


## 源码机制
1. async-lru（以下记为 **alru**）和 python 自带的 lru_cache （以下记为 **lru**）一样，都使用双向链表+map 的方式存储，不同的是，lru 实际采用双向链表+map，alru 使用 python 自带的 OrderedDict，底层原理是一样的。
2. 对于多个 key 同时请求，alru 有一个锁机制，用于保证相同的 key 只会执行一次。
	1. 实现原理：每一个 key 缓存的是 asyncio.future，返回值是 await cache[key].fnt。
3. 使用 asyncio.shield 用于保护协程不会被 cancel ，目的是不浪费计算。

源码：
```python fold
@final
@dataclasses.dataclass
class _CacheItem(Generic[_R]):
    fut: "asyncio.Future[_R]"
    later_call: Optional[asyncio.Handle]

    def cancel(self) -> None:
        if self.later_call is not None:
            self.later_call.cancel()
            self.later_call = None
            
def _task_done_callback(
        self, fut: "asyncio.Future[_R]", key: Hashable, task: "asyncio.Task[_R]"
    ) -> None:
        self.__tasks.discard(task)

        if task.cancelled():
            fut.cancel()
            self.__cache.pop(key, None)
            return

        exc = task.exception()
        if exc is not None:
            fut.set_exception(exc)
            self.__cache.pop(key, None)
            return

        cache_item = self.__cache.get(key)
        if self.__ttl is not None and cache_item is not None:
            loop = asyncio.get_running_loop()
            cache_item.later_call = loop.call_later( # 添加一个过期任务，自动pop key。
                self.__ttl, self.__cache.pop, key, None
            )

        fut.set_result(task.result())
    
async def __call__(self, /, *fn_args: Any, **fn_kwargs: Any) -> _R:
        if self.__closed:
            raise RuntimeError(f"alru_cache is closed for {self}")

        loop = asyncio.get_running_loop()

        key = _make_key(fn_args, fn_kwargs, self.__typed)

        cache_item = self.__cache.get(key)

        if cache_item is not None: # 命中缓存（包含锁机制，每个key只会处理一次）
            self._cache_hit(key)
            if not cache_item.fut.done(): # 任务未完成，等待任务完成并返回任务结果
                return await asyncio.shield(cache_item.fut) # await future等同于返回future.result()，asyncio.shield用于保护fut不会被cancel

            return cache_item.fut.result() # 任务完成，返回任务的结果

        fut = loop.create_future()
        coro = self.__wrapped__(*fn_args, **fn_kwargs)
        task: asyncio.Task[_R] = loop.create_task(coro)
        self.__tasks.add(task)
        task.add_done_callback(partial(self._task_done_callback, fut, key))

        self.__cache[key] = _CacheItem(fut, None)

        if self.__maxsize is not None and len(self.__cache) > self.__maxsize:
            dropped_key, cache_item = self.__cache.popitem(last=False)
            cache_item.cancel() # 取消过期任务

        self._cache_miss(key)
        return await asyncio.shield(fut) # 直接返回结果
```

使用示例
```python fold
sem = asyncio.Semaphore(100) # 控制并发量
@alru_cache(maxsize=200000, ttl=2*60) # 最大缓存200000个，过期时间120s
async def sensitive_cache(sentence: str):
    async with sem:
        await asyncio.sleep(0.1)
        return sentence[0]
```

