# 宏调用

This chapter covers macro invocation, macro arguments and how macro expansion is treated.

### 调用宏

Macro invocations has one of the forms
name
which is a macro invocation without any arguments, or
name(arg1, arg2, ..., argn)
which is a macro invocation with n arguments. Macros can have any number of arguments. All arguments are strings, but di↵erent macros might interpret the arguments in di↵erent ways.
The opening parenthesis must follow the name directly, with no spaces in between. If it does not, the macro is called with no arguments at all.
For a macro call to have no arguments, the parentheses must be left out. The macro call
name()
is a macro call with one argument, which is the empty string, not a call with no arguments.

### 阻止宏调用

An innovation of the m4 language, compared to some of its predecessors (like Strachey’s GPM, for example), is the ability to recognize macro calls without resorting to any special, prefixed invocation character. While generally useful, this feature might sometimes be the source of spurious, unwanted macro calls. So, GNU m4 o↵ers several mechanisms or techniques for inhibiting the recognition of names as macro calls.
First of all, many builtin macros cannot meaningfully be called without arguments. As a GNU extension, for any of these macros, whenever an opening parenthesis does not immediately follow their name, the builtin macro call is not triggered. This solves the most usual cases, like for ‘include’ or ‘eval’. Later in this document, the sentence “This macro is recognized only with parameters” refers to this specific provision of GNU M4, also known as a blind builtin macro. For the builtins defined by POSIX that bear this disclaimer, POSIX specifically states that invoking those builtins without arguments is unspecified, because many other implementations simply invoke the builtin as though it were given one empty argument instead.
$ m4 eval )eval eval(‘1’) )1
There is also a command line option (--prefix-builtins, or -P, see Section 2.1 [In- voking m4], page 7) that renames all builtin macros with a prefix of ‘m4_’ at startup. The option has no e↵ect whatsoever on user defined macros. For example, with this option, one has to write m4_dnl and even m4_m4exit. It also has no e↵ect on whether a macro requires parameters.
$ m4 -P eval
)eval eval(‘1’) )eval(1) m4_eval )m4_eval m4_eval(‘1’) )1
Another alternative is to redefine problematic macros to a name less likely to cause conflicts, using Chapter 5 [Definitions], page 25.
If your version of GNU m4 has the changeword feature compiled in, it o↵ers far more flex- ibility in specifying the syntax of macro names, both builtin or user-defined. See Section 8.4 [Changeword], page 67, for more information on this experimental feature.
Of course, the simplest way to prevent a name from being interpreted as a call to an existing macro is to quote it. The remainder of this section studies a little more deeply how quoting a↵ects macro invocation, and how quoting can be used to inhibit macro invocation.
Even if quoting is usually done over the whole macro name, it can also be done over only a few characters of this name (provided, of course, that the unquoted portions are not also a macro). It is also possible to quote the empty string, but this works only inside the name. For example:
‘divert’ )divert ‘d’ivert )divert di‘ver’t )divert div‘’ert )divert
all yield the string ‘divert’. While in both:
‘’divert
)
divert‘’
)
the divert builtin macro will be called, which expands to the empty string.
The output of macro evaluations is always rescanned. In the following example, the input ‘x‘’y’ yields the string ‘bCD’, exactly as if m4 has been given ‘substr(ab‘’cde, ‘1’, ‘3’)’ as input:
     define(‘cde’, ‘CDE’)
	 )
	      define(‘x’, ‘substr(ab’)
		  )
     define(‘y’, ‘cde, ‘1’, ‘3’)’)
	 )
	 x‘’y )bCD
	 Unquoted strings on either side of a quoted string are subject to being recognized as macro names. In the following example, quoting the empty string allows for the second macro to be recognized as such:
	      define(‘macro’, ‘m’)
		  )
		  macro(‘m’)macro )mmacro macro(‘m’)‘’macro )mm
		  Quoting may prevent recognizing as a macro name the concatenation of a macro expan- sion with the surrounding characters. In this example:
		       define(‘macro’, ‘di$1’)
			   )
			   macro(‘v’)‘ert’ )divert macro(‘v’)ert )
			   the input will produce the string ‘divert’. When the quotes were removed, the divert builtin was called instead.


### 宏参数

