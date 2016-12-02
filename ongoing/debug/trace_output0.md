---
layout: post
title:  "ART虚拟机之trace"
date:   2016-6-22 22:19:53
catalog:    true
tags:
    - android
    - debug
    - art

---

> 分析Art虚拟机的trace原理，相关源码都位于/art/runtime目录：

    /art/runtime/
        - signal_catcher.cc
        - runtime.cc
        - intern_table.cc
        - thread_list.cc
        - java_vm_ext.cc
        - class_linker.cc
        - gc/heap.cc


一、概述

Android 6.0系统采用的art虚拟机，所有的Java进程都运行在art之上，当应用发生ANR(无响应)时，往往会向目标进程发送信号SIGNAL_QUIT, 传统的linux则是终止程序并输出core，而对于Android来说当收到SIGQUIT时，ART虚拟机会捕获该信号，并输出相应的traces信息。当然，我们也可以通过一条命令来获取指定进程的traces信息：

    kill -3 [pid]  //通过指定进程pid，输出结果位于/data/anr/traces.txt

上述命令的作用，便是等效于向目标进程pid发送了信号SIGQUIT，Java层面的进程都是跑在虚拟机之上的，
接下来从虚拟机角度说说目标进程收到该信号的处理过程。

二. ART

### 1. SignalCatcher
[-> SignalCatcher.cc]

    void* SignalCatcher::Run(void* arg) {
      SignalCatcher* signal_catcher = reinterpret_cast<SignalCatcher*>(arg);
      Runtime* runtime = Runtime::Current();
      
      Thread* self = Thread::Current();
      //当前进程状态处于非Runnable是
      DCHECK_NE(self->GetState(), kRunnable);
      {
        MutexLock mu(self, signal_catcher->lock_);
        signal_catcher->thread_ = self;
        signal_catcher->cond_.Broadcast(self);
      }

      //设置需要handle的信号
      SignalSet signals;
      signals.Add(SIGQUIT); //信号3
      signals.Add(SIGUSR1); //信号10

      while (true) {
        int signal_number = signal_catcher->WaitForSignal(self, signals);
        if (signal_catcher->ShouldHalt()) {
          runtime->DetachCurrentThread();
          return nullptr;
        }

        switch (signal_number) {
        case SIGQUIT:
          //收到信号3 【见小节2】
          signal_catcher->HandleSigQuit();
          break;
        case SIGUSR1:
          signal_catcher->HandleSigUsr1();
          break;
        default:
          LOG(ERROR) << "Unexpected signal %d" << signal_number;
          break;
        }
      }
    }


### 2.  SignalCatcher::HandleSigQuit
[-> signal_catcher.cc]

    void SignalCatcher::HandleSigQuit() {
      Runtime* runtime = Runtime::Current();
      std::ostringstream os;
      os << "\n" << "----- pid " << getpid() << " at " << GetIsoDate() << " -----\n";

      DumpCmdLine(os);

      std::string fingerprint = runtime->GetFingerprint();
      os << "Build fingerprint: '" << (fingerprint.empty() ? "unknown" : fingerprint) << "'\n";
      os << "ABI: '" << GetInstructionSetString(runtime->GetInstructionSet()) << "'\n";
      os << "Build type: " << (kIsDebugBuild ? "debug" : "optimized") << "\n";
      // [见小节3]
      runtime->DumpForSigQuit(os);

      os << "----- end " << getpid() << " -----\n";
      Output(os.str());
    }

### 3. Runtime::DumpForSigQuit
[-> runtime.cc]

    void Runtime::DumpForSigQuit(std::ostream& os) {
      GetClassLinker()->DumpForSigQuit(os); //[见小节3.1]
      GetInternTable()->DumpForSigQuit(os); //[见小节3.2]
      GetJavaVM()->DumpForSigQuit(os); //[见小节3.3]
      GetHeap()->DumpForSigQuit(os); //[见小节3.4]
      TrackedAllocators::Dump(os);
      os << "\n";

      thread_list_->DumpForSigQuit(os); //[见小节3.5]
      BaseMutex::DumpAll(os);
    }

