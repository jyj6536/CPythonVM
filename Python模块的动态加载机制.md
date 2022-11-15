# Python模块的动态加载机制

模块的实现机制是python中的重要话题。在这里我们将对python中一个模块是如何与另一个模块进行交互进行剖析。

## 最简单的import语句

~~~python
import sys
# python字节码
  1           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (None)
              4 IMPORT_NAME              0 (sys)
              6 STORE_NAME               0 (sys)
              8 LOAD_CONST               1 (None)
             10 RETURN_VALUE
~~~

在对sys模块进行“import”操作时，python虚拟机首先通过两条load_const指令将0与None压入运行时栈中，然后通过import_name指令引入了sys模块并压入栈顶，最后通过store_name指令将sys模块存储到了当前frame对象的local名字空间中。

在上述字节码中，除了import_name以外都是我们很熟悉的字节码了。我们来看一下import_name的实现。

~~~C
case TARGET(IMPORT_NAME): {
    PyObject *name = GETITEM(names, oparg);//从co_name中取出要引入的模块名sys
    PyObject *fromlist = POP();//弹出栈顶元素（在这里是None）
    PyObject *level = TOP();//获取栈顶元素（在这里是0）
    PyObject *res;
    res = import_name(f, name, fromlist, level)//调用import_name函数
    Py_DECREF(level);
    Py_DECREF(fromlist);
    SET_TOP(res);//将获取到的模块对象设置到栈顶
    if (res == NULL)
        goto error;
    DISPATCH();
}
~~~

import_name的字节码中的各个变量的值已经在注释中给出，这些变量最终都被传给了import_name这个函数，模块导入的关键就在import_name这个函数中。我们来看一下该函数的实现。

~~~C
static PyObject *
import_name(PyFrameObject *f, PyObject *name, PyObject *fromlist, PyObject *level)
{
    _Py_IDENTIFIER(__import__);
    PyObject *import_func, *res;
    PyObject* stack[5];

    import_func = _PyDict_GetItemId(f->f_builtins, &PyId___import__);//获取bilitins名字空间中的__import__方法
    if (import_func == NULL) {
        PyErr_SetString(PyExc_ImportError, "__import__ not found");
        return NULL;
    }

    /* Fast path for not overloaded __import__. */
    if (import_func == _PyInterpreterState_GET_UNSAFE()->import_func) {
        int ilevel = _PyLong_AsInt(level);
        if (ilevel == -1 && PyErr_Occurred()) {
            return NULL;
        }
        res = PyImport_ImportModuleLevelObject(//__import__中最终调用的是这个函数，如果__import__没有被重载的话就直接进行快速调用
                        name,
                        f->f_globals,
                        f->f_locals == NULL ? Py_None : f->f_locals,
                        fromlist,
                        ilevel);
        return res;
    }

    Py_INCREF(import_func);

    stack[0] = name;
    stack[1] = f->f_globals;
    stack[2] = f->f_locals == NULL ? Py_None : f->f_locals;
    stack[3] = fromlist;//from ... import ...语句时生效，为被引入的子模块名被组装成一个tuple
    stack[4] = level;//0表示使用绝对导入。level的正值表示相对于调用__import__的模块所在目录要搜索的父目录级数
    res = _PyObject_FastCall(import_func, stack, 5);//在这里将参数打包并调用__import__方法
    Py_DECREF(import_func);
    return res;
}

PyObject *
PyImport_ImportModuleLevelObject(PyObject *name, PyObject *globals,
                                 PyObject *locals, PyObject *fromlist,
                                 int level)
{
    _Py_IDENTIFIER(_handle_fromlist);
    PyObject *abs_name = NULL;
    PyObject *final_mod = NULL;
    PyObject *mod = NULL;
    PyObject *package = NULL;
    PyInterpreterState *interp = _PyInterpreterState_GET_UNSAFE();
    int has_from;
	//名字不能为空
    if (name == NULL) {
        PyErr_SetString(PyExc_ValueError, "Empty module name");
        goto error;
    }

    /* The below code is importlib.__import__() & _gcd_import(), ported to C
       for added performance. */
	//名字必须为PyUnicode对象
    if (!PyUnicode_Check(name)) {
        PyErr_SetString(PyExc_TypeError, "module name must be a string");
        goto error;
    }
    if (PyUnicode_READY(name) < 0) {
        goto error;
    }
    if (level < 0) {//level必须大于等于0
        PyErr_SetString(PyExc_ValueError, "level must be >= 0");
        goto error;
    }

    if (level > 0) {
        abs_name = resolve_name(name, globals, level);
        if (abs_name == NULL)
            goto error;
    }
    else {  /* level == 0 */
        if (PyUnicode_GET_LENGTH(name) == 0) {
            PyErr_SetString(PyExc_ValueError, "Empty module name");
            goto error;
        }
        abs_name = name;
        Py_INCREF(abs_name);
    }
    
    //首先从sys.modules中获取；python中，一个模块不会被重复导入，一旦导入之后会放到sys.modules中
    mod = PyImport_GetModule(abs_name);
    if (mod != NULL && mod != Py_None) {
       /*...*/
    }
    else {
        Py_XDECREF(mod);
        mod = import_find_and_load(abs_name);//执行导入动作
        if (mod == NULL) {
            goto error;
        }
    }

    /*...*/

  error:
    Py_XDECREF(abs_name);
    Py_XDECREF(mod);
    Py_XDECREF(package);
    if (final_mod == NULL)
        remove_importlib_frames();
    return final_mod;
}
~~~

