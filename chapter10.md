# Chapter 10

# 总结

本文通过逐行阅读一个操作系统xv6，介绍了操作系统的主要思想。一些代码行体现了主要思想的本质（例如，上下文切换，用户/内核边界，锁等），每行代码都很重要。其他代码行提供了实现特定操作系统思想的示例，并可以用不同的方式轻松完成（例如，更好的调度算法，更好的磁盘数据结构来表示文件，更好的日志记录以允许并发事务等）。所有这些思想都是在一个非常成功的系统调用接口Unix接口的背景下阐述的，但是这些思想也适用于其他操作系统的设计。