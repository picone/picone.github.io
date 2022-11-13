---
layout: post
title: GDB core dump 导出 std::string 的 binary 内容到文件。
---

线上有个服务core dump，首先使用gdb打开core文件

```shell
gdb bin/xx xx.xx
```

然后使用 backtrace(bt) 查看 core 对应的栈，并使用 frame(f) 切换到我们业务的层级。

由于 core 的位置是 opencv，调用 cv::mean 时候 core 了，我们需要把原始的图片 dump 出来分析。

业务中有个 `image_data` 局部变量保存了入参的图片，下面我们尝试 dump 出来。

```shell
(gdb) p image_data
$3 = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>},
    _M_p = 0x7f06d5428c58 "\377\330\377\340"}}
```

可以看到 `image_data` 是 `std::string`，里面有层 `_M_dataplus._M_p` 只向了原始的字符串指针。我们需要做的是指定这个字符串长度。 使用网上的
一些方法都不太管用，我们来用自己的方法分析。我使用的是 gcc8 编译，关于 `std::string` 可以看看腾讯的[这个文章](https://zhuanlan.zhihu.com/p/157169295)
写的比较清楚，gcc 的 string 对象内存布局如下：

![gcc string 内存布局]({{ "/assets/images/2021-07-31-gdb-std-string-memory.jpg" | relative_url }})

所以，我们只需要获取 `_M_dataplus._M_p` 的前一个 `_Rep` 地址就可以了。

```shell
(gdb) p ((std::string::_Rep*)image_data)[-1]
$4 = {<std::basic_string<char, std::char_traits<char>, std::allocator<char> >::_Rep_base> = {_M_length = 30752, _M_capacity = 32711, _M_refcount = -1}, 
  static _S_max_size = 4611686018427387897, static _S_terminal = 0 '\000', static _S_empty_rep_storage = <optimized out>}
```

可以看到，字符串长度 30752，分配大小 32711，没有被引用。

接着，我们知道 `_M_p` 的地址是 `0x7f06d5428c58`，因此可以用 gdb 的 dump 命令把内存数据保存到文件。但是这个命令需要知道内存的起止，我们现在
知道了起始位置和长度，需要计算止地址，这个好办。

打开苹果的编程型计算器，输入起始地址

![计算步骤 1]({{ "/assets/images/2021-07-31-calculator-step-1.png" | relative_url }})

然后转换成十进制

![计算步骤 2]({{ "/assets/images/2021-07-31-calculator-step-2.png" | relative_url }})

然后加上长度

![计算步骤 3]({{ "/assets/images/2021-07-31-calculator-step-3.png" | relative_url }})

然后再转换成十六进制就是我们要的地址了。

![计算步骤 4]({{ "/assets/images/2021-07-31-calculator-step-4.png" | relative_url }})

然后执行命令

```shell
(gdb) dump binary memory /tmp/test.jpg 0x7f06d5428c58 0x7F06D5430478
```

然后看看 /tmp/test.jpg 吧~