从调用import_func开始，python开始了真正的模块导入动作。在对具体的导入动作进行分析之前，我们先从黑盒探测的角度，来对python的模块导入机制进行探测。

## 标准import

以引入sys模块为例，来查看import对名字空间的影响。

~~~python
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> import sys
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'sys']
>>> type(sys)
<class 'module'>
~~~

可以看到，在执行import sys后当前local名字空间中增加了一个“sys”符号，而通过type操作我们知道这个“sys”符号背后其实是一个module对象。然而，在之前对python初始化过程的分析中我们知道，在初始化时python已经将sys等一系列的module装入内存。实际上，为了保持名字空间的整洁，python并没有将这些符号暴露在当前的local名字空间中，而是需要用户显式地通过import机制通知python。

这些预加载的module都存放在sys.modules中。

~~~python
>>> os = sys.modules["os"]
>>> id(os)
2467195600960
>>> import os
>>> id(os)
2467195600960
~~~

从两个id的输出结果可以看出：sys.modules中的os与import引入的os是同一个module对象。

再来看一下用户自定义module引入时发生的动作。

~~~python
#hello.py
a = 1
b = 2
~~~

在这个简单的module中仅仅创建了两个符号——a与b。

~~~python
>>> import hello
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'hello']
>>> import sys
>>> sys.modules["hello"]
<module 'hello' from 'C:\\Users\\Administrator\\Documents\\test\\hello.py'>
>>> id(sys.modules["hello"])
1587332564448
>>> id(hello)
1587332564448
>>> dir(hello)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'a', 'b']
~~~

可以看出，import语句创建了一个新的module对象——hello，并且将hello放到了sys.modules以及local名字空间中。

同时，我们在hello的属性中发现了\_\_builtins\_\_这个符号，这个符号与local名字空间中的\_\_builtins\_\_并不是同一个。

~~~python
>>> import hello
>>> id(hello.__builtins__)
2895499107072
>>> id(__builtins__)
2895499094832
>>> id(__builtins__.__dict__)
2895499107072
~~~

显然，hello中的\_\_builtins\_\_正是local名字空间中\_\_builtins\_\_所维护的dict对象。

## 嵌套import

上面我们考察了对单个独立的module的import动作，下面我们来考察一下嵌套的import动作。所谓嵌套的import动作，就是指python在”import A“时，A这个module中又import了另一个module。

~~~python
# hello1.py
import hello2
#hello2.py
import sys
~~~

现在我们来看一下一系列的import动作对名字空间的影响。

~~~python
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> import hello1
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'hello1']
>>> dir(hello1)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'hello2']
>>> dir(hello1.hello2)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'sys']
~~~

可以看到，当前环境下的import动作只对当前名字空间有影响而不会影响上层名字空间。

~~~python
#对全局module集合的影响
>>> sys.modules["hello2"]
<module 'hello2' from 'C:\\Users\\Administrator\\Documents\\test\\hello2.py'>
~~~

可以看到，无论是从交互式环境直接“import”，还是从被“import”的模块中简介“import”，最终都会对全局的module集合——sys.modules产生影响。这样做的好处是显而易见的，即如果在其他地方再次import这个module，虚拟机只需要从全局的module集合缓存中返回该module即可。

## import package

python提供的package机制与module机制是类似的，逻辑上相关联的一些module应该聚合到一个package中，同时，小的package又可以聚合成一个较大的package。在3.3之前的版本中，python只支持一种模式的包导入机制——常规包（reqular package）。在这种机制下，包所在的目录中一定要有一个\_\_init_\_.py文件。在3.3及以上的版本中，python又增加了一种名字命名空间包（namespace package）的包机制。