#### 3.1 ClassLinker
[-> class_linker.cc]

    void ClassLinker::DumpForSigQuit(std::ostream& os) {
      Thread* self = Thread::Current();
      if (dex_cache_image_class_lookup_required_) {
        ScopedObjectAccess soa(self);
        MoveImageClassesToClassTable();
      }
      ReaderMutexLock mu(self, *Locks::classlinker_classes_lock_);
      os << "Zygote loaded classes=" << pre_zygote_class_table_.Size() << " post zygote classes="
         << class_table_.Size() << "\n";
    }

#### 3.2 InternTable
[-> intern_table.cc]

    void InternTable::DumpForSigQuit(std::ostream& os) const {
      os << "Intern table: " << StrongSize() << " strong; " << WeakSize() << " weak\n";
    }

#### 3.3 JavaVMExt
[-> java_vm_ext.cc]

    void JavaVMExt::DumpForSigQuit(std::ostream& os) {
      os << "JNI: CheckJNI is " << (check_jni_ ? "on" : "off");
      if (force_copy_) {
        os << " (with forcecopy)";
      }
      Thread* self = Thread::Current();
      {
        ReaderMutexLock mu(self, globals_lock_);
        os << "; globals=" << globals_.Capacity();
      }
      {
        MutexLock mu(self, weak_globals_lock_);
        if (weak_globals_.Capacity() > 0) {
          os << " (plus " << weak_globals_.Capacity() << " weak)";
        }
      }
      os << '\n';

      {
        MutexLock mu(self, *Locks::jni_libraries_lock_);
        os << "Libraries: " << Dumpable<Libraries>(*libraries_) << " (" << libraries_->size() << ")\n";
      }
    }

#### 3.4 Heap
[-> heap.cc]

    void Heap::DumpForSigQuit(std::ostream& os) {
      os << "Heap: " << GetPercentFree() << "% free, " << PrettySize(GetBytesAllocated()) << "/"
         << PrettySize(GetTotalMemory()) << "; " << GetObjectsAllocated() << " objects\n";
      DumpGcPerformanceInfo(os);  //输出大量gc相关的信息
    }

DumpGcPerformanceInfo()这个方法的参数非常多,先省略, 后续再单独用一篇文章来讲解.

#### 3.5 ThreadList
[-> thread_list.cc]

    void ThreadList::DumpForSigQuit(std::ostream& os) {
        {
            ScopedObjectAccess soa(Thread::Current());

            if (suspend_all_historam_.SampleSize() > 0) {
              Histogram<uint64_t>::CumulativeData data;
              suspend_all_historam_.CreateHistogram(&data);
              suspend_all_historam_.PrintConfidenceIntervals(os, 0.99, data);  // Dump time to suspend.
            }
        }
        Dump(os); // [见小节3.5.1]
        DumpUnattachedThreads(os); //[见小节3.5.2]
    }

##### 3.5.1 Dump
[-> thread_list.cc]

    void ThreadList::Dump(std::ostream& os) {
      {
        MutexLock mu(Thread::Current(), *Locks::thread_list_lock_);
        //输出当前进程的线程个数
        os << "DALVIK THREADS (" << list_.size() << "):\n";
      }
      DumpCheckpoint checkpoint(&os);
      size_t threads_running_checkpoint = RunCheckpoint(&checkpoint);
      if (threads_running_checkpoint != 0) {
        checkpoint.WaitForThreadsToRunThroughCheckpoint(threads_running_checkpoint);
      }
    }

`DALVIK THREADS (25)`代表的是当前虚拟机中的线程个数为25

##### 3.5.2 DumpUnattachedThreads
[-> thread_list.cc]

    void ThreadList::DumpUnattachedThreads(std::ostream& os) {
      DIR* d = opendir("/proc/self/task");
      if (!d) {
        return;
      }

      Thread* self = Thread::Current();
      dirent* e;
      while ((e = readdir(d)) != nullptr) {
        char* end;
        pid_t tid = strtol(e->d_name, &end, 10);
        if (!*end) {
          bool contains;
          {
            MutexLock mu(self, *Locks::thread_list_lock_);
            contains = Contains(tid);
          }
          if (!contains) {
            DumpUnattachedThread(os, tid); //[见小节3.6]
          }
        }
      }
      closedir(d);
    }

获取当前进程中所有的线程

### 3.6 DumpUnattachedThread
[-> thread_list.cc]

    static void DumpUnattachedThread(std::ostream& os, pid_t tid) NO_THREAD_SAFETY_ANALYSIS {
      Thread::DumpState(os, nullptr, tid); //[见小节3.6.1]
      DumpKernelStack(os, tid, "  kernel: ", false); //[见小节3.6.2]
      os << "\n";
    }

