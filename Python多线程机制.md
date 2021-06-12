# Python多线程机制

python中的多线程是基于操作系统的原生线程实现的，同时，python总引入了一个全局解释器锁（Global Interepter Lock, GIL）来互斥不同线程对python虚拟机的使用。在python中，GIL是一个非常大粒度的锁，同一时间的多个python线程中，只有一个持有GIL的线程才能运行。

对于线程调度机制而言，有两个问题是需要解决的：

+ 调度时机：何时挂起当前线程，选择处于等待状态的下一个线程
+ 如何选择下一个线程：在众多的处于等待状态的候选者中，如何选择激活哪一个线程？

对于这两个问题，python分别在不同的层面给出了解决方案。

第一个问题是由python自身决定的。操作系统下当一个进程执行了一段时间后，发生时钟中断，操作系统响应时钟中断，并在这时开始进程调度。同样，python通过软件对时钟中断进行了模拟，来激活线程的调度。python中，默认的解释器切换时间间隔为5ms，可以通过sys.getswitchinterval来查看这个值

~~~python
>>> import sys
>>> sys.getswitchinterval()
0.005
~~~

第二个问题是由操作系统决定的。在python中，当前线程被挂起之后选择哪一个候选线程是由操作系统来决定的，python在这个问题上完全没有进行干预。这意味着python中的线程实际上就是操作系统的原生线程。在不同的操作系统上，线程有不同的实现，然而最终，python提供了一套统一的额抽象机制，给python使用者提供了非常简单而方便的接口——python中的两个module：_thread以及在其之上的threading。

## _thread模块

_thread模块是python提供的基础多线程模块，这是一个builtin模块。这个模块在python2中被称为thread模块。python中提供了一个更高级的模块threading，这是一个标准库中的模块，为用户提供了方便的多线程接口。

~~~C
static PyMethodDef thread_methods[] = {
    {"start_new_thread",        (PyCFunction)thread_PyThread_start_new_thread,
     METH_VARARGS, start_new_doc},
    {"start_new",               (PyCFunction)thread_PyThread_start_new_thread,
     METH_VARARGS, start_new_doc},
    {"allocate_lock",           thread_PyThread_allocate_lock,
     METH_NOARGS, allocate_doc},
    {"allocate",                thread_PyThread_allocate_lock,
     METH_NOARGS, allocate_doc},
    {"exit_thread",             thread_PyThread_exit_thread,
     METH_NOARGS, exit_doc},
    {"exit",                    thread_PyThread_exit_thread,
     METH_NOARGS, exit_doc},
    {"interrupt_main",          thread_PyThread_interrupt_main,
     METH_NOARGS, interrupt_doc},
    {"get_ident",               thread_get_ident,
     METH_NOARGS, get_ident_doc},
    {"_count",                  thread__count,
     METH_NOARGS, _count_doc},
    {"stack_size",              (PyCFunction)thread_stack_size,
     METH_VARARGS, stack_size_doc},
    {"_set_sentinel",           thread__set_sentinel,
     METH_NOARGS, _set_sentinel_doc},
    {NULL,                      NULL}           /* sentinel */
};
~~~

可以看到，\_thread中为用户提供的多线程机制的接口非常少，当然，这也使得python的多线程编程非常方便。

### python线程的创建

要研究的python的线程创建机制，首先要从线程的创建接口开始。

~~~C
static PyObject *
thread_PyThread_start_new_thread(PyObject *self, PyObject *fargs)
{
    PyObject *func, *args, *keyw = NULL;
    struct bootstate *boot;
    unsigned long ident;

    if (!PyArg_UnpackTuple(fargs, "start_new_thread", 2, 3,
                           &func, &args, &keyw))//拆传给_thread.start_new_thread(function, args[, kwargs])的三个参数分别保存在func、args以及keyw中
        return NULL;
    if (!PyCallable_Check(func)) {
        PyErr_SetString(PyExc_TypeError,
                        "first arg must be callable");
        return NULL;
    }
    if (!PyTuple_Check(args)) {
        PyErr_SetString(PyExc_TypeError,
                        "2nd arg must be a tuple");
        return NULL;
    }
    if (keyw != NULL && !PyDict_Check(keyw)) {
        PyErr_SetString(PyExc_TypeError,
                        "optional 3rd arg must be a dictionary");
        return NULL;
    }
    boot = PyMem_NEW(struct bootstate, 1);//创建bootstate结构
    if (boot == NULL)
        return PyErr_NoMemory();
    boot->interp = _PyInterpreterState_Get();//保存PyInterpreterState由所有线程共享
    boot->func = func;
    boot->args = args;
    boot->keyw = keyw;
    boot->tstate = _PyThreadState_Prealloc(boot->interp);
    if (boot->tstate == NULL) {
        PyMem_DEL(boot);
        return PyErr_NoMemory();
    }
    Py_INCREF(func);
    Py_INCREF(args);
    Py_XINCREF(keyw);
    PyEval_InitThreads(); /* Start the interpreter's thread-awareness *///初始化python的多线程环境
    ident = PyThread_start_new_thread(t_bootstrap, (void*) boot);//创建操作系统的原生线程
    if (ident == PYTHREAD_INVALID_THREAD_ID) {
        PyErr_SetString(ThreadError, "can't start new thread");
        Py_DECREF(func);
        Py_DECREF(args);
        Py_XDECREF(keyw);
        PyThreadState_Clear(boot->tstate);
        PyMem_DEL(boot);
        return NULL;
    }
    return PyLong_FromUnsignedLong(ident);
}
~~~