### regular package

常规包通常是一个包含\_\_init\_\_.py的目录。当一个常规包被引入的时候，\_\_init\_\_.py会被执行，并且该文件中定义的对象也会被绑定到包的命名空间中。

~~~python
parent/
    __init__.py
    one/
        __init__.py
        one.py
    two/
        __init__.py
        two.py
    three/
        __init__.py
        three.py
#one.py
a=1
#two.py
b=2
#three.py
c=3
~~~

现在，我们假定四个\_\_init\_\_.py都是空文件，执行import来考察对python名字空间的影响。

~~~python
>>> import parent
>>> dir(parent)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__path__', '__spec__']
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'parent']
~~~

可以看到，“import parent”只加载了parent这个父级包，而并没有将parent的三个子级包同时加载。

~~~python
>>> import parent.one.one
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'parent']
>>> dir(parent)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__path__', '__spec__', 'one']
>>> dir(parent.one)
['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__path__', '__spec__', 'one']
>>> import sys
>>> sys.modules["parent"]
<module 'parent' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\__init__.py'>
>>> sys.modules["parent.one"]
<module 'parent.one' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\one\\__init__.py'>
>>> sys.modules["parent.one.one"]
<module 'parent.one.one' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\one\\one.py'>
~~~

可以看到，我们将parent.one.one这个模块加载之后，parent以及parent.one这两个包也被加载到了sys的module缓存中，这样做的好处在于当我们向加载parent下的其他包时，比如parent.two，可以避免对parent的再次加载；同时，在local名字空间中加载的是parent这个包，如果我们要再加载parent1.one.one这个模块的话，也可以避免名字冲突的问题。

在导入模块时，可以通过from与import结合更加精确地控制对象的加载。这样可以尽可能少地在当前名字空间中引入符号，从而更好地避免名字空间遭到污染。

~~~python
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> from parent import one
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'one']
>>> import sys
>>> sys.modules["parent"]
<module 'parent' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\__init__.py'>
>>> sys.modules["parent.one"]
<module 'parent.one' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\one\\__init__.py'>
~~~

可以看到，from A import B与import A.B的方式在本质上是一致的，区别在于导入模块时在local名字空间中所引入的符号。在import A.B中，虚拟机引入了符号“A”并将其映射到module A，而在from A import B中，虚拟机引入了符号“B”，并将其映射到了module A.B。

python中还可以通过from ... import *的方式将当一个模块中所有**非下划线开头**的成员都导入当前名字空间；而对于包中的子包，只有在\_\_all\_\_中的子包才会被import *引入；同时，如果一个模块中定义了\_\_all\_\_，那么import *就只会导入\_\_all\_\_中列出的成员。

~~~python
# 修改parent目录下的__init__.py
__all__ = ["one","two"]
>>> from parent import *
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'one', 'two']

# 修改parent/one目录下的__init__.py
__all__ = ["one"]
>>> from parent.one import *
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'one']
>>> one
<module 'parent.one.one' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\one\\one.py'>
~~~

在python的导入机制中，还支持通过as进行符号重命名的操作。

