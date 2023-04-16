---
title: "UserModeScheduling in Windows"
date: 2023-04-16T22:13:41+03:00
---

User mode schuduling in Windows is such a hot topic that there is about 3 total post about it on the internet.
The stakes are high! First things first - I already described its mode of operation at [DConf 2018](https://www.youtube.com/watch?v=cnfjyWofYeY).

So here I'll just show some working code from implemenation of project [Photon](https://github.com/DmitryOlshansky/photon).

First here is how we spawn UserModeScheduled thread:

```d
public void spawn(void delegate() func)
{
    ubyte[128] buf = void;
    size_t size = buf.length;
    PROC_THREAD_ATTRIBUTE_LIST* attrList = cast(PROC_THREAD_ATTRIBUTE_LIST*)buf.ptr;
    wenforce(InitializeProcThreadAttributeList(attrList, 1, 0, &size), "failed to initialize proc thread");
    scope(exit) DeleteProcThreadAttributeList(attrList);
    
    UMS_CONTEXT* ctx;
    wenforce(CreateUmsThreadContext(&ctx), "failed to create UMS context");

    // power of 2 random choices:
    size_t a = uniform!"[)"(0, scheds.length);
    size_t b = uniform!"[)"(0, scheds.length);
    uint loadA = scheds[a].assigned; // take into account active queue.size?
    uint loadB = scheds[b].assigned; // ditto
    if (loadA < loadB) atomicOp!"+="(scheds[a].assigned, 1);
    else atomicOp!"+="(scheds[b].assigned, 1);
    UMS_CREATE_THREAD_ATTRIBUTES umsAttrs;
    umsAttrs.UmsCompletionList = loadA < loadB ? scheds[a].completionList : scheds[b].completionList;
    umsAttrs.UmsContext = ctx;
    umsAttrs.UmsVersion = UMS_VERSION;

    wenforce(UpdateProcThreadAttribute(attrList, 0, PROC_THREAD_ATTRIBUTE_UMS_THREAD, &umsAttrs, umsAttrs.sizeof, null, null), "failed to update proc thread");
    HANDLE handle = wenforce(CreateRemoteThreadEx(GetCurrentProcess(), null, 0, &worker, new Functor(func), 0, attrList, null), "failed to create thread");
    atomicOp!"+="(activeThreads, 1);
}

```

That's all there is to it. The only difficulty is which shceduler to put the freshly built ums-thread.


```d
package(photon) void schedulerEntry(size_t n)
{
    schedNum = n;
    UMS_SCHEDULER_STARTUP_INFO info;
    info.UmsVersion = UMS_VERSION;
    info.CompletionList = scheds[n].completionList;
    info.SchedulerProc = &umsScheduler;
    info.SchedulerParam = null;
    wenforce(SetThreadAffinityMask(GetCurrentThread(), 1<<n), "failed to set affinity");
    wenforce(EnterUmsSchedulingMode(&info), "failed to enter UMS mode\n");
}
```

And this is entry point for the scheduler thread. And the scheduler in its full glory:

```d
extern(Windows) VOID umsScheduler(UMS_SCHEDULER_REASON Reason, ULONG_PTR ActivationPayload, PVOID SchedulerParam)
{
    UMS_CONTEXT* ready;
    auto completionList = scheds[schedNum].completionList;
       logf("-----\nGot scheduled, reason: %d, schedNum: %x\n"w, Reason, schedNum);
    if(!DequeueUmsCompletionListItems(completionList, 0, &ready)){
        logf("Failed to dequeue ums workers!\n"w);
        return;
    }    
    for (;;)
    {
      scheds[schedNum].lock.lock();
      auto queue = &scheds[schedNum].queue; // struct, so take a ref
      while (ready != null)
      {
          logf("Dequeued UMS thread context: %x\n"w, ready);
          queue.push(ready);
          ready = GetNextUmsListItem(ready);
      }
      scheds[schedNum].lock.unlock();
      while(!queue.empty)
      {
        UMS_CONTEXT* ctx = queue.pop;
        logf("Fetched thread context from our queue: %x\n", ctx);
        BOOLEAN terminated;
        uint size;
        if(!QueryUmsThreadInformation(ctx, UMS_THREAD_INFO_CLASS.UmsThreadIsTerminated, &terminated, BOOLEAN.sizeof, &size))
        {
            logf("Query UMS failed: %d\n"w, GetLastError());
            return;
        }
        if (!terminated)
        {
            auto ret = ExecuteUmsThread(ctx);
            if (ret == ERROR_RETRY) // this UMS thread is locked, try it later
            {
                logf("Need retry!\n");
                queue.push(ctx);
            }
            else
            {
                logf("Failed to execute thread: %d\n"w, GetLastError());
                return;
            }
        }
        else
        {
            logf("Terminated: %x\n"w, ctx);
            //TODO: delete context or maybe cache them somewhere?
            DeleteUmsThreadContext(ctx);
            atomicOp!"-="(scheds[schedNum].assigned, 1);
            atomicOp!"-="(activeThreads, 1);
        }
      }
      if (activeThreads == 0)
      {
          logf("Shutting down\n"w);
          return;
      }
      if(!DequeueUmsCompletionListItems(completionList, INFINITE, &ready))
      {
           logf("Failed to dequeue UMS workers!\n"w);
           return;
      }
    }
}
```

The talk is cheap, show me the code.
