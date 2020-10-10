# 2 使用GDB查看core文件

默认编译出来的程序在出现Segmentation fault 时并没有生成core崩溃文件，可以在gcc/g++编译时增加-g选项。

如果仍然没有生成core文件，则可能是因为系统设置了core文件大小为0，可以通过：`ulimit -a` 查询得知。

执行 `ulimit -c unlimited` 命令后可以使core文件大小不受限制。此时再次运行程序应该就能在同级目录看到`core.XXX`文件了

使用 `gdb ./a.out core.XXX` 可以查看出错所在行信息，这样就进入了 gdb core 调试模式。

追踪产生segmenttation fault的位置及代码函数调用情况：

`gdb>bt`

这样，一般就可以看到出错的代码是哪一句了，还可以打印出相应变量的数值，进行进一步分析。

[返回目录](https://www.cnblogs.com/kuliuheng/p/11698378.html#_labelTop)

# **3 使用GDB调试程序**

如上述流程不能解决问题，下面可使用gdb单步调试程序。重新编译程序，编译命令中加入-g。如：

gcc -lm -O3 -g file.c -o file
之后使用gdb命令

`gdb file`
开始调试。

输入`start`使程序运行到`main`中第一行运行代码。`next`或者n为执行下一行程序，`until xx`执行到xx行，print或p可输出变量值，`b xx`用于在xx行设置断点，`run`或`r`用于执行程序至下一断点，`d xx`删除xx行断点。

我们可以先run一遍程序，这时它会提示出错行信息。然后`until`到出错行前5行，交替执行`next`和`print`，输出与出错行变量相关变量或指针的值。最终定位出错的根本操作在哪一行。修改之即可。