# How to pause/resume an asyncio app

Folks.
Today I want to show you how to safely pause the execution of an asyncio loop, and resume it where it left off.

## Motivation

If you're reading this page, chances are you already have a problem to solve.
However, it's important to make sure we are on the same page here.
So, let's take a look at the vague concept I had in my mind before writing this post:

```python
async def f():
    await part_one()
    await pause()
    await part_two()
    await pause()
    await part_three()

def main():
    event_loop = asyncio.new_event_loop()
    task_1 = event_loop.create_task(f())
    while True:
        event_loop.run_until_pause()
        do_something()

if __name__ == "__main__":
    main()
```

A traditional approach here is to rewrite `f` as a class member function and store the state in the class in a way
that we can call `f` repeatedly and each time it would run a different code based on the state.
With an approach like that, there won't be a need for `pause()` because `f` itself would return whenever a step is done.
However, this is not in the spirit of asyncio.
We like `f` as it is: a linear function that tells a nice story about what it does.

## Implementing the `pause()`

Using two `Future` objects, one can implement the `pause()` functionality.
Try running the following code:

```python
import asyncio


class PausableEventLoop(asyncio.SelectorEventLoop):
    def __init__(self):
        super().__init__()
        self.pause_future = None
        self.resume_future = None

    def run_until_pause(self):
        self.pause_future = asyncio.Future(loop=self)
        self.resume_future = asyncio.Future(loop=self)
        self.run_until_complete(self.pause_future)
        self.resume_future.set_result(None)
        self.pause_future = None

    def check_all_tasks_completed(self, _):
        # Makes sure the loop finishes after running the piece after the last pause
        if all(
            t
            for t in asyncio.all_tasks(loop=self)
            if not t.done()
        ):
            self.pause_future.set_result(None)  # signal completion

    def create_task(self, *args, **kwargs):
        task = super().create_task(*args, **kwargs)
        task.add_done_callback(self.check_all_tasks_completed)
        return task


async def pause():  # handy function
    loop = asyncio.get_event_loop()
    loop.pause_future.set_result(None)
    await loop.resume_future


async def f():
    print("Part 1")
    await asyncio.sleep(1)
    await pause()
    print("Part 2")
    await asyncio.sleep(2)
    await pause()
    print("Part 3")
    await asyncio.sleep(3)


def main():
    event_loop = PausableEventLoop()
    task_1 = event_loop.create_task(f())
    while not task_1.done():
        event_loop.run_until_pause()
        print("Paused")


if __name__ == "__main__":
    main()
```

The implementation can be more sophisticated but I want all parts to be visible for now.

The problem with this approach is that if you have more than one task running at the same time,
the pause will happen as soon as one of them calls `pause()`.
This is not a problem per se, as asyncio is single threaded which means that when a `pause()` happens,
all other coroutines are also `await`ing somewhere with proper callbacks already set up in the event loop.

So pausing abruptly shouldn't break anything on paper.
But still, if the pause is long it might increase the chance of a buffer overflow, etc.
It may also make reasoning about the code difficult because each coroutine can be at some random state when a pause happens.
Also based on what we want to do when we pause, there may be a logical requirement to make the pause depend on more than one coroutine calling a function.

An interesting approach worth exploring is an implementation that uses a notion of *safepoints* to make the pause more predictable.
In this design, the loop will pause only if *all* coroutines call a `pause()`, or what we may now call `safepoint()`.
Try running the following code:

```python
import asyncio

class SafepointBlock:
    def __init__(self, event_loop):
        self.awaited_count = 0
        self.event_loop = event_loop
        self.join_future = asyncio.Future(loop=event_loop)  # hits when all tasks are here -> event loop cares about this
        self.resume_future = asyncio.Future(loop=event_loop)  # hits when we can to continue -> running tasks care about this

    async def stop_if_needed(self):
        self.awaited_count += 1
        self.signal_safe_state_if_needed()
        await self.resume_future
    
    def signal_safe_state_if_needed(self):
        active_tasks = [
            t
            for t in asyncio.all_tasks(loop=self.event_loop)
            if not t.done()
        ]
        if len(active_tasks) == self.awaited_count:
            self.join_future.set_result(None)

class PausableEventLoop(asyncio.SelectorEventLoop):
    def __init__(self):
        super().__init__()
        self.safepoint_block = None

    def run_until_safe_state(self):
        self.safepoint_block = SafepointBlock(self)
        self.run_until_complete(self.safepoint_block.join_future)
        self.safepoint_block.resume_future.set_result(None)
        self.safepoint_block = None
    
    def check_safepoint_at_task_completion(self, _):
        if self.safepoint_block is not None:
            self.safepoint_block.signal_safe_state_if_needed()

    def create_task(self, *args, **kwargs):
        task = super().create_task(*args, **kwargs)
        task.add_done_callback(self.check_safepoint_at_task_completion)
        return task
    
    def pause_soon(self):
        # not used in this demo but it's a useful utility.
        self.safepoint_block = SafepointBlock(self)

async def safepoint():  # handy function for devs
    safepoint_block = asyncio.get_event_loop().safepoint_block
    if safepoint_block is None:
        return  # pause not requested
    await safepoint_block.stop_if_needed()

async def function_1():
    print("Task 1 started.")
    await asyncio.sleep(1)
    print("Task 1 at safe state.")
    await safepoint()
    print("Task 1 resuming.")
    await asyncio.sleep(1)
    print("Task 1 finished.")

async def function_2():
    print("Task 2 started.")
    await asyncio.sleep(2)
    print("Task 2 at safe state.")
    await safepoint()
    print("Task 2 resuming.")
    await asyncio.sleep(2)
    print("Task 2 finished.")

def main():
    print("App started.")
    event_loop = PausableEventLoop()
    task_1 = event_loop.create_task(function_1())
    task_2 = event_loop.create_task(function_2())
    tasks = [task_1, task_2]
    print("Starting event loop")
    while not all([t.done() for t in tasks]):
        event_loop.run_until_safe_state()
        print("Event loop stopped. Resuming now")

if __name__ == "__main__":
    main()

```

Look at that output!
Now we only stop the loop when all the tasks are in a safe state.
This is much nicer than the former approach. There are no random frozen callbacks and the pause is more predictable.
Of course, this means that there will be no pause until all the tasks either finish or arrive at a safe-point.
It may sound too strict, but in reality, the only way a function never ends or hits a certain line is by running into an infinite loop.
Running into an infinite loop is an issue with any program, so the requirement for a safe-point design is nothing more than the bare minimum requirements of any working code.
