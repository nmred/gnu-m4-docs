# 调用 m4

m4 命令格式如下：

>**$ m4 [option...] [file...]**

所有的命令参数都是以 `-`  开头，如果是长命令参数则以 `--` 开头。对于长参数不需要写完成,只要有一个明确的前缀就足够了。POSIX 要求 m4 可以支持参数和文件混合指定，即使设置了 `POSIXLY_CORRECT`<sup>2.1</sup> 环境变量。大部分参数生效的效果和指定的位置没有关系，但是有一些则会有影响，如符号`--` 表示参数选项结束, 该符号前面指定参数，后面指定输入文件

>2.1: 在一些情况下，GNU utility 的默认行为和 POSIX 标准不兼容。为了解决这种不兼容情况，Linux 系统引入了环境变量’POSIXLY_CORRECT’。当设置了该环境变量时，GNU utilities 完全遵从 POSIX 标准。

对于短命令参数选项，不带参数的选项可以合并成一个单一的命令参数，有的参数选项强制参数可能需要提供过一个或两个参数，有的可选参数选项必须提供一个参数。换句话说，`m4 - QPDfoo -d a -df` 和 ` m4 -Q -P -D foo -d -df -- ./a` 是等价的，只是后者的使用方式更加普遍

对于长命令参数选项，对于强制参数的选项必须提供一个或两个参数使用等号连接，对于可选参数选项必须提供一个参数。换句话说， `m4 --def foo --debug a` 和 `m4 --define=foo --debug= -- ./a` 是等价的，只是后者使用方式更加普及。

下面将通过功能划分来理解参数选项

### 操作模式参数选项

如下参数选项将从总体上控制 m4 的执行：

--help 在标准输出中打印帮助信息，并且退出


--version 打印程序的版本号到标准输出，然后立即退出，不会读取任何输入文件也不会执行任何动作


-E

--fatal-warnings 控制 warning 错误对程序的影响。如果没有指定，当有 warning 错误输出时程序会继续执行并且退出状态也不会受到影响。如果确切的指定一次，warning 错误变成 fatal 错误，当有问题发生时，程序会继续执行，只是退出状态变成非零。如果指定多次，则在第一次出现退出状态为非零的时候程序会退出。对于分等级的行为配置在 m4 1.4.9 版本后生效，对于以前的版本应该通过指定 `-E` 两次实现

-i

--interactive

-e m4 通过交互模式调用。意思是所有的输出将直接输出，并且忽略所有的中断暂停。选项 `-e` 是为了兼容其他 m4 实现的，使用该选项将会受到 warning 错误提示，因为在后续 GNU m4 版本中将废弃该选项

-P

--prefix-builtins 将所有的内建宏的名称前加前缀 `m4_` 。例如，使用该参数应该用 `m4_define` 代替 `define`，`m4___file__` 代替 `__file__` 。该参数在指定 `_R`  参数也指定时失效

-Q

--quiet

--silent 屏蔽 warning 错误，例如一些错误、调用宏是多余参数或者处理空字符串转化为零的操作等。


--warn-macro-sequence[=regexp] 如果正则表达式匹配任意宏定义匹配结果是非空将报 warning 错误 (define 或者 pushdef).如果匹配结果为空则忽略；然而如果正则表达式是空字符串则关闭任何错误。如果是可选的正则表达式没有提供默认正则表达式为: `\$\({[^}]*}\|[0-9][0-9]+\)` (字符 $ 后面紧随多个数字或者左括号)，通过这些特性在 GNU m4 2.0 中将改变一些语义（其中一个变化就是如何在宏定义中, 具体参考 [参数]()）。通过自定义正则表达式可以自定义反向检索一些特性和宏定义

-W regexp 

--word-regexp=regexp 使用正则表达式改变宏的语法。这个试验性的选项参数并不是所有的 GNU m4 都实现了，参考[Change word]()

### 预处理特性选项参数

如下选项参数可以使得 m4 更加像一个预处理器. 可以通过命令行控制定义宏、删除宏定义、改变搜索路劲、通过输出跟踪输入来源等特性

