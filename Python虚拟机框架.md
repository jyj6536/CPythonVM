# Python虚拟机框架

### x86下函数调用时栈帧的变化

Python虚拟机运行字节码的过程实际上是模拟操作系统运行可执行文件的过程。所以首先我们需要了解在普通的x86机器上，可执行文件以一种什么方式运行。在这里，我们主要关注运行时栈帧的变化。

C语言代码如下：

```C
int add(int a,int b)
{
    int c = a;
    return c+b;
}

int main()
{
	int x = 1,y = 2,sum;
	sum = add(x,y);
	return 0;
}
```

在进入函数add的过程就是为add构建栈帧的过程，首先要把参数入栈

![](Python虚拟机框架.assets/image-20201114162915393.png)

为了能够从add流程返回main流程，还需要把返回地址入栈

![image-20201114164322697](Python虚拟机框架.assets/image-20201114164322697.png)

开始构建add的栈帧，将ebp入栈

![image-20201114164532805](Python虚拟机框架.assets/image-20201114164532805.png)

局部变量以及需要保护的寄存器等入栈．

![image-20201114170030870](Python虚拟机框架.assets/image-20201114170030870.png)

此时，add的栈帧构建完成，开始执行add内部的流程．当add的内部流程执行完成之后，将esp与ebp还原为add栈帧创建之前的值，将返回地址压入eip，从而继续在main中的流程．

### PyFrameObject

Python中通过PyFrameObject来模拟栈帧，实现了Python字节码的有序执行．

```c
typedef struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    /* Call PyFrame_GetLineNumber() instead of reading this field
       directly.  As of 2.3 f_lineno is only valid when tracing is
       active (i.e. when f_trace is set).  At other times we use
       PyCode_Addr2Line to calculate the line from the current
       bytecode index. */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
} PyFrameObject;
```

+ f_back：指向执行环境链的上一个栈帧，使得当前栈帧在执行结束之后能顺利返回旧栈帧之中．
+ f_code：该frame对象所对应的code对象．
+ f_builtins，f_globals，f_locals：三个独立的名字空间．
+ f_valuestack与f_stacktop：分别指向运行时栈的栈底和栈顶．
+ f_lasti：当前已经执行过的字节码指令的位置．

PyFrameObject 是一个变长对象，其变长部分为f_localsplus．从注释可以看出，f_localsplus由locals与stack两部分组成．

+ locals部分用来保存局部变量．
+ stack指的是“运行时栈”，是在执行运算时所需要的内存空间，这个栈的大小存储在f_code.co_stacksize中，编译PyCodeObject对象的时候会计算出来，由于不同Code Block在执行时所需的栈空间的大小是不同的，所以决定了PyFrameObject的开头一定有一个PyObject_VAR_HEAD

某个时刻下，Python的运行时环境．

![image-20201115103138774](Python虚拟机框架.assets/image-20201115103138774.png)

### 动态内存空间的申请

```C
        Py_ssize_t extras, ncells, nfrees;
        ncells = PyTuple_GET_SIZE(code->co_cellvars);
        nfrees = PyTuple_GET_SIZE(code->co_freevars);
        extras = code->co_stacksize + code->co_nlocals + ncells +
            nfrees;
        if (free_list == NULL) {
            f = PyObject_GC_NewVar(PyFrameObject, &PyFrame_Type,
            extras);
            if (f == NULL) {
                Py_DECREF(builtins);
                return NULL;
            }
        }
        else {
            assert(numfree > 0);
            --numfree;
            f = free_list;
            free_list = free_list->f_back;
            if (Py_SIZE(f) < extras) {
                PyFrameObject *new_f = PyObject_GC_Resize(PyFrameObject, f, extras);
                if (new_f == NULL) {
                    PyObject_GC_Del(f);
                    Py_DECREF(builtins);
                    return NULL;
                }
                f = new_f;
            }
            _Py_NewReference((PyObject *)f);
        }

        f->f_code = code;
        extras = code->co_nlocals + ncells + nfrees;
        f->f_valuestack = f->f_localsplus + extras;
        for (i=0; i<extras; i++)
            f->f_localsplus[i] = NULL;
        f->f_locals = NULL;
        f->f_trace = NULL;
    }
    f->f_stacktop = f->f_valuestack;
```

可以看出，frame对象也使用了内存池机制．

frame对象中动态大小的内存由两部分组成：

+ 局部变量locals
  + code->co_nclocals Code Block中局部变量的个数
  + code->co_cellvars Code Block中中嵌套函数所引用的局部变量名集合
  + code->co_freevars Code Block 中的自由变量名集合
+ 运行时栈stack code->co_stacksize

f_valuestack维护栈底，f_stacktop维护栈顶．

新创建的frame对象的内存布局如下图所示

![image-20201115115451527](Python虚拟机框架.assets/image-20201115115451527.png)