在thread\_PyThread\_start\_new\_thread中，虚拟机主要通过三个动作创建一个线程：

+ 准备工作
  - 参数拆解
  - bootstate结构体的创建以及初始化
+ 初始化python的多线程环境
+ 创建操作系统级别的线程

在python启动时是不支持多线程模式的，这是因为大多数python程序都不需要多线程的支持。所以，只有当用户明确提出多线程需求时，虚拟机才会创建相关包括GIL在内的多线程支持的数据结构。

### 多线程环境的初始化

python多线程环境的建立主要就是创建GIL。

~~~C
void
PyEval_InitThreads(void)
{
    if (gil_created())//如果GIL已经被创建则直接返回，避免重复创建
        return;
    create_gil();//创建GIL
    take_gil(_PyThreadState_GET());//持有GIL
    _PyRuntime.ceval.pending.main_thread = PyThread_get_thread_ident();//将当前线程设置为主线程
    if (!_PyRuntime.ceval.pending.lock)
        _PyRuntime.ceval.pending.lock = PyThread_allocate_lock();
}
~~~

首先，为了避免GIL被重复创建，虚拟机通过gil_created进行判断，如果GIL已经存则说明多线程环境已经准备就绪直接return。

如果GIL不存在则继续多线程环境的初始化流程。首先来看一下create_gil的实现。

~~~C
static void create_gil(void)
{
    MUTEX_INIT(_PyRuntime.ceval.gil.mutex);//创建互斥锁
#ifdef FORCE_SWITCHING
    MUTEX_INIT(_PyRuntime.ceval.gil.switch_mutex);
#endif
    COND_INIT(_PyRuntime.ceval.gil.cond);//创建条件变量
#ifdef FORCE_SWITCHING
    COND_INIT(_PyRuntime.ceval.gil.switch_cond);
#endif
    _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil.last_holder, 0);
    _Py_ANNOTATE_RWLOCK_CREATE(&_PyRuntime.ceval.gil.locked);
    _Py_atomic_store_explicit(&_PyRuntime.ceval.gil.locked, 0,
                              _Py_memory_order_release);
}

#define MUTEX_INIT(mut) \
    if (PyMUTEX_INIT(&(mut))) { \
        Py_FatalError("PyMUTEX_INIT(" #mut ") failed"); };

#define COND_INIT(cond) \
    if (PyCOND_INIT(&(cond))) { \
        Py_FatalError("PyCOND_INIT(" #cond ") failed"); };
~~~

我们知道，python作为一种跨平台的语言，需要针对不同平台进行特俗处理，这从MUTEX_INIT与COND_INIT这两个宏的定义上就可以看出来。我们来看condvar.h中对他们的定义

~~~C
#ifdef _POSIX_THREADS
/*
 * POSIX support
 */
/**/
#define PyMUTEX_INIT(mut)       pthread_mutex_init((mut), NULL)
/**/
#define PyCOND_INIT(cond)       pthread_cond_init((cond), NULL)
/**/
#elif defined(NT_THREADS)
/**/
Py_LOCAL_INLINE(int)
PyMUTEX_INIT(PyMUTEX_T *cs)
{
    InitializeCriticalSection(cs);
    return 0;
}
/*
 * Windows (XP, 2003 server and later, as well as (hopefully) CE) support
 *
 * Emulated condition variables ones that work with XP and later, plus
 * example native support on VISTA and onwards.
 */
/**/
~~~

针对不同的平台，python调用了相应平台下对应的锁创建相关的API，在这里，我们以POSIX API为例进行后续考察。

GIL创建完成之后，我们再来看一下take_gil的实现。

~~~C
static void take_gil(PyThreadState *tstate)
{
    int err;
    if (tstate == NULL)
        Py_FatalError("take_gil: NULL tstate");

    err = errno;
    MUTEX_LOCK(_PyRuntime.ceval.gil.mutex);//加锁

    if (!_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked))//如果GIL未被持有，那么直接跳转到_ready并获取锁
        goto _ready;
	//GIL已被持有
    while (_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked)) {
        int timed_out = 0;
        unsigned long saved_switchnum;

        saved_switchnum = _PyRuntime.ceval.gil.switch_number;//记录切换次数
        COND_TIMED_WAIT(_PyRuntime.ceval.gil.cond, _PyRuntime.ceval.gil.mutex,
                        INTERVAL, timed_out);//INTERVAL线程切换的间隔（毫秒，默认值为5）
        /* If we timed out and no switch occurred in the meantime, it is time
           to ask the GIL-holding thread to drop it. */
        if (timed_out &&
            _Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked) &&
            _PyRuntime.ceval.gil.switch_number == saved_switchnum) {
            SET_GIL_DROP_REQUEST();//发送请求释放GIL的信号——将gil_drop_request和eval_breaker这两个变量置为1
        }
    }
_ready:
#ifdef FORCE_SWITCHING
    /* This mutex must be taken before modifying
       _PyRuntime.ceval.gil.last_holder (see drop_gil()). */
    MUTEX_LOCK(_PyRuntime.ceval.gil.switch_mutex);
#endif
    /* We now hold the GIL *///成功获取GIL
    _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil.locked, 1);//locked置为1
    _Py_ANNOTATE_RWLOCK_ACQUIRED(&_PyRuntime.ceval.gil.locked, /*is_write=*/1);

    if (tstate != (PyThreadState*)_Py_atomic_load_relaxed(
                    &_PyRuntime.ceval.gil.last_holder))//是否确实发生了一次切换
    {
        _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil.last_holder,
                                 (uintptr_t)tstate);//记录最近一次持有GIL的线程
        ++_PyRuntime.ceval.gil.switch_number;//增加切换计数
    }