-D name[=value] 

--define=name[=value] 将宏定义加入到符号表中。如果 `=value` 是错误的，则该值会置为空字符串。`value` 值可以是任意字符串，并且可以定义带参数的宏，和在输入文件定义类似。该选项参数可以指定多次,关于文件名的顺序是重要的，并且可以重复定义替换以前的宏定义

-I directory 

--include=directory 

提供 m4 在当前工作目录没找到包含文件时搜索的目录路劲。更多信息参考 [搜索路劲]() ,该参数可以指定多次

-s

--synclines

生成对应的行号，这个对于 C 预处理器或者其他类似的工具很有用。该选项参数对于保证文件名的顺序是很重要。例如当 m4 用于编译器前端是，源文件的名称和行号信息将通过 `#line linenum "file"` 形式的指令表示，最终插入到输出的中间文件中。这些指令可以跟踪输入文件内容的来源及其行号。`"file"` 部分经常可以省略在上一个指令文件名没有变化的情况下。

同步指令一直会通过自身提供完整的行号。当同步指令在输出中间文件中不同步时，关联的同步指令会延迟执行直到在不包含引号的字符串或者注释的新一行会打印出来, 例如：

>**define(twoline, ‘1 **

>**2’)**

>**=>#line 2 "stdin"**

>**changecom(‘/\*’, ‘\*/’)**

>**=>**

>**define(comment, ‘/\*1 **

>**2\*/’)**

>**=>#line 5**

>**=>**

>**dnl no line **

>**hello**

>**=>**

>**=>#line 7**

>**=>hello**

>**twoline**

>**=>1**

>**=>#line 8**

>**=>2**

>**comment**

>**=>/\*1**

>**=>2\*/**

>**one comment ‘two **

>**three’**

>**=>#line 10**

>**=>one /\*1**

>**=>2\*/ two**

>**=>three**

>**goodbye**

>**=>#line 12**

>**=>goodbye**

-U name 

--undefine=name 删除一个已经定义的宏。显然只有已经定义的宏才可以通过该选项参数删除。该选项参数可以多次指定；如果删除一个没有定义的宏将忽略。指定是文件名称的顺序很重要


### 限制控制选项参数

m4 的一些限制可以修改。为了兼容性，m4 也接收一些控制限制的选项参数在其他实现中，但是对于 GNU m4 是默认情况下是无限制的（限制可能来源于你的机器的硬件和操作系统）

-g

--gnu 是否开启所有实现扩展。在 m4 发布版中，该参数默认是开启的；这个目前仅适用于在 `--traditional` 之前使用。由于一些客户端程序必须使用非 POSIX 的命令行强制所有的 POSIX 行为，GNU 一些行为特性不太可能写出来严格符合 POSIX 标准的客户端程序，为了避免 GNU m4 扩展的不兼容。因此，未来版本中通过隐式使用选项参数 `--traditional` 在设置环境变量 `POSIXLY\_CORRECT` 时， 这样在设置 `POSIXLY\_CORRECT` 环境变量以及升级最新版 m4 后就不会无故的发生程序错误。如果项目中使用 GNU 扩展可以认为是使用了 `--gnu` 选项

-G

--traditional

相对于 System V 系统，屏蔽在该实现中所有的扩展，具体参见 [兼容性]()

-H num 

--hashsize=num

限制内部符号表的大小。为了更好的性能，该数应该是一个质数，但是并不去检查, 默认是 509。除非你定义的宏的个数过高，一般情况下不需要修改该值。

-L num 

--nesting-limit=num

人为的限制嵌套宏调用层级，如果嵌套层数超过了限制大小则停止程序的执行，当不指定的时候嵌套层级默认没有限制在平台可以发现堆栈溢出的时候，否则是最多 1024 层。该值为 0 表示不限制；但是严重的嵌套是堆栈溢出的潜在原因。

该选项参数影响会与文本的嵌套有关而不是动态递归。该选项在处理复杂的 m4 输入时会很有用，对于诊断递归算法作用不大。许多用户从来不会修改该参数。

