# SED单行脚本快速参考

在以下地址可找到本文档的最新（英文）版本：
   [http://sed.sourceforge.net/sed1line.txt](http://sed.sourceforge.net/sed1line.txt)
   [http://www.pement.org/sed/sed1line.txt](http://www.pement.org/sed/sed1line.txt)

其他语言版本：

  - 中文  http://sed.sourceforge.net/sed1line_zh-CN.html
  - 捷克语 http://sed.sourceforge.net/sed1line_cz.html
  - 荷语   http://sed.sourceforge.net/sed1line_nl.html
  - 法语   http://sed.sourceforge.net/sed1line_fr.html
  - 德语   http://sed.sourceforge.net/sed1line_de.html
  - 葡语   http://sed.sourceforge.net/sed1line_pt-BR.html


## 文本间隔：

**在每一行后面增加一空行**	

`sed G`

**将原来的所有空行删除并在每一行后面增加一空行**。

`sed '/^$/d;G'` \# 这样在输出的文本中每一行后面将有且只有一空行。

**在每一行后面增加两行空行**

`sed 'G;G'`

**将第一个脚本所产生的所有空行删除（即删除所有偶数行）**

`sed 'n;d'`

**在匹配式样“regex”的行之前插入一空行**

`sed '/regex/{x;p;x;}'`

**在匹配式样“regex”的行之后插入一空行**

`sed '/regex/G'`

**在匹配式样“regex”的行之前和之后各插入一空行**

`sed '/regex/{x;p;x;G;}'`

## 编号：

 1. 为文件中的每一行进行编号（简单的左对齐方式）。这里使用了“制表符”（tab，见本文末尾关于'\t'的用法的描述）而不是空格来对齐边缘。
	` sed = filename | sed 'N;s/\n/\t/'`

 2. 对文件中的所有行编号（行号在左，文字右端对齐）。

	`sed = filename | sed 'N; s/^/     /; s/ *\(.\{6,\}\)\n/\1  /'`


 3. 对文件中的所有行编号，但只显示非空白行的行号。
 `sed '/./=' filename | sed '/./N; s/\n/ /'`

 4. 计算行数 （模拟 "wc -l"）
 `sed -n '$='`

## 文本转换和替代：

 1. Unix环境：转换DOS的新行符（CR/LF）为Unix格式。

    ```shell
    sed 's/.$//' # 假设所有行以CR/LF结束
    sed 's/^M$//'   # 在bash/tcsh中，将按Ctrl-M改为按Ctrl-V
    sed 's/\x0D$//' # ssed、gsed 3.02.80，及更高版本```  
    ```

 2. Unix环境：转换Unix的新行符（LF）为DOS格式。

    ```shell
    sed "s/$/`echo -e \\\r`/" # 在ksh下所使用的命令
    sed 's/$'"/`echo \\\r`/"  # 在bash下所使用的命令`  
    sed "s/$/`echo \\\r`/"   # 在zsh下所使用的命令
    sed 's/$/\r/'        # gsed 3.02.80 及更高版本```
    ```

 3. DOS环境：转换Unix新行符（LF）为DOS格式。

    ```shell
    sed "s/$//"    # 方法 1
    sed -n p       # 方法 2
    ```

 4. DOS环境：转换DOS新行符（CR/LF）为Unix格式。

    下面的脚本只对 `UnxUtils sed 4.0.7` 及更高版本有效。要识别UnxUtils版本的
    sed可以通过其特有的 `--text` 选项。你可以使用帮助选项（`--help`）看
    其中有无一个“--text”项以此来判断所使用的是否是UnxUtils版本。其它DOS
    版本的的sed则无法进行这一转换。但可以用 `tr `来实现这一转换。

    ```shell
    sed "s/\r//" infile >outfile  # UnxUtils sed > 4.0.6
    tr -d \r <infile >outfile    # GNU tr >1.21
    ```

 5. 将每一行前导的“空白字符”（空格，制表符）删除使之左对齐

    `sed 's/^[ \t]*//'  # 见本文末尾关于'\t'用法的描述`

 6. 将每一行拖尾的“空白字符”（空格，制表符）删除

    `sed 's/[ \t]*$//'   # 见本文末尾关于'\t'用法的描述`

 7. 将每一行中的前导和拖尾的空白字符删除

    `sed 's/^[ \t]*//;s/[ \t]*$//'`

 8. 在每一行开头处插入5个空格（使全文向右移动5个字符的位置）

    `sed 's/^/     /'`

 9. 以79个字符为宽度，将所有文本右对齐

    `sed -e :a -e 's/^.\{1,78\}$/ &/;ta'  # 78个字符外加最后的一个空格`

 10. 以79个字符为宽度，使所有文本居中。在方法1中，为了让文本居中每一行的前头和后头都填充了空格。 在方法2中，在居中文本的过程中只在文本的前面填充空格，并且最终这些空格将有一半会被删除。此外每一行的后头并未填充空格。

    ```shell
    sed  -e :a -e 's/^.\{1,77\}$/ & /;ta'                     # 方法1
    sed  -e :a -e 's/^.\{1,77\}$/ &/;ta' -e 's/\( *\)\1/\1/'  # 方法2
    ```

 11. 在每一行中查找字串“foo”，并将找到的“foo”替换为“bar”

    ```shell
    sed 's/foo/bar/'   # 只替换每一行中的第一个“foo”字串
    sed 's/foo/bar/4'  # 只替换每一行中的第四个“foo”字串
    sed 's/foo/bar/g'  # 将每一行中的所有“foo”都换成“bar”
    sed 's/\(.*\)foo\(.*foo\)/\1bar\2/' # 替换倒数第二个“foo”
    sed 's/\(.*\)foo/\1bar/' # 替换最后一个“foo”
    ```
 12. 只在行中出现字串“baz”的情况下将“foo”替换成“bar”

    `sed '/baz/s/foo/bar/g'`

 13. 将“foo”替换成“bar”，并且只在行中未出现字串“baz”的情况下替换
 	
    `sed '/baz/!s/foo/bar/g'`

 14. 不管是 "scarlet", "ruby" 还是 "puce"，一律换成 "red"

    ```shell
    sed 's/scarlet/red/g;s/ruby/red/g;s/puce/red/g'  #对多数的sed都有效
    gsed 's/scarlet\|ruby\|puce/red/g'               # 只对GNU sed有效
    ```
 15. 倒置所有行，第一行成为最后一行，依次类推（模拟“tac”）。
 由于某些原因，使用下面命令时HHsed v1.5会将文件中的空行删除

    ```shell
    sed '1!G;h;$!d'               # 方法1`
    sed -n '1!G;h;$p'             # 方法2`
    ```
 16. 将行中的字符逆序排列，第一个字成为最后一字，……（模拟“rev”）
 `sed '/\n/!G;s/\(.\)\(.*\n\)/&\2\1/;//D;s/.//'`

 17. 将每两行连接成一行（类似“paste”）
 `sed '$!N;s/\n/ /'`

 18. 如果当前行以反斜杠“\”结束，则将下一行并到当前行末尾
 并去掉原来行尾的反斜杠
    `sed -e :a -e '/\\$/N; s/\\\n//; ta'`

 19. 如果当前行以等号开头，将当前行并到上一行末尾并以单个空格代替原来行头的“=”
 `sed -e :a -e '$!N;s/\n=/ /;ta' -e 'P;D'`

 20. 为数字字串增加逗号分隔符号，将“1234567”改为“1,234,567”
 `gsed ':a;s/\B[0-9]\{3\}\>/,&/;ta'                     # GNU sed`
    `sed -e :a -e 's/\(.*[0-9]\)\([0-9]\{3\}\)/\1,\2/;ta'  # 其他sed`

 21. 为带有小数点和负号的数值增加逗号分隔符（GNU sed）
 `gsed -r ':a;s/(^|[^0-9.])([0-9]+)([0-9]{3})/\1\2,\3/g;ta'`

 22. 在每5行后增加一空白行 （在第5，10，15，20，等行后增加一空白行）
 `gsed '0~5G'     # 只对GNU sed有效`
    `sed 'n;n;n;n;G;' # 其他sed`

## 选择性地显示特定行：

 1. 显示文件中的前10行 （模拟“head”的行为）
 sed 10q

 2. 显示文件中的第一行 （模拟“head -1”命令）
 sed q

 3. 显示文件中的最后10行 （模拟“tail”）
 `sed -e :a -e '$q;N;11,$D;ba'`

 4. 显示文件中的最后2行（模拟“tail -2”命令）
 `sed '$!N;$!D`'

 5. 显示文件中的最后一行（模拟“tail -1”）
 `sed '$!d'        # 方法1`
    `sed -n '$p'      # 方法2`

 6. 显示文件中的倒数第二行
 `sed -e '$!{h;d;}' -e x              # 当文件中只有一行时，输入空行`
    `sed -e '1{$q;}' -e '$!{h;d;}' -e x  # 当文件中只有一行时，显示该行`
    `sed -e '1{$d;}' -e '$!{h;d;}' -e x  # 当文件中只有一行时，不输出`

 7. 只显示匹配正则表达式的行（模拟“grep”）
 `sed -n '/regexp/p'               # 方法1`
    `sed '/regexp/!d'                 # 方法2`

 8. 只显示“不”匹配正则表达式的行（模拟“grep -v”）
 `sed -n '/regexp/!p'   # 方法1，与前面的命令相对应`
    `sed '/regexp/d'       # 方法2，类似的语法`

 9. 查找“regexp”并将匹配行的上一行显示出来，但并不显示匹配行
 `sed -n '/regexp/{g;1!p;};h'`

 10. 查找“regexp”并将匹配行的下一行显示出来，但并不显示匹配行
 `sed -n '/regexp/{n;p;}'`

 1. 显示包含“regexp”的行及其前后行，并在第一行之前加上“regexp”所在行的行号 （类似“grep -A1 -B1”）
 `sed -n -e '/regexp/{=;x;1!p;g;$!N;p;D;}' -e h`

1. 显示包含“AAA”、“BBB”或“CCC”的行（任意次序）
 `sed '/AAA/!d; /BBB/!d; /CCC/!d'  # 字串的次序不影响结果`

1. 显示包含“AAA”、“BBB”和“CCC”的行（固定次序）
 `sed '/AAA.*BBB.*CCC/!d'`

1. 显示包含“AAA”“BBB”或“CCC”的行 （模拟“egrep”）
` sed -e '/AAA/b' -e '/BBB/b' -e '/CCC/b' -e d    # 多数sed`
 `gsed '/AAA\|BBB\|CCC/!d'   # 对GNU sed有效`

1. 显示包含“AAA”的段落 （段落间以空行分隔）
  HHsed v1.5 必须在“x;”后加入“G;”，接下来的3个脚本都是这样
   `sed -e '/./{H;$!d;}' -e 'x;/AAA/!d;'`

1. 显示包含“AAA”“BBB”和“CCC”三个字串的段落 （任意次序）
 `sed -e '/./{H;$!d;}' -e 'x;/AAA/!d;/BBB/!d;/CCC/!d'`

1. 显示包含“AAA”、“BBB”、“CCC”三者中任一字串的段落 （任意次序）
 `sed -e '/./{H;$!d;}' -e 'x;/AAA/b' -e '/BBB/b' -e '/CCC/b' -e d`
    `gsed '/./{H;$!d;};x;/AAA\|BBB\|CCC/b;d'         # 只对GNU sed有效`

1. 显示包含65个或以上字符的行
 `sed -n '/^.\{65\}/p'`

1. 显示包含65个以下字符的行
 `sed -n '/^.\{65\}/!p'  # 方法1，与上面的脚本相对应`
    `sed '/^.\{65\}/d'      # 方法2，更简便一点的方法`

1. 显示部分文本——从包含正则表达式的行开始到最后一行结束
 `sed -n '/regexp/,$p'`
21. 显示部分文本——指定行号范围（从第8至第12行，含8和12行）
      ```shell
      sed -n '8,12p'                   # 方法1
      sed '8,12!d'                     # 方法2
      ```

22. 显示第52行

    ```shell
    sed -n '52p'                     # 方法1
    sed '52!d'                       # 方法2
    sed '52q;d'                      # 方法3, 处理大文件时更有效率
    ```
1. 从第3行开始，每7行显示一次    

    ```shell
    gsed -n '3~7p'                   # 只对GNU sed有效
    sed -n '3,${p;n;n;n;n;n;n;}'     # 其他sed
    ```
1. 显示两个正则表达式之间的文本（包含）

	 `sed -n '/Iowa/,/Montana/p'       # 区分大小写方式`

## 选择性地删除特定行：

1. 显示通篇文档，除了两个正则表达式之间的内容

 `sed '/Iowa/,/Montana/d'`

1. 删除文件中相邻的重复行（模拟“uniq”）只保留重复行中的第一行，其他行删除

`sed '$!N; /^\(.*\)\n\1$/!P; D'`

1. 删除文件中的重复行，不管有无相邻。注意hold space所能支持的缓存大小，或者使用GNU sed。
 sed -n 'G; s/\n/&&/; /^\([ -~]*\n\).*\n\1/d; s/\n//; h; P'

1. 删除除重复行外的所有行（模拟“uniq -d”）
 `sed '$!N; s/^\(.*\)\n\1$/\1/; t; D'`

1. 删除文件中开头的10行
 sed '1,10d'

1. 删除文件中的最后一行
 sed '$d'

1. 删除文件中的最后两行
 `sed 'N;$!P;$!D;$d'`

1. 删除文件中的最后10行
 `sed -e :a -e '$d;N;2,10ba' -e 'P;D'   # 方法1`
    `sed -n -e :a -e '1,10!{P;N;D;};N;ba'  # 方法2`

1. 删除8的倍数行
 gsed '0~8d'        # 只对GNU sed有效
    sed 'n;n;n;n;n;n;n;d;'  # 其他sed

1. 删除匹配式样的行
 sed '/pattern/d'                      # 删除含pattern的行。当然pattern
                                       # 可以换成任何有效的正则表达式

1. 删除文件中的所有空行（与“grep '.' ”效果相同）
 sed '/^$/d'                           # 方法1
    sed '/./!d'                           # 方法2

1. 只保留多个相邻空行的第一行。并且删除文件顶部和尾部的空行。
1. （模拟“cat -s”）
 `sed '/./,/^$/!d'        #方法1，删除文件顶部的空行，允许尾部保留一空行`
    `sed '/^$/N;/\n$/D'      #方法2，允许顶部保留一空行，尾部不留空行`

1. 只保留多个相邻空行的前两行。
 `sed '/^$/N;/\n$/N;//D'`

1. 删除文件顶部的所有空行
 sed '/./,$!d'

1. 删除文件尾部的所有空行
 `sed -e :a -e '/^\n*$/{$d;N;ba' -e '}'  # 对所有sed有效`
    `sed -e :a -e '/^\n*$/N;/\n$/ba'        # 同上，但只对 gsed 3.02.*有效`

1. 删除每个段落的最后一行
 `sed -n '/^$/{p;h;};/./{x;/./p;}'`

## 特殊应用：

1. 移除手册页（man page）中的nroff标记。在Unix System V或bash shell下使用'echo'命令时可能需要加上 -e 选项。
 ```sed "s/.`echo \\\b`//g"    # 外层的双括号是必须的（Unix环境）```
    `sed 's/.^H//g'             # 在bash或tcsh中, 按 Ctrl-V 再按 Ctrl-H`
    `sed 's/.\x08//g'    # sed 1.5，GNU sed，ssed所使用的十六进制的表示方法`

1. 提取新闻组或 e-mail 的邮件头
 `sed '/^$/q' # 删除第一行空行后的所有内容`

1. 提取新闻组或 e-mail 的正文部分
 `sed '1,/^$/d'  # 删除第一行空行之前的所有内容`

1. 从邮件头提取“Subject”（标题栏字段），并移除开头的“Subject:”字样
 `sed '/^Subject: */!d; s///;q'`

1. 从邮件头获得回复地址
 `sed '/^Reply-To:/q; /^From:/h; /./d;g;q'`

1. 获取邮件地址。在上一个脚本所产生的那一行邮件头的基础上进一步的将非电邮地址的部分剃除。（见上一脚本）
 `sed 's/ *(.*)//; s/>.*//; s/.*[:<] *//'`

1. 在每一行开头加上一个尖括号和空格（引用信息）
 `sed 's/^/> /'`

1. 将每一行开头处的尖括号和空格删除（解除引用）
 `sed 's/^> //'`

1. 移除大部分的HTML标签（包括跨行标签）
 `sed -e :a -e 's/<[^>]*>//g;/</N;//ba'`

1. 将分成多卷的uuencode文件解码。移除文件头信息，只保留uuencode编码部分。文件必须以特定顺序传给sed。下面第一种版本的脚本可以直接在命令行下输入；第二种版本则可以放入一个带执行权限的shell脚本中。（由Rahul Dhesi的一个脚本修改而来。）
 `sed '/^end/,/^begin/d' file1 file2 ... fileX | uudecode   # vers. 1`
    ` sed '/^end/,/^begin/d' "$@" | uudecode                    # vers. 2`

1. 将文件中的段落以字母顺序排序。段落间以（一行或多行）空行分隔。GNU sed使用字元“\v”来表示垂直制表符，这里用它来作为换行符的占位符——当然你也可以用其他未在文件中使用的字符来代替它。

    ```shell
    sed '/./{H;d;};x;s/\n/={NL}=/g' file | sort | sed '1s/={NL}=//;s/={NL}=/\n/g'
    gsed '/./{H;d};x;y/\n/\v/' file | sort | sed '1s/\v//;y/\v/\n/'
    ```

使用 `SED：Sed` 接受一个或多个编辑命令，并且每读入一行后就依次应用这些命令。
当读入第一行输入后，sed对其应用所有的命令，然后将结果输出。接着再读入第二行输入，对其应用所有的命令……并重复这个过程。上一个例子中sed由标准输入设备（即命令解释器，通常是以管道输入的形式）获得输入。在命令行给出一个或多个文件名作为参数时，这些文件取代标准输入设备成为sed的输入。sed的输出将被送到标准输出（显示器）。因此：

  ```shell
   cat filename | sed '10q'         # 使用管道输入
   sed '10q' filename               # 同样效果，但不使用管道输入
   sed '10q' filename > newfile     # 将输出转移（重定向）到磁盘上
  ```

要了解sed命令的使用说明，包括如何通过脚本文件（而非从命令行）来使用这些命令，请参阅《sed & awk》第二版，作者Dale Dougherty和Arnold Robbins（O'Reilly，1997；[http://www.ora.com](http://www.ora.com)），《UNIX Text Processing》，作者
Dale Dougherty 和 Tim O'Reilly（Hayden Books，1987）或者是Mike Arst写的教程——压缩包的名称是“`U-SEDIT2.ZIP`（在许多站点上都找得到）。 

要发掘sed的潜力，则必须对 `正则表达式` 有足够的理解。正则表达式的资料可以看
《Mastering Regular Expressions》作者Jeffrey Friedl（O'reilly 1997）。  
Unix系统所提供的手册页（“man”）也会有所帮助（试一下这些命令 `man sed`、`man regexp`，或者看 `man ed` 中关于正则表达式的部分），但手册提供的信息比较“抽象”——这也是它一直为人所诟病的。不过，它本来就不是用来教初学者如何使用sed或正则表达式的教材，而只是为那些熟悉这些工具的人提供的一些文本参考。

括号语法：前面的例子对sed命令基本上都使用单引号（`'...'`）而非双引号（`"..."`）这是因为sed通常是在Unix平台上使用。单引号下，Unix的shell（命令解释器）不会对美元符（ \$）和后引号（`...`）进行解释和执行。而在双引号下美元符会被展开为变量或参数的值，后引号中的命令被执行并以输出的结果代替后引号中的内容。而在 `csh` 及其衍生的 shell 中使用感叹号（ \!）时需要在其前面加上转义用的反斜杠（就像这样：\!）以保证上面所使用的例子能正常运行（包括使用单引号的情况下）。DOS版本的Sed则一律使用双引号（"..."）而不是引号来圈起命令。

