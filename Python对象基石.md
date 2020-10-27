## Python对象基石

### 基本对象PyObject

```C
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;//引用计数
    struct _typeobject *ob_type;//类型信息
} PyObject;
```

### 变长对象PyVarObject

```C
typedef struct {
    PyObject ob_base;//基本对象类型
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```

![image-20201022230502367](/home/jyj6536/文档/笔记/Python源码学习/Python对象基石.assets/image-20201022230502367.png "基本对象与变长对象")

## PyType_Type与其他对象

![image-20201024201503278](/home/jyj6536/文档/笔记/Python源码学习/Python对象基石.assets/image-20201024201503278.png)

## Python中long/int对象的内存布局

![image-20201024225006234](/home/jyj6536/文档/笔记/Python源码学习/Python对象基石.assets/image-20201024225006234.png)

### Python中的整数存储方法

#### 整数0

ob_size为0代表整数0，这是一种特殊情况．

![image-20201026224802741](/home/jyj6536/文档/笔记/Python源码学习/Python对象基石.assets/image-20201026224802741.png)

#### 整数1

在python中digit的定义是32位(uint32_t)还是16(unsigned short)位取决于操作系统，为了讨论方便，我们将```PYLONG_BITS_IN_DIGIT```的值定义为15

```C
#if PYLONG_BITS_IN_DIGIT == 30
typedef uint32_t digit;
typedef int32_t sdigit; /* signed variant of digit */
typedef uint64_t twodigits;
typedef int64_t stwodigits; /* signed variant of twodigits */
#define PyLong_SHIFT    30
#define _PyLong_DECIMAL_SHIFT   9 /* max(e such that 10**e fits in a digit) */
#define _PyLong_DECIMAL_BASE    ((digit)1000000000) /* 10 ** DECIMAL_SHIFT */
#elif PYLONG_BITS_IN_DIGIT == 15
typedef unsigned short digit;
typedef short sdigit; /* signed variant of digit */
typedef unsigned long twodigits;
typedef long stwodigits; /* signed variant of twodigits */
#define PyLong_SHIFT    15
#define _PyLong_DECIMAL_SHIFT   4 /* max(e such that 10**e fits in a digit) */
#define _PyLong_DECIMAL_BASE    ((digit)10000) /* 10 ** DECIMAL_SHIFT */
```

当我们需要存储整数1的时候，ob_size的值变成了1表示ob_digit的长度为1，ob_digit以unsigned short的方式存储了整数1．

![image-20201026225953081](/home/jyj6536/文档/笔记/Python源码学习/Python对象基石.assets/image-20201026225953081.png)

#### 整数-1

ob_size的正负代表整数的正负，ob_size为负值代表整数为负值．

![image-20201026230223083](/home/jyj6536/文档/笔记/Python源码学习/Python对象基石.assets/image-20201026230223083.png)



### 小整数对象池smallints

在Python中，针对数值比较小的整数使用了对象池技术以避免频繁地在堆上申请内存．该整数池的定义如下

```c
static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];
```

其中，NSMALLNEGINTS与NSMALLPOSINTS定义了小整数的范围－－[NSMALLNEGINTS,NSMALLPOSINTS)

```c
#define NSMALLPOSINTS           257
#define NSMALLNEGINTS           5
```

samll_ints的初始化代码如下

```C
int
_PyLong_Init(void)
{
#if NSMALLNEGINTS + NSMALLPOSINTS > 0
    int ival, size;
    PyLongObject *v = small_ints;

    for (ival = -NSMALLNEGINTS; ival <  NSMALLPOSINTS; ival++, v++) {
        size = (ival < 0) ? -1 : ((ival == 0) ? 0 : 1);//负数longobject对象的ob_size为负值，0的ob_size值0,正数的ob_size为正值
        if (Py_TYPE(v) == &PyLong_Type) {
            /* The element is already initialized, most likely
             * the Python interpreter was initialized before.
             */
            Py_ssize_t refcnt;
            PyObject* op = (PyObject*)v;

            refcnt = Py_REFCNT(op) < 0 ? 0 : Py_REFCNT(op);
            _Py_NewReference(op);
            /* _Py_NewReference sets the ref count to 1 but
             * the ref count might be larger. Set the refcnt
             * to the original refcnt + 1 */
            Py_REFCNT(op) = refcnt + 1;
            assert(Py_SIZE(op) == size);
            assert(v->ob_digit[0] == (digit)abs(ival));
        }
        else {
            (void)PyObject_INIT(v, &PyLong_Type);
        }
        Py_SIZE(v) = size;
        v->ob_digit[0] = (digit)abs(ival);
    }
```

当需要使用小整数对象时，Python使用get_small_int从small_ints缓冲池中获取

```c
static PyObject *
get_small_int(sdigit ival)
{
    PyObject *v;
    assert(-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS);
    v = (PyObject *)&small_ints[ival + NSMALLNEGINTS];
    Py_INCREF(v);
#ifdef COUNT_ALLOCS
    if (ival >= 0)
        _Py_quick_int_allocs++;
    else
        _Py_quick_neg_int_allocs++;
#endif
    return v;
}
```

