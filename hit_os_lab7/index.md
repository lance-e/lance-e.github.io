# Hit_os_lab7


<!--more-->

# hit os lab7

### 1.实验内容

本实验的基本内容是修改 Linux 0.11 的终端设备处理代码，对键盘输入和字符显示进行非常规的控制。

在初始状态，一切如常。用户按一次 F12 后，把应用程序向终端输出所有字母都替换为“*”。用户再按一次 F12，又恢复正常。第三次按 F12，再进行输出替换。依此类推。

以 ls 命令为例：

正常情况：

```bash
# ls
hello.c hello.o hello
```

第一次按 F12，然后输入 ls：

```bash
# **
*****.* *****.* *****
```

第二次按 F12，然后输入 ls：

```bash
# ls
hello.c hello.o hello
```

第三次按 F12，然后输入 ls：

```bash
# **
*****.* *****.* *****
```

### 2.实验开始

找到F12对应的例程：

~~~assembly
...
key_table:
	.long none,do_self,do_self,do_self	/* 00-03 s0 esc 1 2 */
	.long do_self,do_self,do_self,do_self	/* 04-07 3 4 5 6 */
	.long do_self,do_self,do_self,do_self	/* 08-0B 7 8 9 0 */
	.long do_self,do_self,do_self,do_self	/* 0C-0F + ' bs tab */
	.long do_self,do_self,do_self,do_self	/* 10-13 q w e r */
	.long do_self,do_self,do_self,do_self	/* 14-17 t y u i */
	.long do_self,do_self,do_self,do_self	/* 18-1B o p } ^ */
	.long do_self,ctrl,do_self,do_self	/* 1C-1F enter ctrl a s */
	.long do_self,do_self,do_self,do_self	/* 20-23 d f g h */
	.long do_self,do_self,do_self,do_self	/* 24-27 j k l | */
	.long do_self,do_self,lshift,do_self	/* 28-2B { para lshift , */
	.long do_self,do_self,do_self,do_self	/* 2C-2F z x c v */
	.long do_self,do_self,do_self,do_self	/* 30-33 b n m , */
	.long do_self,minus,rshift,do_self	/* 34-37 . - rshift * */
	.long alt,do_self,caps,func		/* 38-3B alt sp caps f1 */
	.long func,func,func,func		/* 3C-3F f2 f3 f4 f5 */
	.long func,func,func,func		/* 40-43 f6 f7 f8 f9 */
	.long func,num,scroll,cursor		/* 44-47 f10 num scr home */
	.long cursor,cursor,do_self,cursor	/* 48-4B up pgup - left */
	.long cursor,cursor,do_self,cursor	/* 4C-4F n5 right + end */
	.long cursor,cursor,cursor,cursor	/* 50-53 dn pgdn ins del */
	.long none,none,do_self,func		/* 54-57 sysreq ? < f11 */
	.long func,none,none,none		/* 58-5B f12 ? ? ? */           /*  <---- 在这个注释我们就能找到F12对应的例程*/
	.long none,none,none,none		/* 5C-5F ? ? ? ? */
...
~~~

我们单独写一段F12的例程，并修改对应位置

~~~assembly
newfunc:
	pushl	%eax
	pushl %ecx
	pushl %edx
	call 	f12func
	popl 	%edx
	popl 	%ecx
	popl 	%eax
	ret
~~~

在console.c中添加一个flag全局变量，以及修改flag的函数,同时修改con_write函数

~~~c
int flag = 0;
void f12func()
{
	flag ^= 1;
  return ;
}


...
void con_write(struct tty_struct * tty)
{
	int nr;
	char c;

	nr = CHARS(tty->write_q);
	while (nr--) {
		GETCH(tty->write_q,c);
		switch(state) {
			case 0:
				if (c>31 && c<127) {
        	//这里作出更改
          if (flag){
            c = '*';
          }
					if (x>=video_num_columns) {
						x -= video_num_columns;
						pos -= video_size_row;
						lf();
					}
	...
~~~

在linux/tty.h中添加f12func函数的声明：

~~~c
void f12func();
~~~

结束

### 3.回答问题

完成实验后，在实验报告中回答如下问题：

- 在原始代码中，按下 F12，中断响应后，中断服务程序会调用 func？它实现的是什么功能？
  - 原始代码中，会调用func，将功能字符转换为特殊字符
- 在你的实现中，是否把向文件输出的字符也过滤了？如果是，那么怎么能只过滤向终端输出的字符？如果不是，那么怎么能把向文件输出的字符也一并进行过滤？
  - 不是，修改file_write而不是con_write

### 4.总结

最简单的一集😋

