---
title: cmake之string命令
tags: tools cmake
mermaid: true
---

# string
## 查找
```cmake
string(FIND <string> <substring> <output_variable> [REVERSE])
```
在<string>中查找<substring>，返回值存放于<output_variable>，找到则返回在<string>中的下标，找不到返回-1。默认为首次出现的匹配，如果使用了REVERSE则为最后一次匹配。注：下标从0开始，以字节做为单位，因此遇到中文时下标表示字符编码第一字节的位置。

## 替换
```cmake
string(REPLACE <match_string> <replace_string> 
		<output_variable> <input> [<input>...])
```
从所有<input> ...中查找<match_string>并使用<replace_string>替换，替换后的字符串存放于<output_variable>。多个输入时，先将所有输入连接后，再做查找替换。

正则表达式
查找
```cmake
string(REGEX MATCH <regular_expression>
       <output_variable> <input> [<input>...])
```
从所有<input> ...中查找<regular_expression>匹配到的字符串，并存放于<output_variable>，查不到输出为空字符串。多个输入时先连接再做操作。只匹配第一次。
```cmake
string(REGEX MATCHALL <regular_expression>
       <output_variable> <input> [<input>...])
```
MATCHALL和MATCH类似，区别是会查找所有符合的匹配，并将他们连接起来输出。

替换
```cmake
string(REGEX REPLACE <regular_expression>
       <replacement_expression> <output_variable>
       <input> [<input>...])
```
根据正则表达式查找，并替换。找不到则输出为输入字符串。<replacement_expression>可以使用\\1 \\2 ..,\\9来表示()元字符匹配到的字符 。这里使用两个\是因为\要先在字符串中转义，然后在正则匹配中进行转义。

元字符
元字符	意义
^	匹配字符串开头
$	匹配字符串结尾
.	匹配任意单个字符
\<char>	转义元字符。在字符串中要使用\\<char>
[]	匹配括号内任何字符
[^]	匹配任何不在括号内的字符
[a-e]	表示范围，意义同[abcde]
*	匹配0次或多次
+	匹配1次或多次
?	匹配0次或1次
|	或运算符
()	保存匹配的子表达式
字符串操作
尾部追加
```cmake
string(APPEND <string_variable> [<input>...])
```
将所有<input>连接成一个字符串后，追加于<string_variable>后面，新字符串赋值于<string_variable>

头部追加
```cmake
string(PREPEND <string_variable> [<input>...])
```
与APPEND相反。将所有<input>连接成一个字符串后，放于<string_variable>前面，新字符串赋值于<string_variable>

连接
```cmake
string(JOIN <glue> <output_variable> [<input>...])
```
使用<glue>作为连接符，连接所有<input>，输出于<output_variable>

大小写转换
```cmake
string(TOLOWER <string> <output_variable>)
```
<string>转换为小写，输出于<output_variable>
```cmake
string(TOUPPER <string> <output_variable>)
```
<string>转换为大写，输出于<output_variable>

长度
```cmake
string(LENGTH <string> <output_variable>)
```
计算<string>的字节长度，输出于<output_variable>。注：中文这种多字节编码的字符，字节长度大于字符长度。

取子串
```cmake
string(SUBSTRING <string> <begin> <length> <output_variable>)
```
从<string>中取出<begin>为起始下标，长度为<length>的子串，输出于<output_variable>。注：如果<length>为-1或大于源字符串长度则子串为余下全部字符串。

删除收尾空白字符
```cmake
string(STRIP <string> <output_variable>)
```
空白字符包括：空格、tab’\t’、换行’\n’

删除子串中的生成器表达式
```cmake
string(GENEX_STRIP <string> <output_variable>)
```
复制字符串
```cmake
string(REPEAT <string> <count> <output_variable>)
```
将<string>复制<count>个并连接在一起，输出于<output_variable>

比较
```cmake
string(COMPARE LESS <string1> <string2> <output_variable>) # 输出1：string1 < string2，否则输出0
string(COMPARE GREATER <string1> <string2> <output_variable>)#输出1：string1 > string2，否则输出0
string(COMPARE EQUAL <string1> <string2> <output_variable>) #输出1：string1 = string2，否则输出0
string(COMPARE NOTEQUAL <string1> <string2> <output_variable>)#输出1：string1 != string2，否则输出0
string(COMPARE LESS_EQUAL <string1> <string2> <output_variable>)#输出1：string1 <= string2，否则输出0
string(COMPARE GREATER_EQUAL <string1> <string2> <output_variable>)#输出1：string1 >= string2，否则输出0
```
字符串加密
```cmake
string(<HASH> <output_variable> <input>)
```
根据指定的编码规则，将<input>进行加密编码，输出于<output_variable>
<HASH>可取值如下：MD5、SHA1、SHA224、SHA256、SHA384、SHA512、SHA3_224、SHA3_256、SHA3_384、SHA3_512

转换
数字/字符互转
```cmake
string(ASCII <number> [<number> ...] <output_variable>)
```
将ASCii码转为对应的字符，如65转为A，66转为B
```cmake
string(HEX <string> <output_variable>)
```
将<string>转换为对应的16进制ASii码，如A转为61（61为A十六进制ascii）。

变量替换
```cmake
string(CONFIGURE <string> <output_variable>
           [@ONLY] [ESCAPE_QUOTES])
# 示例：
set(myVar "test")
string(CONFIGURE "first${myVar}end, next${myVar2}end" POS) #输出POS变量值为"firsttestend, nextend"
```
将<string>字符串中对变量的引用替换为变量值（自定义或系统的），如没有定义变量则取空字符串。

[ESCAPE_QUOTES]表示支持转义字符’\’

_替换
```cmake
string(MAKE_C_IDENTIFIER <string> <output_variable>)
```
将<string>中所有非字符数字替换为_。如+hello*world()会被替换为_hello_world__

UUID生成
```cmake
string(UUID <output_variable> NAMESPACE <namespace> NAME <name>
       TYPE <MD5|SHA1> [UPPER])
```
根据NAMESPACE和NAME生成一个UUID字符串。其中NAMESPACE必须为UUID格式的字符串。相同的NAMESPACE和NAME生成的UUID是相同的。
UUID字符串格式xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
[UPPER]表示输出的UUID字符串为大写
TYPE <MD5|SHA1>表示生成UUID时使用的算法。