将进程中的每个线程都执行一次该方法

#### 3.6.1 Thread::DumpState
[-> thread.cc]

    void Thread::DumpState(std::ostream& os, const Thread* thread, pid_t tid) {
      std::string group_name;
      int priority;
      bool is_daemon = false;
      Thread* self = Thread::Current();

      if (thread != nullptr) {
        ScopedObjectAccessUnchecked soa(self);
        Thread* this_thread = const_cast<Thread*>(thread);
        Closure* flip_func = this_thread->GetFlipFunction();
        if (flip_func != nullptr) {
          flip_func->Run(this_thread);
        }
      }

      if (gAborting == 0 && self != nullptr && thread != nullptr && thread->tlsPtr_.opeer != nullptr) {
        ScopedObjectAccessUnchecked soa(self);
        priority = soa.DecodeField(WellKnownClasses::java_lang_Thread_priority)
            ->GetInt(thread->tlsPtr_.opeer);
        is_daemon = soa.DecodeField(WellKnownClasses::java_lang_Thread_daemon)
            ->GetBoolean(thread->tlsPtr_.opeer);

        mirror::Object* thread_group =
            soa.DecodeField(WellKnownClasses::java_lang_Thread_group)->GetObject(thread->tlsPtr_.opeer);

        if (thread_group != nullptr) {
          ArtField* group_name_field =
              soa.DecodeField(WellKnownClasses::java_lang_ThreadGroup_name);
          mirror::String* group_name_string =
              reinterpret_cast<mirror::String*>(group_name_field->GetObject(thread_group));
          group_name = (group_name_string != nullptr) ? group_name_string->ToModifiedUtf8() : "<null>";
        }
      } else {
        priority = GetNativePriority();
      }

      std::string scheduler_group_name(GetSchedulerGroupName(tid));
      if (scheduler_group_name.empty()) {
        scheduler_group_name = "default";
      }

      if (thread != nullptr) {
        os << '"' << *thread->tlsPtr_.name << '"';
        if (is_daemon) {
          os << " daemon";
        }
        os << " prio=" << priority
           << " tid=" << thread->GetThreadId()
           << " " << thread->GetState();  //获取线程状态
        if (thread->IsStillStarting()) {
          os << " (still starting up)";
        }
        os << "\n";
      } else {
        os << '"' << ::art::GetThreadName(tid) << '"'
           << " prio=" << priority
           << " (not attached)\n";
      }

      if (thread != nullptr) {
        MutexLock mu(self, *Locks::thread_suspend_count_lock_);
        os << "  | group=\"" << group_name << "\""
           << " sCount=" << thread->tls32_.suspend_count
           << " dsCount=" << thread->tls32_.debug_suspend_count
           << " obj=" << reinterpret_cast<void*>(thread->tlsPtr_.opeer)
           << " self=" << reinterpret_cast<const void*>(thread) << "\n";
      }

      os << "  | sysTid=" << tid
         << " nice=" << getpriority(PRIO_PROCESS, tid)
         << " cgrp=" << scheduler_group_name;
      if (thread != nullptr) {
        int policy;
        sched_param sp;
        CHECK_PTHREAD_CALL(pthread_getschedparam, (thread->tlsPtr_.pthread_self, &policy, &sp),
                           __FUNCTION__);
        os << " sched=" << policy << "/" << sp.sched_priority
           << " handle=" << reinterpret_cast<void*>(thread->tlsPtr_.pthread_self);
      }
      os << "\n";

      std::string scheduler_stats;
      if (ReadFileToString(StringPrintf("/proc/self/task/%d/schedstat", tid), &scheduler_stats)) {
        scheduler_stats.resize(scheduler_stats.size() - 1);  // Lose the trailing '\n'.
      } else {
        scheduler_stats = "0 0 0";
      }

      char native_thread_state = '?';
      int utime = 0;
      int stime = 0;
      int task_cpu = 0;
      GetTaskStats(tid, &native_thread_state, &utime, &stime, &task_cpu);

      os << "  | state=" << native_thread_state
         << " schedstat=( " << scheduler_stats << " )"
         << " utm=" << utime
         << " stm=" << stime
         << " core=" << task_cpu
         << " HZ=" << sysconf(_SC_CLK_TCK) << "\n";
      if (thread != nullptr) {
        os << "  | stack=" << reinterpret_cast<void*>(thread->tlsPtr_.stack_begin) << "-"
            << reinterpret_cast<void*>(thread->tlsPtr_.stack_end) << " stackSize="
            << PrettySize(thread->tlsPtr_.stack_size) << "\n";

        os << "  | held mutexes=";
        for (size_t i = 0; i < kLockLevelCount; ++i) {
          if (i != kMonitorLock) {
            BaseMutex* mutex = thread->GetHeldMutex(static_cast<LockLevel>(i));
            if (mutex != nullptr) {
              os << " \"" << mutex->GetName() << "\"";
              if (mutex->IsReaderWriterMutex()) {
                ReaderWriterMutex* rw_mutex = down_cast<ReaderWriterMutex*>(mutex);
                if (rw_mutex->GetExclusiveOwnerTid() == static_cast<uint64_t>(tid)) {
                  os << "(exclusive held)";
                } else {
                  os << "(shared held)";
                }
              }
            }
          }
        }
        os << "\n";
      }
    }

