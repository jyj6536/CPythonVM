## Python虚拟机中的函数机制

根据“一切皆对象”的原则，python中的函数也应该是一个对象。在python中，函数机制是通过python对象——PyFunctionObject来实现的。

~~~C
typedef struct {
    PyObject_HEAD
    PyObject *func_code;        /* A code object, the __code__ attribute *///对应编译后的PyCodeObject对象
    PyObject *func_globals;     /* A dictionary (other mappings won't do) *///函数运行时的globals名字空间
    PyObject *func_defaults;    /* NULL or a tuple *///默认参数
    PyObject *func_kwdefaults;  /* NULL or a dict *///关键字参数
    PyObject *func_closure;     /* NULL or a tuple of cell objects *///用于实现闭包
    PyObject *func_doc;         /* The __doc__ attribute, can be anything *///函数的文档
    PyObject *func_name;        /* The __name__ attribute, a string object *///函数__name__属性
    PyObject *func_dict;        /* The __dict__ attribute, a dict or NULL *///函数的__dict__属性
    PyObject *func_weakreflist; /* List of weak references */
    PyObject *func_module;      /* The __module__ attribute, can be anything *///函数的__module__属性
    PyObject *func_annotations; /* Annotations, a dict or NULL */
    PyObject *func_qualname;    /* The qualified name */

    /* Invariant:
     *     func_closure contains the bindings for func_code->co_freevars, so
     *     PyTuple_Size(func_closure) == PyCode_GetNumFree(func_code)
     *     (func_closure may be NULL if PyCode_GetNumFree(func_code) == 0).
     */
} PyFunctionObject;
~~~