#ifdef FORCE_SWITCHING
    COND_SIGNAL(_PyRuntime.ceval.gil.switch_cond);
    MUTEX_UNLOCK(_PyRuntime.ceval.gil.switch_mutex);
#endif
    if (_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil_drop_request)) {
        RESET_GIL_DROP_REQUEST();//将gil_drop_request和eval_breaker置为0
    }
    if (tstate->async_exc != NULL) {
        _PyEval_SignalAsyncExc();
    }

    MUTEX_UNLOCK(_PyRuntime.ceval.gil.mutex);//解锁
    errno = err;
}

#define SET_GIL_DROP_REQUEST() \
    do { \
        _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil_drop_request, 1); \
        _Py_atomic_store_relaxed(&_PyRuntime.ceval.eval_breaker, 1); \
    } while (0)

#define RESET_GIL_DROP_REQUEST() \
    do { \
        _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil_drop_request, 0); \
        COMPUTE_EVAL_BREAKER(); \
    } while (0)
~~~

从take_gil的实现可以看出，GIL的申请逻辑都被\_PyRuntime.ceval.gil.mutex锁住。如果GIL已经被持有，则进入等待状态，通过COND_TIMED_WAIT实现超时等待，该宏在POSIX平台上实际上时调用的pthread_cond_timedwait；如果超时，则通过SET_GIL_DROP_REQUEST () 方法将 gil_drop_request、eval_breaker 这两个变量置 1，通知正在持有GIL的线程释放GIL。

python虚拟机通过drop_gil主动释放GIL。

~~~C
static void drop_gil(PyThreadState *tstate)
{
    if (!_Py_atomic_load_relaxed(&_PyRuntime.ceval.gil.locked))
        Py_FatalError("drop_gil: GIL is not locked");
    /* tstate is allowed to be NULL (early interpreter init) */
    if (tstate != NULL) {
        /* Sub-interpreter support: threads might have been switched
           under our feet using PyThreadState_Swap(). Fix the GIL last
           holder variable so that our heuristics work. */
        _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil.last_holder,
                                 (uintptr_t)tstate);
    }

    MUTEX_LOCK(_PyRuntime.ceval.gil.mutex);
    _Py_ANNOTATE_RWLOCK_RELEASED(&_PyRuntime.ceval.gil.locked, /*is_write=*/1);
    _Py_atomic_store_relaxed(&_PyRuntime.ceval.gil.locked, 0);//将locked置为0
    COND_SIGNAL(_PyRuntime.ceval.gil.cond);//通知正在等待中的其他线程
    MUTEX_UNLOCK(_PyRuntime.ceval.gil.mutex);

#ifdef FORCE_SWITCHING
    /*...*/
#endif
}
~~~

释放GIL的逻辑比较简单，此处不再赘述。

### 子线程的诞生

多线程环境初始化完毕后，虚拟机会开始创建底层平台的原生thread，这些工作在PyThread_start_new_thread中完成。