该选项对于无限循环扫描的情况不会生效，因为这种情况下不一定会耗费大量的内存或者堆栈空间。可以巧妙的使用循环扫描通过 m4 解决复杂的、耗时的计算最终得到结果。使用这个限制选项在这个场景下 m4 很难控制. 这块有许多有问题的案例： `define('a', 'a')a` 仅仅是一个最简单的例子(参考 [兼容性]()) , 期望 GNU m4 可以发现无线循环和期望一个编译器发现与诊断无线循环类似：通常这是个比较难的问题，无法鉴定

-B num 

-S num

-T num

该参数选项是为了兼容 System V m4 而存在的，对于本版本实现不做任何事情，在未来版本中可能会去掉

-N num 

--diversions=num

该选项参数仅仅是为了兼容老版本的 m4, 并且控制在同一时间的转向次数。在这里没有不起任何作用，在随后的版本中可能会去掉该参数

### 转存状态选项参数

GNU m4 配备了转出
GNU m4 comes with a feature of freezing internal state (see Chapter 15 [Frozen  les], page 105). This can be used to speed up m4 execution when reusing a common initialization script.

-F file --freeze-state=file

Once execution is  nished, write out the frozen state on the speci ed  le. It is conventional, but not required, for  le to end in ‘.m4f’.

-R file --reload-state=file

Before execution starts, recover the internal state from the speci ed frozen  le. The options -D, -U, and -t take e ect after state is reloaded, but before the input  les are read.

### 调试选项参数

Finally, there are several options for aiding in debugging m4 scripts.

-d[flags] --debug[=flags]

Set the debug-level according to the  ags  ags. The debug-level controls the format and amount of information presented by the debugging functions. See Section 7.3 [Debug Levels], page 58, for more details on the format and meaning of  ags. If omitted,  ags defaults to ‘aeq’.

--debugfile[=file] -o file --error-output=file

Redirect dumpdef output, debug messages, and trace output to the named  le. Warnings, error messages, and errprint output are still printed to standard error. If these options are not used, or if  le is unspeci ed (only possible for --debugfile), debug output goes to standard error; if  le is the empty string, debug output is discarded. See Section 7.4 [Debug Output], page 60, for more details. The option --debugfile may be given more than once, and order is signi cant with respect to  le names. The spellings -o and --error-output are misleading and inconsistent with other GNU tools; for now they are silently accepted as synonyms of --debugfile and only recognized once, but in a future version of M4, using them will cause a warning to be issued.

-l num --arglength=num

Restrict the size of the output generated by macro tracing to num characters per trace line. If unspeci ed or zero, output is unlimited. See Section 7.3 [Debug Levels], page 58, for more details.

-t name --trace=name

This enables tracing for the macro name, at any point where it is de ned. name need not be de ned when this option is given. This option may be given more than once, and order is signi cant with respect to  le names. See Section 7.2 [Trace], page 55, for more details.

### 指定输入文件命令行

The remaining arguments on the command line are taken to be input  le names. If no names are present, standard input is read. A  le name of - is taken to mean standard input. It is conventional, but not required, for input  les to end in ‘.m4’.

The input  les are read in the sequence given. Standard input can be read more than once, so the  le name - may appear multiple times on the command line; this makes a di erence when input is from a terminal or other special  le type. It is an error if an input  le ends in the middle of argument collection, a comment, or a quoted string.

The options --define (-D), --undefine (-U), --synclines (-s), and --trace (-t) only take e ect after processing input from any  le names that occur earlier on the command line. For example, assume the  le foo contains:

$ cat foo bar

The text ‘bar’ can then be rede ned over multiple uses of foo:

$ m4 -Dbar=hello foo -Dbar=world foo ⇒hello
⇒world

If none of the input  les invoked m4exit (see Section 14.3 [M4exit], page 103), the exit status of m4 will be 0 for success, 1 for general failure (such as problems with reading an input  le), and 63 for version mismatch (see Section 15.1 [Using frozen  les], page 105).

If you need to read a  le whose name starts with a -, you can specify it as ‘./-file’, or use -- to mark the end of options.