~~~python
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> import parent.one as One
>>> dir()
['One', '__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> One
<module 'parent.one' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\one\\__init__.py'>
>>> import sys
>>> sys.modules["parent.one"]
<module 'parent.one' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\one\\__init__.py'>
~~~

可以看到，以import A as B的形式加载时，虚拟机在local名字空间中引入的符号是as控制的符号B。虽然A依然在sys的模块缓存中，但是在本地名字空间中我们已经找不到A了。

为了使用一个module，我们必须通过import将其加载到内存中，使用了这个module之后，我们可能还需要将其删除。为了将其删除，我们通常使用del。

~~~python
>> import parent
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__', 'parent']
>>> del parent
>>> dir()
['__annotations__', '__builtins__', '__doc__', '__loader__', '__name__', '__package__', '__spec__']
>>> import sys
>>> sys.modules["parent"]
<module 'parent' from 'C:\\Users\\Administrator\\Documents\\test\\parent\\__init__.py'>
~~~

从上面的输出可以看到，我们使用del删除一个module的时候，只是从本地名字空间中删除了parent这个符号，而sys缓存中的parent模块并没有被删除。之所以python要采取这种看上去类似module池的import动作，一个重要原因就是一个完整系统的多个py文件都会对某个module进行import动作，希望使用这个module所提供的功能。到了这里，我们看到，python中的“动态加载”这个概念，他的真实含义是希望某个module能被感知，即**将某个module以某个符号的形式引入某个名字空间**。

### namespace packages

命名空间包是一种用于在磁盘上的多个目录之间拆分单个Python包的机制。通过命名空间包导入机制本身可以构建构成包的目录列表。与常规包相比，命名空间包中没有\_\_init\_\_.py文件。

假设我们有以下的目录结构

~~~shell
Lib/test/namespace_pkgs
    project1
        parent
            child
                one.py
    project2
        parent
            child
                two.py
~~~

在这类，parent和child包都是命名空间包：parent存在于不同的目录中并且都没有\_\_init\_\_.py文件。

我们将parent的父目录添加到sys.path中

~~~python
>>> import sys
>>> sys.path += ['Lib/test/namespace_pkgs/project1', 'Lib/test/namespace_pkgs/project2']
>>> import parent.child.one
>>> parent.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent', 'Lib/test/namespace_pkgs/project2/parent'])
>>> parent.child.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent/child', 'Lib/test/namespace_pkgs/project2/parent/child'])
>>> import parent.child.two
~~~

可以看到，我们成功构建了一个名为parent的命名空间包。

## import机制的实现

我们知道，python中import的起点是builtin模块中\_\_import\_\_操作，也就是builtin___import\_\_函数，我们以该函数为起点开始分析。

~~~C
static PyObject *
builtin___import__(PyObject *self, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"name", "globals", "locals", "fromlist",
                             "level", 0};
    PyObject *name, *globals = NULL, *locals = NULL, *fromlist = NULL;
    int level = 0;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "U|OOOi:__import__",//从args中解析出所需要的参数
                    kwlist, &name, &globals, &locals, &fromlist, &level))
        return NULL;
    return PyImport_ImportModuleLevelObject(name, globals, locals,
                                            fromlist, level);
}
~~~

在python中，PyArg_ParseTupleAndKeywords是一个被广泛使用的函数，其原型如下：

~~~c
PyArg_ParseTupleAndKeywords(PyObject* args,PyObject* kwds,const char *fromat,
                    char *keyword[],...)
~~~

该函数的目的是根据format中指定的格式将args与kwds解析成各种对象（python中的对象或者C的原生类型）。在**U|OOOi:\_\_import\_\_**这个format中，U代表目标是一个Unicode对象；i则用来将int对象解析为int类型的值；而O代表解析目标依然为python中的合法对象。至于“|”与“:”，“|”表示其后的格式字符是可选的；“:”表示格式字符串到此结束，其后的字符串用于在解析过程中输出错误信息使用。

完成参数解析之后，python进入了PyImport_ImportModuleLevelObject。我们继续对PyImport_ImportModuleLevelObject的调用栈进行追溯：

~~~C
PyObject *
PyImport_ImportModuleLevelObject(PyObject *name, PyObject *globals,
                                 PyObject *locals, PyObject *fromlist,
                                 int level)
{
    /*...*/

    mod = PyImport_GetModule(abs_name);
    if (mod != NULL && mod != Py_None) {
        _Py_IDENTIFIER(__spec__);
        _Py_IDENTIFIER(_lock_unlock_module);
        PyObject *spec;

        /* Optimization: only call _bootstrap._lock_unlock_module() if
           __spec__._initializing is true.
           NOTE: because of this, initializing must be set *before*
           stuffing the new module in sys.modules.
         */
        spec = _PyObject_GetAttrId(mod, &PyId___spec__);
        if (_PyModuleSpec_IsInitializing(spec)) {
            PyObject *value = _PyObject_CallMethodIdObjArgs(interp->importlib,
                                            &PyId__lock_unlock_module, abs_name,
                                            NULL);
            if (value == NULL) {
                Py_DECREF(spec);
                goto error;
            }
            Py_DECREF(value);
        }
        Py_XDECREF(spec);
    }
    else {
        Py_XDECREF(mod);
        mod = import_find_and_load(abs_name); //调用import_find_and_load
        if (mod == NULL) {
            goto error;
        }
    }

    /*...*/
    return final_mod;
}

static PyObject *
import_find_and_load(PyObject *abs_name)
{
   /*..*/

    if (PyDTrace_IMPORT_FIND_LOAD_START_ENABLED())
        PyDTrace_IMPORT_FIND_LOAD_START(PyUnicode_AsUTF8(abs_name));

    mod = _PyObject_CallMethodIdObjArgs(interp->importlib, //调用_PyObject_CallMethodIdObjArgs
                                        &PyId__find_and_load, abs_name,
                                        interp->import_func, NULL);

    /*...*/

    return mod;
}

