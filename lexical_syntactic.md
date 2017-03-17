# 语法与词法

As m4 reads its input, it separates it into tokens. A token is either a name, a quoted string, or any single character, that is not a part of either a name or a string. Input to m4 can also contain comments. GNU m4 does not yet understand multibyte locales; all operations are byte-oriented rather than character-oriented (although if your locale uses a single byte encoding, such as ISO-8859-1, you will not notice a di↵erence). However, m4 is eight-bit clean, so you can use non-ascii characters in quoted strings (see Section 8.2 [Changequote], page 62), comments (see Section 8.3 [Changecom], page 65), and macro names (see Section 5.7 [Indir], page 34), with the exception of the nul character (the zero byte ‘’\0’’).

### 宏名称

A name is any sequence of letters, digits, and the character ‘_’ (underscore), where the first character is not a digit. m4 will use the longest such sequence found in the input. If a name has a macro definition, it will be subject to macro expansion (see Chapter 4 [Macros], page 19). Names are case-sensitive.
Examples of legal names are: ‘foo’, ‘_tmp’, and ‘name01’.

### m4 引号输入

A quoted string is a sequence of characters surrounded by quote strings, defaulting to ‘‘’ and ‘’’, where the nested begin and end quotes within the string are balanced. The value of a string token is the text, with one level of quotes stripped o↵. Thus
‘’
)
is the empty string, and double-quoting turns into single-quoting.
‘‘quoted’’ )‘quoted’
The quote characters can be changed at any time, using the builtin macro changequote. See Section 8.2 [Changequote], page 62, for more information.

### 注释

Comments in m4 are normally delimited by the characters ‘#’ and newline. All charac- ters between the comment delimiters are ignored, but the entire comment (including the delimiters) is passed through to the output—comments are not discarded by m4.
Comments cannot be nested, so the first newline after a ‘#’ ends the comment. The commenting e↵ect of the begin-comment string can be inhibited by quoting it.
$ m4
‘quoted text’ # ‘commented text’ )quoted text # ‘commented text’ ‘quoting inhibits’ ‘#’ ‘comments’ )quoting inhibits # comments
The comment delimiters can be changed to any string at any time, using the builtin macro changecom. See Section 8.3 [Changecom], page 65, for more information.

### 其他语法

Any character, that is neither a part of a name, nor of a quoted string, nor a comment, is a token by itself. When not in the context of macro expansion, all of these tokens are just copied to output. However, during macro expansion, whitespace characters (space, tab, newline, formfeed, carriage return, vertical tab), parentheses (‘(’ and ‘)’), comma (‘,’), and dollar (‘$’) have additional roles, explained later.

### m4 输入文件转化为输出文件规则

As m4 reads the input token by token, it will copy each token directly to the output imme-
diately.
The exception is when it finds a word with a macro definition. In that case m4 will calculate the macro’s expansion, possibly reading more input to get the arguments. It then inserts the expansion in front of the remaining input. In other words, the resulting text from a macro call will be read and parsed into tokens again.
m4 expands a macro as soon as possible. If it finds a macro call when collecting the arguments to another, it will expand the second call first. This process continues until there are no more macro calls to expand and all the input has been consumed.
For a running example, examine how m4 handles this input: format(‘Result is %d’, eval(‘2**15’))
First, m4 sees that the token ‘format’ is a macro name, so it collects the tokens ‘(’, ‘‘Result is %d’’, ‘,’, and ‘ ’, before encountering another potential macro. Sure enough, ‘eval’ is a macro name, so the nested argument collection picks up ‘(’, ‘‘2**15’’, and ‘)’, invoking the eval macro with the lone argument of ‘2**15’. The expansion of ‘eval(2**15)’ is ‘32768’, which is then rescanned as the five tokens ‘3’, ‘2’, ‘7’, ‘6’, and ‘8’; and combined with the next ‘)’, the format macro now has all its arguments, as if the user had typed:
     format(‘Result is %d’, 32768)