~~~C
unsigned long//根据thread_PyThread_start_new_thread中的调用关系可知，这里传入的第一个参数为t_bootstrap，第二个参数为boot
PyThread_start_new_thread(void (*func)(void *), void *arg)
{
    pthread_t th;//线程id
    int status;
#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
    pthread_attr_t attrs;
#endif
#if defined(THREAD_STACK_SIZE)
    size_t      tss;
#endif

    dprintf(("PyThread_start_new_thread called\n"));
    if (!initialized)
        PyThread_init_thread();

/*...*/

    pythread_callback *callback = PyMem_RawMalloc(sizeof(pythread_callback));

    if (callback == NULL) {
      return PYTHREAD_INVALID_THREAD_ID;
    }
	//参数打包
    callback->func = func;
    callback->arg = arg;
	//创建原生线程
    status = pthread_create(&th,
#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
                             &attrs,
#else
                             (pthread_attr_t*)NULL,
#endif
                             pythread_wrapper, callback);

#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
    pthread_attr_destroy(&attrs);
#endif

    if (status != 0) {
        PyMem_RawFree(callback);
        return PYTHREAD_INVALID_THREAD_ID;
    }

    pthread_detach(th);//将线程设置为detached状态（线程退出时将堆栈、推出状态等资源立即将资源回收而不是将其保存在内存中）

#if SIZEOF_PTHREAD_T <= SIZEOF_LONG
    return (unsigned long) th;
#else
    return (unsigned long) *(unsigned long *) &th;
#endif
}

//该结构体包含了一个线程的所有必须信息
struct bootstate {
    PyInterpreterState *interp;//进程状态
    PyObject *func;//线程函数体
    PyObject *args;//位置参数
    PyObject *keyw;//关键字参数
    PyThreadState *tstate;//线程状态
};

//将t_bootstrap和参数打包
typedef struct {
    void (*func) (void *);
    void *arg;
} pythread_callback;

//在该函数中执行t_bootstrap(boot)
pythread_wrapper(void *arg)
{
    /* copy func and func_arg and free the temporary structure */
    pythread_callback *callback = arg;
    void (*func)(void *) = callback->func;//t_bootstrap函数
    void *func_arg = callback->arg;//thread_PyThread_start_new_thread中的boot
    PyMem_RawFree(arg);

    func(func_arg);//[A]线程真正的执行现场
    return NULL;
}
~~~

### 子线程的执行与退出

线程创建完毕之后，在上述代码[A]处调用t_bootstrap开始了线程执行。

~~~C
static void
t_bootstrap(void *boot_raw)
{
    struct bootstate *boot = (struct bootstate *) boot_raw;
    PyThreadState *tstate;
    PyObject *res;

    tstate = boot->tstate;
    tstate->thread_id = PyThread_get_thread_ident();//调用pthread_self获取当前线程id
    _PyThreadState_Init(tstate);
    PyEval_AcquireThread(tstate);//竞争GIL并将_PyRuntime.gilstate.tstate_current设置为当前线程
    tstate->interp->num_threads++;//线程数加一
    res = PyObject_Call(boot->func, boot->args, boot->keyw);//执行用户代码，最终进入_PyEval_EvalFrameDefault字节码执行引擎
    if (res == NULL) {//线程相关异常处理
        if (PyErr_ExceptionMatches(PyExc_SystemExit))
            PyErr_Clear();
        else {
            PyObject *file;
            PyObject *exc, *value, *tb;
            PySys_WriteStderr(
                "Unhandled exception in thread started by ");
            PyErr_Fetch(&exc, &value, &tb);
            file = _PySys_GetObjectId(&PyId_stderr);
            if (file != NULL && file != Py_None)
                PyFile_WriteObject(boot->func, file, 0);
            else
                PyObject_Print(boot->func, stderr, 0);
            PySys_WriteStderr("\n");
            PyErr_Restore(exc, value, tb);
            PyErr_PrintEx(0);
        }
    }
    else
        Py_DECREF(res);
    Py_DECREF(boot->func);//释放相关数据结构
    Py_DECREF(boot->args);
    Py_XDECREF(boot->keyw);
    PyMem_DEL(boot_raw);
    tstate->interp->num_threads--;//线程数减一
    PyThreadState_Clear(tstate);//线程资源清理
    PyThreadState_DeleteCurrent();
    PyThread_exit_thread();
}
~~~

t_bootstrap中，在用户代码执行完毕之后就开始了线程的销毁工作。

~~~C
void
PyThreadState_Clear(PyThreadState *tstate)
{
    int verbose = tstate->interp->core_config.verbose;

    if (verbose && tstate->frame != NULL)
        fprintf(stderr,
          "PyThreadState_Clear: warning: thread still has a frame\n");

    Py_CLEAR(tstate->frame);//线程相关资源清理

    Py_CLEAR(tstate->dict);
    Py_CLEAR(tstate->async_exc);

    /*...*/

    Py_CLEAR(tstate->coroutine_wrapper);
    Py_CLEAR(tstate->async_gen_firstiter);
    Py_CLEAR(tstate->async_gen_finalizer);

    Py_CLEAR(tstate->context);
}