When a name is seen, and it has a macro definition, it will be expanded as a macro.
If the name is followed by an opening parenthesis, the arguments will be collected before the macro is called. If too few arguments are supplied, the missing arguments are taken to be the empty string. However, some builtins are documented to behave di↵erently for a missing optional argument than for an explicit empty string. If there are too many arguments, the excess arguments are ignored. Unquoted leading whitespace is stripped o↵ all arguments, but whitespace generated by a macro expansion or occurring after a macro that expanded to an empty string remains intact. Whitespace includes space, tab, newline, carriage return, vertical tab, and formfeed.
     define(‘macro’, ‘$1’)
	 )
	 macro( unquoted leading space lost) )unquoted leading space lost macro(‘ quoted leading space kept’) ) quoted leading space kept macro(
	 divert ‘unquoted space kept after expansion’) ) unquoted space kept after expansion macro(macro(‘
	 ’)‘whitespace from expansion kept’)

)whitespace from expansion kept macro(‘unquoted trailing whitespace kept’ )
)unquoted trailing whitespace kept
)
Normally m4 will issue warnings if a builtin macro is called with an inappropriate number of arguments, but it can be suppressed with the --quiet command line option (or --silent, or -Q, see Section 2.1 [Invoking m4], page 7). For user defined macros, there is no check of the number of arguments given.
$ m4 index(‘abc’)
m4:stdin:1: Warning: too few arguments to builtin ‘index’ )0
index(‘abc’,)
)0
index(‘abc’, ‘b’, ‘ignored’)
m4:stdin:3: Warning: excess arguments to builtin ‘index’ ignored )1
$ m4 -Q
index(‘abc’)
)0
index(‘abc’,)
)0
index(‘abc’, ‘b’, ‘ignored’) )1
Macros are expanded normally during argument collection, and whatever commas, quotes and parentheses that might show up in the resulting expanded text will serve to define the arguments as well. Thus, if foo expands to ‘, b, c’, the macro call
bar(a foo, d)
is a macro call with four arguments, which are ‘a ’, ‘b’, ‘c’ and ‘d’. To understand why the first argument contains whitespace, remember that unquoted leading whitespace is never part of an argument, but trailing whitespace always is.
It is possible for a macro’s definition to change during argument collection, in which case the expansion uses the definition that was in e↵ect at the time the opening ‘(’ was seen.
     define(‘f’, ‘1’)
	 )
f(define(‘f’, ‘2’)) )1
f
)2
It is an error if the end of file occurs while collecting arguments.
hello world )hello world define(

^D


4.4 宏参数包装

Each argument has unquoted leading whitespace removed. Within each argument, all un- quoted parentheses must match. For example, if foo is a macro,
     foo(() (‘(’) ‘(’)
	 is a macro call, with one argument, whose value is ‘() (() (’. Commas separate arguments, except when they occur inside quotes, comments, or unquoted parentheses. See Section 5.3 [Pseudo Arguments], page 27, for examples.
	 It is common practice to quote all arguments to macros, unless you are sure you want the arguments expanded. Thus, in the above example with the parentheses, the ‘right’ way to do it is like this:
	      foo(‘() (() (’)
		  It is, however, in certain cases necessary (because nested expansion must occur to create the arguments for the outer macro) or convenient (because it uses fewer characters) to leave out quotes for some arguments, and there is nothing wrong in doing it. It just makes life a bit harder, if you are not careful to follow a consistent quoting style. For consistency, this manual follows the rule of thumb that each layer of parentheses introduces another layer of single quoting, except when showing the consequences of quoting rules. This is done even when the quoted string cannot be a macro, such as with integers when you have not changed the syntax via changeword (see Section 8.4 [Changeword], page 67).
		  The quoting rule of thumb of one level of quoting per parentheses has a nice property: when a macro name appears inside parentheses, you can determine when it will be expanded. If it is not quoted, it will be expanded prior to the outer macro, so that its expansion becomes the argument. If it is single-quoted, it will be expanded after the outer macro. And if it is double-quoted, it will be used as literal text instead of a macro name.
		       define(‘active’, ‘ACT, IVE’)
			   )
		       define(‘show’, ‘$1 $1’)
			   )
		  show(active)
		  )ACT ACT show(‘active’) )ACT, IVE ACT, IVE show(‘‘active’’) )active active


### 宏展开

When the arguments, if any, to a macro call have been collected, the macro is expanded, and the expansion text is pushed back onto the input (unquoted), and reread. The expansion text from one macro call might therefore result in more macros being called, if the calls are included, completely or partially, in the first macro calls’ expansion.

Taking a very simple example, if foo expands to ‘bar’, and bar expands to ‘Hello’, the input
$ m4 -Dbar=Hello -Dfoo=bar foo
)Hello
will expand first to ‘bar’, and when this is reread and expanded, into ‘Hello’.