The format macro expands to ‘Result is 32768’, and we have another round of scanning for the tokens ‘Result’, ‘ ’, ‘is’, ‘ ’, ‘3’, ‘2’, ‘7’, ‘6’, and ‘8’. None of these are macros, so the final output is
)Result is 32768
As a more complicated example, we will contrast an actual code example from the Gnulib project1, showing both a buggy approach and the desired results. The user desires to output a shell assignment statement that takes its argument and turns it into a shell variable by converting it to uppercase and prepending a prefix. The original attempt looks like this:
     changequote([,])dnl
     define([gl_STRING_MODULE_INDICATOR],
       [
         dnl comment
         GNULIB_]translit([$1],[a-z],[A-Z])[=1
       ])dnl
       gl_STRING_MODULE_INDICATOR([strcase])

) GNULIB_strcase=1 )
Oops – the argument did not get capitalized. And although the manual is not able to easily show it, both lines that appear empty actually contain two trailing spaces. By stepping through the parse, it is easy to see what happened. First, m4 sees the token ‘changequote’, which it recognizes as a macro, followed by ‘(’, ‘[’, ‘,’, ‘]’, and ‘)’ to form the argument list. The macro expands to the empty string, but changes the quoting characters to something more useful for generating shell code (unbalanced ‘‘’ and ‘’’ appear all the time in shell scripts, but unbalanced ‘[]’ tend to be rare). Also in the first line, m4 sees the token ‘dnl’, which it recognizes as a builtin macro that consumes the rest of the line, resulting in no output for that line.
The second line starts a macro definition. m4 sees the token ‘define’, which it recognizes as a macro, followed by a ‘(’, ‘[gl_STRING_MODULE_INDICATOR]’, and ‘,’. Because an unquoted comma was encountered, the first argument is known to be the expansion of the single-quoted string token, or ‘gl_STRING_MODULE_INDICATOR’. Next, m4 sees ‘NL’, ‘ ’, and ‘ ’, but this whitespace is discarded as part of argument collection. Then comes a rather lengthy single-quoted string token, ‘[NL dnl commentNL GNULIB_]’. This is followed by the token ‘translit’, which m4 recognizes as a macro name, so a nested macro expansion has started.
The arguments to the translit are found by the tokens ‘(’, ‘[$1]’, ‘,’, ‘[a-z]’, ‘,’, ‘[A-Z]’, and finally ‘)’. All three string arguments are expanded (or in other words, the quotes are stripped), and since neither ‘$’ nor ‘1’ need capitalization, the result of the macro is ‘$1’. This expansion is rescanned, resulting in the two literal characters ‘$’ and ‘1’.
Scanning of the outer macro resumes, and picks up with ‘[=1NL ]’, and finally ‘)’. The collected pieces of expanded text are concatenated, with the end result that the macro ‘gl_STRING_MODULE_INDICATOR’ is now defined to be the sequence ‘NL dnlcommentNL GNULIB_$1=1NL ’. Once again, ‘dnl’ is recognized and avoids a newline in the output.
The final line is then parsed, beginning with ‘ ’ and ‘ ’ that are output literally. Then ‘gl_STRING_MODULE_INDICATOR’ is recognized as a macro name, with an argument list of ‘(’, ‘[strcase]’, and ‘)’. Since the definition of the macro contains the sequence ‘$1’, that sequence is replaced with the argument ‘strcase’ prior to starting the rescan. The rescan sees ‘NL’ and four spaces, which are output literally, then ‘dnl’, which discards the text ‘ commentNL’. Next comes four more spaces, also output literally, and the token ‘GNULIB_strcase’, which resulted from the earlier parameter substitution. Since that is not a macro name, it is output literally, followed by the literal tokens ‘=’, ‘1’, ‘NL’, and two more spaces. Finally, the original ‘NL’ seen after the macro invocation is scanned and output literally.
Now for a corrected approach. This rearranges the use of newlines and whitespace so that less whitespace is output (which, although harmless to shell scripts, can be visually unappealing), and fixes the quoting issues so that the capitalization occurs when the macro ‘gl_STRING_MODULE_INDICATOR’ is invoked, rather then when it is defined. It also adds another layer of quoting to the first argument of translit, to ensure that the output will be rescanned as a string rather than a potential uppercase macro name needing further expansion.

     changequote([,])dnl
     define([gl_STRING_MODULE_INDICATOR],
[dnl comment
       GNULIB_[]translit([[$1]], [a-z], [A-Z])=1dnl
     ])dnl
gl_STRING_MODULE_INDICATOR([strcase]) ) GNULIB_STRCASE=1
The parsing of the first line is unchanged. The second line sees the name of the macro to define, then sees the discarded ‘NL’ and two spaces, as before. But this time, the next token is ‘[dnlcommentNL GNULIB_[]translit([[$1]],[a-z],[A-Z])=1dnlNL]’, which includes nested quotes, followed by ‘)’ to end the macro definition and ‘dnl’ to skip the newline. No early expansion of translit occurs, so the entire string becomes the definition of the macro.
The final line is then parsed, beginning with two spaces that are output literally, and an invocation of gl_STRING_MODULE_INDICATOR with the argument ‘strcase’. Again, the ‘$1’ in the macro definition is substituted prior to rescanning. Rescanning first encounters ‘dnl’, and discards ‘ commentNL’. Then two spaces are output literally. Next comes the token ‘GNULIB_’, but that is not a macro, so it is output literally. The token ‘[]’ is an empty string, so it does not a↵ect output. Then the token ‘translit’ is encountered.
This time, the arguments to translit are parsed as ‘(’, ‘[[strcase]]’, ‘,’, ‘ ’, ‘[a-z]’, ‘,’, ‘ ’, ‘[A-Z]’, and ‘)’. The two spaces are discarded, and the translit results in the desired result ‘[STRCASE]’. This is rescanned, but since it is a string, the quotes are stripped and the only output is a literal ‘STRCASE’. Then the scanner sees ‘=’ and ‘1’, which are output literally, followed by ‘dnl’ which discards the rest of the definition of gl_STRING_MODULE_ INDICATOR. The newline at the end of output is the literal ‘NL’ that appeared after the invocation of the macro.
The order in which m4 expands the macros can be further explored using the trace facilities of GNU m4 (see Section 7.2 [Trace], page 55).