PyObject *
_PyObject_CallMethodIdObjArgs(PyObject *obj,
                              struct _Py_Identifier *name, ...)
{
    /*...*/

    callable = _PyObject_GetAttrId(obj, name);
    if (callable == NULL) {
        return NULL;
    }

    va_start(vargs, name);
    result = object_vacall(callable, vargs); //调用interp->importlib的_find_and_load方法
    va_end(vargs);

    Py_DECREF(callable);
    return result;
}
~~~

通过上述追溯我们可以发现，在import一个模块时，最终调用的是interp->importlib这个模块的_find_and_load方法。

~~~C
static _PyInitError
initimport(PyInterpreterState *interp, PyObject *sysmod)
{
    PyObject *importlib;
    PyObject *impmod;
    PyObject *value;

    /* Import _importlib through its frozen version, _frozen_importlib. */
    /*...*/
    importlib = PyImport_AddModule("_frozen_importlib");
    if (importlib == NULL) {
        return _Py_INIT_ERR("couldn't get _frozen_importlib from sys.modules");
    }
    interp->importlib = importlib;
    Py_INCREF(interp->importlib);
    
    interp->import_func = PyDict_GetItemString(interp->builtins, "__import__");//interp->import_func被初始化为builtin___import__
    if (interp->import_func == NULL)
        return _Py_INIT_ERR("__import__ not found");
    Py_INCREF(interp->import_func);

    /*...*/

    return _Py_INIT_OK();
}
~~~

全局搜索_frozen_importlib，可以发现一个二进制文件——importlib.h：

~~~C
/* Auto-generated by Programs/_freeze_importlib.c */
const unsigned char _Py_M__importlib_bootstrap[] = {
    99,0,0,0,0,0,0,0,0,0,0,0,0,4,0,0,
    0,64,0,0,0,115,194,1,0,0,100,0,90,0,100,1,
    97,1,100,2,100,3,132,0,90,2,100,4,100,5,132,0,
    /*...*/
~~~

根据注释可知，这个文件是由_freeze_importlib.c自动生成的，\_freeze_importlib.c会把Lib/importlib/\_bootstrap.py中的python代码转换为二进制存储到Python/importlib.h中。从这里我们可以发现，python中的import流程大致上是用python实现的，interp->importlib的\_find_and_load方法可以映射到Lib/importlib/\_bootstrap.py中的\_find_and_load这个python函数。

```python
def _find_and_load(name, import_): #根据之前的分析可知，name为abs_name，import_为interp->import_func即builtin___import__
    """Find and load the module."""
    with _ModuleLockManager(name):
        module = sys.modules.get(name, _NEEDS_LOADING) #在缓存sys.modules中查找
        if module is _NEEDS_LOADING: #没找到则调用_find_and_load_unlocked
            return _find_and_load_unlocked(name, import_) #调用

    if module is None:
        message = ('import of {} halted; '
                   'None in sys.modules'.format(name))
        raise ModuleNotFoundError(message, name=name)

    _lock_unlock_module(name)
    return module
```

可以看到，python首先在缓存中查找要加载的module，找不到的话则调用\_find\_and\_load\_unlocked

```python
def _find_and_load_unlocked(name, import_):
    path = None
    parent = name.rpartition('.')[0] #a.b.c分解为（"a.b",".","c"）
    if parent: #搜索parent module
        if parent not in sys.modules: #缓存中不存在的话，则继续加载parent
            _call_with_frames_removed(import_, parent) #这里的import_就是builtin___import__，可以看出，python的module加载过程是一个递归的过程
        # Crazy side-effects!
        if name in sys.modules:
            return sys.modules[name]
        parent_module = sys.modules[parent]
        try:
            path = parent_module.__path__
        except AttributeError:
            msg = (_ERR_MSG + '; {!r} is not a package').format(name, parent)
            raise ModuleNotFoundError(msg, name=name) from None
    spec = _find_spec(name, path) #创建ModuleSpec对象
    if spec is None:
        raise ModuleNotFoundError(_ERR_MSG.format(name), name=name)
    else:
        module = _load_unlocked(spec) #从ModuleSpec对象创建module
    if parent:
        # Set the module as an attribute on its parent.
        parent_module = sys.modules[parent]
        setattr(parent_module, name.rpartition('.')[2], module) #将当前module设置为其父module的属性
    return module
```

从上面过程可以看出，python创建module是一个递归过程，首先从最低级的module开始，如果父module已经创建则直接创建子module，否则自下而上递归进行创建工作，直至所有module创建完成。
