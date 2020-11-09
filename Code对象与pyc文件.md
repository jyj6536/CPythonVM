# Code对象与pyc文件

Python虽然是一种解释性的语言，其执行过程依然要有编译的过程．.py文件在执行过程中首先被解释器编译为pyc文件，然后装载到内存中开始执行．在pyc文件中存放的是PyCodeObject对象，该对象是CPython内部使用的，用来表示一段可执行的Python代码段的对象，比如函数，模块，一个类或者迭代器．PyCodeObject对象的内存布局如下图所示．

![image-20201109222814061](Code对象与pyc文件.assets/image-20201109222814061.png)