void
PyThreadState_DeleteCurrent()
{
    PyThreadState *tstate = _PyThreadState_GET();
    if (tstate == NULL)
        Py_FatalError(
            "PyThreadState_DeleteCurrent: no current tstate");
    tstate_delete_common(tstate);//调整线程链表，将当前线程从链表中删除
    if (_PyRuntime.gilstate.autoInterpreterState &&
        PyThread_tss_get(&_PyRuntime.gilstate.autoTSSkey) == tstate)
    {
        PyThread_tss_set(&_PyRuntime.gilstate.autoTSSkey, NULL);
    }
    _PyThreadState_SET(NULL);
    PyEval_ReleaseLock();//释放GIL
}

void
PyThread_exit_thread(void)
{
    dprintf(("PyThread_exit_thread called\n"));
    if (!initialized)
        exit(0);
    pthread_exit(0);//线程退出
}
~~~

### 线程的标准调度

当子线程和主线程都进入了\_PyEval\_EvalFrameDefault字节码执行引擎之后，python线程之间的切换就完全由python的线程调度机制掌控。之前在对\_PyEval\_EvalFrameDefault进行分析时，我们只考虑了执行机制，而没有考虑调度机制。下面我们给出加入了调度机制的\_PyEval_EvalFrameDefault框架。

