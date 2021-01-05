# nohup的使用
在某些时候，我们一般想让某个程序在后台运行，于是我们将常会用 & 在程序结尾来让程序自动运行。比如我们要运行mysql在后台： /usr/local/mysql/bin/mysqld_safe –user=mysql &。可是有很多程序并不想mysqld一样，这时就有了nohup发挥的余地

### nohup 介绍：
用于在系统后台不挂断的运行命令，退出终端不会影响程序的运行；
默认会输出一个nohup.out的文件到当前目录下，如果当前目录的nohuo.out文件不可写，输出重定向到 $HOME/nohup.out 文件中

### 语法格式：
```
nohup Command [Args] &
```
说明：
- Command: 要执行的命令
- Args：参数
- &：命令后台运行

### 退出状态：该命令返回下列出口值：
- 126 可以查找但不能调用 Command 参数指定的命令。
- 127 nohup 命令发生错误或不能查找由 Command 参数指定的命令。
否则，nohup 命令的退出状态是 Command 参数指定命令的退出状态。

### 指定输出到指定文件
```
nohup command >> root.log 2>&1 &
```
**2>&1** 解释：
将标准错误2重定向到标准输出 &1， 标准输出 &1 再被重定向追加到 root.log 中。
- 0：stdin(standard input, 标准输入)
- 1：stdout(standard output, 标准输出)
- 2：stderr(standard error, 标准错误输出)

### 杀死后台进程
使用以下命令查找nohup运行脚本的PID，然后kill
```
ps -aux | grep "Command"
```