#### 3.6..2 DumpKernelStack
[-> art/runtime/utils.cc]

    void DumpKernelStack(std::ostream& os, pid_t tid, const char* prefix, bool include_count) {
      if (tid == GetTid()) {
        return;
      }

      //内核栈是通过读取节点/proc/self/task/[tid]/stack
      std::string kernel_stack_filename(StringPrintf("/proc/self/task/%d/stack", tid));
      std::string kernel_stack;
      if (!ReadFileToString(kernel_stack_filename, &kernel_stack)) {
        os << prefix << "(couldn't read " << kernel_stack_filename << ")\n";
        return;
      }

      std::vector<std::string> kernel_stack_frames;
      Split(kernel_stack, '\n', &kernel_stack_frames);

      kernel_stack_frames.pop_back();
      for (size_t i = 0; i < kernel_stack_frames.size(); ++i) {

        const char* text = kernel_stack_frames[i].c_str();
        const char* close_bracket = strchr(text, ']');
        if (close_bracket != nullptr) {
          text = close_bracket + 2;
        }
        os << prefix;
        if (include_count) {
          os << StringPrintf("#%02zd ", i);
        }
        os << text << "\n";
      }
    }

内核栈是通过读取节点/proc/self/task/[tid]/stack

## 实例

    //[见小节2]
    ----- pid 874 at 2016-11-11 22:22:22 -----
    Cmd line: system_server
    ABI: arm
    Build type: optimized
    //[见小节3.1]
    Loaded classes: 6805 allocated classes
    //[见小节3.2]
    Intern table: 6422 strong; 10703 weak
    //[见小节3.3]
    JNI: CheckJNI is off; globals=2418 (plus 115 weak)
    Libraries: /system/lib/libandroid.so /system/lib/libandroid_servers.so /system/lib/libaudioeffect_jni.so /system/lib/libcompiler_rt.so /system/lib/libjavacrypto.so /system/lib/libjnigraphics.so /system/lib/libmedia_jni.so /system/lib/libmiui_security.so /system/lib/libmiuiclassproxy.so /system/lib/libmiuinative.so /system/lib/librs_jni.so /system/lib/libsechook.so /system/lib/libshell_jni.so /system/lib/libsoundpool.so /system/lib/libwebviewchromium_loader.so /system/lib/libwifi-service.so /vendor/lib/libalarmservice_jni.so /vendor/lib/liblocationservice.so libjavacore.so (19)
    //[见小节3.4]
    Heap: 27% free, 29MB/40MB; 307772 objects
    ... //省略GC相关信息

    //[见小节3.5]
    DALVIK THREADS (99):
    //[见小节3.6]
    "main" prio=5 tid=1 Native
      | group="main" sCount=1 dsCount=0 obj=0x7474e0d0 self=0xb5007800
      | sysTid=874 nice=-2 cgrp=apps sched=0/0 handle=0xb6fc7ec8
      | state=S schedstat=( 0 0 0 ) utm=900 stm=435 core=2 HZ=100
      | stack=0xbe78e000-0xbe790000 stackSize=8MB
      | held mutexes=
      native: #00 pc 0003684c  /system/lib/libc.so (__epoll_pwait+20)
      native: #01 pc 00011b1f  /system/lib/libc.so (epoll_pwait+26)
      native: #02 pc 00011b2d  /system/lib/libc.so (epoll_wait+6)
      native: #03 pc 00010fab  /system/lib/libutils.so (_ZN7android6Looper9pollInnerEi+98)
      native: #04 pc 000111d5  /system/lib/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+92)
      native: #05 pc 0007c625  /system/lib/libandroid_runtime.so (_ZN7android18NativeMessageQueue8pollOnceEP7_JNIEnvi+22)
      native: #06 pc 000b0e47  /data/dalvik-cache/arm/system@framework@boot.oat (Java_android_os_MessageQueue_nativePollOnce__JI+102)
      at android.os.MessageQueue.nativePollOnce(Native method)
      at android.os.MessageQueue.next(MessageQueue.java:143)
      at android.os.Looper.loop(Looper.java:122)
      at com.android.server.SystemServer.run(SystemServer.java:282)
      at com.android.server.SystemServer.main(SystemServer.java:183)
      at java.lang.reflect.Method.invoke!(Native method)
      at java.lang.reflect.Method.invoke(Method.java:372)
      at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:906)
      at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:701)
      
   ... //此处省略N个线程.
    