~~~C
main_loop:
    for (;;) {
        if (_Py_atomic_load_relaxed(&_PyRuntime.ceval.eval_breaker)) {//[A]如果eval_breaker被设置为1
            opcode = _Py_OPCODE(*next_instr);
            if (opcode == SETUP_FINALLY ||
                opcode == SETUP_WITH ||
                opcode == BEFORE_ASYNC_WITH ||
                opcode == YIELD_FROM) {
                goto fast_next_opcode;//如果是上述字节码则跳过调度转入fast_next_opcode
            }
            if (_Py_atomic_load_relaxed(
                        &_PyRuntime.ceval.signals_pending))
            {
                if (handle_signals() != 0) {
                    goto error;
                }
            }
            if (_Py_atomic_load_relaxed(
                        &_PyRuntime.ceval.pending.calls_to_do))
            {
                if (make_pending_calls() != 0) {
                    goto error;
                }
            }

            if (_Py_atomic_load_relaxed(
                        &_PyRuntime.ceval.gil_drop_request))//其他线程请求释放GIL
            {
                /* Give another thread a chance */
                if (PyThreadState_Swap(NULL) != tstate)//将当前进程变量_PyRuntime.gilstate.tstate_current置为NULL
                    Py_FatalError("ceval: tstate mix-up");
                drop_gil(tstate);//释放GIL

                /* Other threads may run now */

                take_gil(tstate);//重新竞争GIL

                /* Check if we should make a quick exit. *///快速退出检测
                if (_Py_IsFinalizing() &&
                    !_Py_CURRENTLY_FINALIZING(tstate))
                {
                    drop_gil(tstate);//释放GIL
                    PyThread_exit_thread();//线程终止
                }

                if (PyThreadState_Swap(tstate) != NULL)//将_PyRuntime.gilstate.tstate_current设置为当前进程
                    Py_FatalError("ceval: orphan tstate");
            }
            /* Check for asynchronous exceptions. */
        }

    fast_next_opcode:
        switch (opcode) {
~~~

在上述代码中我们可以看到，如果eval_breaker被设置为1，那么每当执行到代码[A]处时，python虚拟机就会进入调度流程。我们来想象一下python线程的调度过程——首先，我们假设主线程持有GIL锁，子线程处在等待状态。此时，子线程正处在take_gile中等待GIL。如果子线程已经等待超过5ms，那么COND_TIMED_WAIT就会因为超时而返回，此时，子线程将执行SET_GIL_DROP_REQUEST发送请求释放GIL的信号。如果主线程执行到了代码[A]处，那么就会进入调度流程，释放GIL，使得子线程获得执行机会。

值得注意的是，python中并不是每执行一条指令都要进行调度判断。从源码中可以看到，python虚拟机在执行完一条字节之后有两种跳转方式DISPATCH与FAST_DISPATCH。

~~~C
#define DISPATCH() continue
#define FAST_DISPATCH() goto fast_next_opcode
~~~

当python完成一条字节码之后，如果执行的是FAST\_DISPATCH，那么就不会触发标准调度机制。

### 线程的阻塞调度

在程序的执行过程中，除了需要标准调度以外，还需要其他调度机制。比如，如果线程A调用了sleep函数将自身挂起，那么线程A应该主动释放GIL，使其他等待GIL的线程获得执行机会。

我们以time.sleep为例对python的阻塞调度机制进行分析。

sleep函数对应到time模块中的time_sleep函数，time模块所对应的实现是timemodule.c。当调用time.sleep之后，会依次调用time\_sleep->pysleep->平台相关时延函数，在这里我们以posix系统为例。

~~~C
        if (_PyTime_AsTimeval(secs, &timeout, _PyTime_ROUND_CEILING) < 0)
            return -1;

        Py_BEGIN_ALLOW_THREADS
        err = select(0, (fd_set *)0, (fd_set *)0, (fd_set *)0, &timeout);
        Py_END_ALLOW_THREADS

        if (err == 0)
            break;

        if (errno != EINTR) {
            PyErr_SetFromErrno(PyExc_OSError);
            return -1;
        }
~~~

可以看到，在进入select系统调用前后，python分别执行了两个宏——Py_BEGIN_ALLOW_THREADS与Py_END_ALLOW_THREADS，这两个宏就是阻塞调度的关键。

~~~C
#define Py_BEGIN_ALLOW_THREADS { \
                        PyThreadState *_save; \
                        _save = PyEval_SaveThread();
#define Py_END_ALLOW_THREADS    PyEval_RestoreThread(_save); \
                 }

PyThreadState *
PyEval_SaveThread(void)
{
    PyThreadState *tstate = PyThreadState_Swap(NULL);
    if (tstate == NULL)
        Py_FatalError("PyEval_SaveThread: NULL tstate");
    assert(gil_created());
    drop_gil(tstate);//释放GIL
    return tstate;
}

void
PyEval_RestoreThread(PyThreadState *tstate)
{
    if (tstate == NULL)
        Py_FatalError("PyEval_RestoreThread: NULL tstate");
    assert(gil_created());

    int err = errno;
    take_gil(tstate);//获取GIL
    /* _Py_Finalizing is protected by the GIL */
    if (_Py_IsFinalizing() && !_Py_CURRENTLY_FINALIZING(tstate)) {
        drop_gil(tstate);
        PyThread_exit_thread();
        Py_UNREACHABLE();
    }
    errno = err;

    PyThreadState_Swap(tstate);
}
~~~

在Py_BEGIN_ALLOW_THREADS，python释放了GIL，而在Py_END_ALLOW_THREADS中，python重新竞争了GIL。在子线程调用了

Py_BEGIN_ALLOW_THREADS之后，一直到子线程调用Py_END_ALLOW_THREADS之前，子线程与主线程都可能会被操作系统的线程调度机制选中。这意味着在某系情况下，python的线程可以脱离GIL的控制。然而在Py_BEGIN_ALLOW_THREADS与Py_END_ALLOW_THREADS之间，python并没有调用任何的C API，只是调用了操作系统API，这不会导致共享资源的访问冲突，所以依然是线程安全的。

### 用户级别的线程互斥与同步

python虚拟机通过GIL实现了对虚拟机一级的共享资源的保护，但是这种保护机制是用户不能控制的。为了实现对用户资源的保护，我们需要一种可控的互斥机制——用户级别互斥。在\_thread库中提供了lock对象来实现这一目标。

~~~python
>>> import _thread
>>> lock = _thread.allocate()
>>> lock
<unlocked _thread.lock object at 0x00000217535F9420>
>>> lock.acquire()
True
>>> lock
<locked _thread.lock object at 0x00000217535F9420>
>>> lock.release()
>>> lock
<unlocked _thread.lock object at 0x00000217535F9420>
~~~

我们从\_thread.allocate入手进行分析。

~~~C
static PyObject *
thread_PyThread_allocate_lock(PyObject *self, PyObject *Py_UNUSED(ignored))
{
    return (PyObject *) newlockobject();
}

static lockobject *
newlockobject(void)
{
    lockobject *self;
    self = PyObject_New(lockobject, &Locktype);
    if (self == NULL)
        return NULL;
    self->lock_lock = PyThread_allocate_lock();//创建lock对象
    self->locked = 0;
    self->in_weakreflist = NULL;
    if (self->lock_lock == NULL) {
        Py_DECREF(self);
        PyErr_SetString(ThreadError, "can't allocate lock");
        return NULL;
    }
    return self;
}

PyThread_type_lock
PyThread_allocate_lock(void)
{
    pthread_lock *lock;
    int status, error = 0;

    dprintf(("PyThread_allocate_lock called\n"));
    if (!initialized)
        PyThread_init_thread();

    lock = (pthread_lock *) PyMem_RawMalloc(sizeof(pthread_lock));
    if (lock) {
        memset((void *)lock, '\0', sizeof(pthread_lock));
        lock->locked = 0;

        status = pthread_mutex_init(&lock->mut,
                                    pthread_mutexattr_default);//创建互斥锁
        CHECK_STATUS_PTHREAD("pthread_mutex_init");
        /* Mark the pthread mutex underlying a Python mutex as
           pure happens-before.  We can't simply mark the
           Python-level mutex as a mutex because it can be
           acquired and released in different threads, which
           will cause errors. */
        _Py_ANNOTATE_PURE_HAPPENS_BEFORE_MUTEX(&lock->mut);

        status = pthread_cond_init(&lock->lock_released,
                                   pthread_condattr_default);//创建条件变量
        CHECK_STATUS_PTHREAD("pthread_cond_init");

        if (error) {
            PyMem_RawFree((void *)lock);
            lock = 0;
        }
    }

    dprintf(("PyThread_allocate_lock() -> %p\n", lock));
    return (PyThread_type_lock) lock;
}

typedef void *PyThread_type_lock;

typedef struct {
    char             locked; /* 0=unlocked, 1=locked */
    /* a <cond, mutex> pair to handle an acquire of a locked lock */
    pthread_cond_t   lock_released;
    pthread_mutex_t  mut;
} pthread_lock;
~~~

可以看到，thread_PyThread_allocate_lock调用newlockobject创建了一个lockobject对象，而这个lockobject的lock_lock又是通过

PyThread_allocate_lock分配的。我们来看一下lockobject所提供的属性集合。

~~~C
static PyMethodDef lock_methods[] = {
    {"acquire_lock", (PyCFunction)(void(*)(void))lock_PyThread_acquire_lock,
     METH_VARARGS | METH_KEYWORDS, acquire_doc},
    {"acquire",      (PyCFunction)(void(*)(void))lock_PyThread_acquire_lock,
     METH_VARARGS | METH_KEYWORDS, acquire_doc},
    {"release_lock", (PyCFunction)lock_PyThread_release_lock,
     METH_NOARGS, release_doc},
    {"release",      (PyCFunction)lock_PyThread_release_lock,
     METH_NOARGS, release_doc},
    {"locked_lock",  (PyCFunction)lock_locked_lock,
     METH_NOARGS, locked_doc},
    {"locked",       (PyCFunction)lock_locked_lock,
     METH_NOARGS, locked_doc},
    {"__enter__",    (PyCFunction)(void(*)(void))lock_PyThread_acquire_lock,
     METH_VARARGS | METH_KEYWORDS, acquire_doc},
    {"__exit__",    (PyCFunction)lock_PyThread_release_lock,
     METH_VARARGS, release_doc},
    {NULL,           NULL}              /* sentinel */
};
~~~

lockobject提供的操作很简单——acquire、release和判断当前lock是否被锁住 的locked。

我们首先来看申请锁的动作——lock_PyThread_acquire_lock。

~~~C
static PyObject *
lock_PyThread_acquire_lock(lockobject *self, PyObject *args, PyObject *kwds)
{
    _PyTime_t timeout;
    PyLockStatus r;
    /*参数解析
    	acquire(blocking=True, timeout=-1) -> bool
    	根据blocking与timeout的不同情况进行参数解析
    	blocking==True
    		timeout >= 0
    			挂起timeout秒若未获取到锁则超时
    		timeout == -1
    			挂起直到获取锁
    		timeout < 0 and timeout != -1
    			出错
    	blocking==False
    		timeout != -1
    			出错
    		timeout == -1
    			立即返回
    */
    if (lock_acquire_parse_args(args, kwds, &timeout) < 0)
        return NULL;

    r = acquire_timed(self->lock_lock, timeout);//竞争lock
    if (r == PY_LOCK_INTR) {
        return NULL;
    }

    if (r == PY_LOCK_ACQUIRED)
        self->locked = 1;
    return PyBool_FromLong(r == PY_LOCK_ACQUIRED);
}
~~~

真正的竞争lock的动作是在acquire_timed中发生的。

~~~C
static PyLockStatus//如果锁等被信号中断，则运行信号处理程序，如果信号处理过程中抛出异常，则返回PY_LOCK_INTR；否则根据是否在超时时间内获取锁返回PY_LOCK_ACQUIRED或者PY_LOCK_FAILURE
acquire_timed(PyThread_type_lock lock, _PyTime_t timeout)
{
    PyLockStatus r;
    _PyTime_t endtime = 0;
    _PyTime_t microseconds;

    if (timeout > 0)
        endtime = _PyTime_GetMonotonicClock() + timeout;

    do {
        microseconds = _PyTime_AsMicroseconds(timeout, _PyTime_ROUND_CEILING);

        /* first a simple non-blocking try without releasing the GIL */
        r = PyThread_acquire_lock_timed(lock, 0, 0);//尝试性地获取锁
        if (r == PY_LOCK_FAILURE && microseconds != 0) {
            Py_BEGIN_ALLOW_THREADS//[A]释放GIL允许其他线程调度
            r = PyThread_acquire_lock_timed(lock, microseconds, 1);//获取锁
            Py_END_ALLOW_THREADS//[B]竞争GIL
        }

        if (r == PY_LOCK_INTR) {
            /* Run signal handlers if we were interrupted.  Propagate
             * exceptions from signal handlers, such as KeyboardInterrupt, by
             * passing up PY_LOCK_INTR.  */
            if (Py_MakePendingCalls() < 0) {
                return PY_LOCK_INTR;
            }

            /* If we're using a timeout, recompute the timeout after processing
             * signals, since those can take time.  */
            if (timeout > 0) {
                timeout = endtime - _PyTime_GetMonotonicClock();

                /* Check for negative values, since those mean block forever.
                 */
                if (timeout < 0) {
                    r = PY_LOCK_FAILURE;
                }
            }
        }
    } while (r == PY_LOCK_INTR);  /* Retry if we were interrupted. */

    return r;
}

PyLockStatus
PyThread_acquire_lock_timed(PyThread_type_lock lock, PY_TIMEOUT_T microseconds,
                            int intr_flag)
{
    PyLockStatus success = PY_LOCK_FAILURE;
    pthread_lock *thelock = (pthread_lock *)lock;
    int status, error = 0;

    dprintf(("PyThread_acquire_lock_timed(%p, %lld, %d) called\n",
             lock, microseconds, intr_flag));

    if (microseconds == 0) {
        status = pthread_mutex_trylock( &thelock->mut );//非阻塞
        if (status != EBUSY)
            CHECK_STATUS_PTHREAD("pthread_mutex_trylock[1]");
    }
    else {
        status = pthread_mutex_lock( &thelock->mut );//阻塞
        CHECK_STATUS_PTHREAD("pthread_mutex_lock[1]");
    }
    if (status == 0) {
        if (thelock->locked == 0) {
            success = PY_LOCK_ACQUIRED;
        }//当前锁已被占有，进入等待流程
        else if (microseconds != 0) {
            struct timespec ts;
            if (microseconds > 0)
                MICROSECONDS_TO_TIMESPEC(microseconds, ts);
            /* continue trying until we get the lock */

            /* mut must be locked by me -- part of the condition
             * protocol */
            while (success == PY_LOCK_FAILURE) {
                if (microseconds > 0) {
                    status = pthread_cond_timedwait(//将当前线程挂起并释放互斥锁
                        &thelock->lock_released,
                        &thelock->mut, &ts);
                    if (status == ETIMEDOUT)
                        break;
                    CHECK_STATUS_PTHREAD("pthread_cond_timed_wait");
                }
                else {
                    status = pthread_cond_wait(//将当前线程挂起并释放互斥锁
                        &thelock->lock_released,
                        &thelock->mut);
                    CHECK_STATUS_PTHREAD("pthread_cond_wait");
                }

                if (intr_flag && status == 0 && thelock->locked) {
                    /* We were woken up, but didn't get the lock.  We probably received
                     * a signal.  Return PY_LOCK_INTR to allow the caller to handle
                     * it and retry.  */
                    success = PY_LOCK_INTR;
                    break;
                }
                else if (status == 0 && !thelock->locked) {//成功
                    success = PY_LOCK_ACQUIRED;
                }
            }
        }
        if (success == PY_LOCK_ACQUIRED) thelock->locked = 1;//成功
        status = pthread_mutex_unlock( &thelock->mut );//释放互斥锁
        CHECK_STATUS_PTHREAD("pthread_mutex_unlock[1]");
    }

    if (error) success = PY_LOCK_FAILURE;
    dprintf(("PyThread_acquire_lock_timed(%p, %lld, %d) -> %d\n",
             lock, microseconds, intr_flag, success));
    return success;
}
~~~

在代码[A]与代码[B]处，为了避免死锁，我们同样看到了释放以及竞争GIL的代码。现在，我们可以轻易地猜出lock的release操作所完成的操作。

~~~C
void
PyThread_release_lock(PyThread_type_lock lock)
{
    pthread_lock *thelock = (pthread_lock *)lock;
    int status, error = 0;

    (void) error; /* silence unused-but-set-variable warning */
    dprintf(("PyThread_release_lock(%p) called\n", lock));

    status = pthread_mutex_lock( &thelock->mut );
    CHECK_STATUS_PTHREAD("pthread_mutex_lock[3]");

    thelock->locked = 0;

    /* wake up someone (anyone, if any) waiting on the lock */
    status = pthread_cond_signal( &thelock->lock_released );//唤醒其他等待的线程
    CHECK_STATUS_PTHREAD("pthread_cond_signal");

    status = pthread_mutex_unlock( &thelock->mut );//释放互斥锁
    CHECK_STATUS_PTHREAD("pthread_mutex_unlock[3]");
}
~~~
