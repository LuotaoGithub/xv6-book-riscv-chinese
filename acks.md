# 前言和致谢

这是一份针对操作系统课程的草稿文本，通过研究一个名为xv6的示例内核，解释操作系统的主要概念。xv6的模型基于Dennis Ritchie和Ken Thompson的Unix Version 6（v6）。xv6大致遵循v6的结构和风格，但是用ANSI C语言实现，用于多核RISC-V处理器架构。

阅读本文应该与xv6的源代码一起进行，这种方法的灵感来源于约翰·莱恩斯（John Lions）的《UNIX第六版注释》（Commentary on UNIX 6th Edition）。请访问[https://pdos.csail.mit.edu/6.1810](https://pdos.csail.mit.edu/6.1810)获取有关v6和xv6的在线资源的指向，包括使用xv6进行的多个实验任务。

我们将此文本用于MIT的操作系统课程6.828和6.1810中。我们要感谢那些直接或间接为xv6做出贡献的教师、助理教师和学生们。特别地，我们要感谢Adam Belay、Austin Clements和Nickolai Zeldovich。最后，我们要感谢那些向我们报告文本错误或提出改进建议的人：Abutalib Aghayev、Sebastian Boehm、brandb97、Anton Burtsev、Raphael Carvalho、Tej Chajed、Rasit Eskicioglu、Color Fuzzy、Wojciech Gac、Giuseppe、Tao Guo、Haibo Hao、Naoki Hayama、Chris Henderson、Robert Hilderman、Eden Hochbaum、Wolfgang Keller、Henry Laih、Jin Li、Austin Liew、Pavan Maddamsetti、Jacek Masiulaniec、Michael McConville、m3hm00d、miguelgvieira、Mark Morrissey、Muhammed Mourad、Harry Pan、Harry Porter、Siyuan Qian、 Askar Safin、Salman Shah、Huang Sha、Vikram Shenoy、Adeodato Simó、Ruslan Savchenko、Pawel Szczurko、Warren Toomey、tyfkda、tzerbib、Vanush Vaswani、Xi Wang、Zou Chang Wei、Sam Whitlock、LucyShawYang和Meng Zhou。

如果您发现错误或有改进建议，请发送电子邮件至Frans Kaashoek和Robert Morris（kaashoek, rtm@csail.mit.edu）。