## 四. 线程状态

### 4.1 虚拟机角度

[-> art/runtime/thread.h]

状态表:

    kStarting,                        // NEW            TS_WAIT      native thread started, not yet ready to run managed code
    kTerminated,                      // TERMINATED     TS_ZOMBIE    Thread.run has returned, but Thread* still around
    kRunnable,                        // RUNNABLE       TS_RUNNING   runnable
    kNative,                          // RUNNABLE       TS_RUNNING   running in a JNI native method
    kSuspended,                       // RUNNABLE       TS_RUNNING   suspended by GC or debugger

    kBlocked,                         // BLOCKED        TS_MONITOR   blocked on a monitor
    kSleeping,                        // TIMED_WAITING  TS_SLEEPING  in Thread.sleep()
    kTimedWaiting,                    // TIMED_WAITING  TS_WAIT      in Object.wait() with a timeout
    kWaiting,                         // WAITING        TS_WAIT      in Object.wait()

    kWaitingForGcToComplete,          // WAITING        TS_WAIT      blocked waiting for GC
    kWaitingForCheckPointsToRun,      // WAITING        TS_WAIT      GC waiting for checkpoints to run
    kWaitingPerformingGc,             // WAITING        TS_WAIT      performing GC
    kWaitingForDebuggerSend,          // WAITING        TS_WAIT      blocked waiting for events to be sent
    kWaitingForDebuggerToAttach,      // WAITING        TS_WAIT      blocked waiting for debugger to attach
    kWaitingInMainDebuggerLoop,       // WAITING        TS_WAIT      blocking/reading/processing debugger events
    kWaitingForDebuggerSuspension,    // WAITING        TS_WAIT      waiting for debugger suspend all
    kWaitingForJniOnLoad,             // WAITING        TS_WAIT      waiting for execution of dlopen and JNI on load code
    kWaitingForSignalCatcherOutput,   // WAITING        TS_WAIT      waiting for signal catcher IO to complete
    kWaitingInMainSignalCatcherLoop,  // WAITING        TS_WAIT      blocking/reading/processing signals
    kWaitingForDeoptimization,        // WAITING        TS_WAIT      waiting for deoptimization suspend all
    kWaitingForMethodTracingStart,    // WAITING        TS_WAIT      waiting for method tracing to start
    kWaitingForVisitObjects,          // WAITING        TS_WAIT      waiting for visiting objects
    kWaitingForGetObjectsAllocated,   // WAITING        TS_WAIT      waiting for getting the number of allocated objects


### 4.2 内核角度

[-> kernel/include/linux/sched.h]

    #define TASK_RUNNING		0         //R
    #define TASK_INTERRUPTIBLE	1         //S
    #define TASK_UNINTERRUPTIBLE	2     //D
    #define __TASK_STOPPED		4         //T
    #define __TASK_TRACED		8         //T
    #define EXIT_ZOMBIE		16            //Z
    #define EXIT_DEAD		32            //X

进程状态: R/S/D/T/Z/X

### 4.3 映射关系

R: Runnable, Native, Suspended
S: TimedWaiting, Waiting, Sleeping, Native,  Blocked