# 文本处理工具使用总结
本文总结了文本处理工具grep, vim，sed，awk使用方法。
## grep
```bash
grep是文本过滤工具，使用正则表达式形式的PATTERN过滤匹配的文本至标准输出，不对文件内容修改编辑。
egrep | grep -E -- 支持使用扩展的正则表达式形式的PATTERN
fgrep -- 不支持正则表达式的元字符；当无需使用正则表达式元字符编写过滤PATTERN时，fgrep的性能更好
```
**正则表达式元字符**
```bash
 1. 字符匹配
    .:匹配任意单个字符
    []:匹配范围内任意单个字符
    [^]:匹配范围外的任意单个字符
 2. 次数匹配
    *：匹配其前面字符任意次
    \?:匹配前面字符0次或1次
    \+:匹配前面字符至少1次
    \{m\}:匹配前面字符m次
    \{m,n\}:匹配前面字符至少m次，至多n次
 3. 位置锚定
     ^:行首锚定
     $:行尾锚定
     ^PATTERN$:匹配整行
     \<或\b:锚定词首
     \>或\b:锚定词尾
     \<PATTERN\>或\bPATTERN\b:精确匹配单词
 4. 分组及引用
      \(\):将一个或多个字符捆绑在一起，当作一个整体处理
      note：分组括号中的模式匹配到的内容会被正则表达式引擎自动记录于内部的变量中
         这些变量为：\1:模式从左侧起，第一个左括号以及与其匹配的右括号之间的模式匹配到的内容
                    \2:模式从左侧起，第二个左括号以及与其匹配的右括号之间的模式匹配到的内容
      后向引用：引用前面分组括号中模式匹配到的字符
```
## vim使用
vim是一个全屏文本编辑器，有不同的工作模式，在特定工作模式下有各种命令以实现文本处理；vim常用的工作模式有Normal, Insert, Ex，Visual四种；模式间可以切换；其中Normal模式是vim打开文件的默认模式，也被翻译为编辑模式，在此模式下，可以使用特定键盘键实现光标跳转、文本内容删除、复制、粘贴；Insert模式下可以在指定位置插入字符，所以中文称为插入模式；Ex模式是vim内建的命令行接口，在此模式下，可以给定地址范围查找文本内容，或查找替换文本内容，执行脚本外部命令等，因Ex模式下在屏幕最后一行输入执行操作，所以中文也称为末行模式；Visual模式下可以使用键盘的上下左右方向键选取文件，中文称为可视化模式，结合编辑命令使用；Normal模式切换至Insert模式可以使用i,a,o,I,A,O等字符键实现，不同的字符切换指定Insert模式插入位置;Insert模式切换至Normal模式使用esc键即可；Normal模式切换至Ex模式使用：键，esc键返回Normal模式；Normal模式切换至Visual模式使用v,V字符键，同样esc键返回Normal模式；下面详细介绍每种工作模式下可实现的操作。
```bash
使用vim打开文件

  #vim [options] FILENAME
options:
  +#:指定文件打开后光标位置处在第#行行首，不指定行号则默认文件尾部行首
  +PATTERN：光标在PATTERN第一个匹配到行行首，PATTERN支持正则表达式

关闭文件

  Normal模式：ZZ，保存并退出；ZQ，不保存退出

  Ex模式：
   + :q!，强制退出
   - :wq, 保存并退出；或者 :x
   * :w /PATH/TO/SOME，另存为
```

**Normal模式**
```bash
normal模式下可实现光标按字符、单词、行跳转；文本内容删除、复制、粘贴；

按字符跳转：h, l, j, k字符键，依次为光标向左、右、下、上移动，前面加数字可指定依次跳转的字符数；

按单词跳转：
   + w:跳转至下一个单词词首
   - b:返回至上一个单词词首
   * e:跳转至单词词尾

按行跳转：
   + ^: 光标由当前位置跳转至行首
   - $: 光标由当前位置跳转至行尾
   * #G：跳转至指定行行首；1G，gg，跳转至第一行行首；G，跳转至最后一行行首

以上操作均可在字符键前指定数字。

文本编辑操作：可结合光标跳转字符，实现范围删除、复制、粘贴
   + 字符删除：x
   - 字符替换：tCHAR
   + 文本删除：d字符键
   - 文本复制：y字符键
   * 文本粘贴：p,P字符键

撤销操作：
   + u:撤销上一次编辑操作
   - U：撤销光标所在行的所有操作
   * crtl+r:撤销“撤销操作”
```
**Insert模式**
```bash
Insert模式下可以实现在文本插入字符。
```
**Ex模式**
```bash

Ex模式可以进行地址定界，可以结合编辑命令范围处理，在指定地址范围进行查找、查找替换文本内容。

 1. 地址定界

   + #:特定的第#行
   - #，#:指定行范围
   * #,+#:指定行范围，左侧为起始行的绝对编号，右侧数字为相对于左侧行号的偏移量；
   + 1,$或%：全文
   - /pattern/:从光标所在处起始向文件尾部第一次被模式匹配到的行. PATTERN支持正则表达式
   * /pattern1/,/pattern2/:从光标所在处起始，第一次由pattern1匹配到的行开始，至第一次被pattern2匹配到的行结束

可以使用 w /PATH/TO/SOMEFILE 将范围内的文本保存至指定文件;

使用 r /PATH/FROM/SOMEFILE:将指定文件中的文本读入并插入至指定位置 ；不指定地址范围默认全文 

2. 查找

   + :/PATTERN, 从当前光标所在处向文件尾部查找能够被模式匹配到的所有字符串
   - :?PATTERN, 从当前光标所在处向文件首部查找能够被模式匹配到的所有字符串
3. 查找替换

  使用格式：
   :地址定界s@查找内容@替换内容@修饰符

   * 要查找的内容：可使用正则表达式
   * 替换为的内容：不能使用正则表达式，但可以引用；
           如果“要查找内容”的模式中使用分组符号，则在“替换为的内容”中可以使用后向引用；
            或者直接引用查找模式匹配到的全部文本，使用&符号实现；
   * 修饰符：
           i：查找时忽略大小写
           g：全局替换，默认只替换每一行第一次匹配到的内容。
```
**定制vim的工作特性**
```bash
可以在针对全局或用户个人的vim的配置文件中定制vim 的工作特性；
vim的全局配置文件/etc/vimrc;
用户个人vim配置文件在其家目录下~/.vimrc;
```
## sed
sed是行文本编辑器，一次处理一行内容；sed并不在原处修改文件内容，而是有临时的缓冲区，称为模式空间，处理时将文件从第一行开始依次读入模式空间，一次一行，sed判断读入行是否在地址定界范围内，如果不是，不进行操作直接输出模式空间内容至标准输出，如果是地址范围内的行，sed按指定动作对行进行处理后输出模式空间内容至标准输出。sed可以批量处理文件，有相同于vim中的地址定界方式，可进行查找和查找替换等操作；使用格式如下：
```bash
  #sed [options] 'script' FILE1 FILE2 ...
options:
  -n:不显示模式空间内容
  -r:支持使用扩展正则表达式
  -f /PATH/TO/FILE:从指定文件读取编辑脚本
  -e:支持多点编辑
  -i.bak:原处编辑并备份原文件，备份文件后缀为.bak
  
'script'中内容包括地址范围和执行命令；'地址范围编辑命令1;编辑命令2;...'
1. 地址定界

   (1) 不给地址：对全文进行逐行处理
   (2) 单地址：
               #；指定行号
       /pattern/:被pattern匹配到的每一行
   (3) 地址范围：
         * #,#；指定行号范围
         * #,+#；#指定起始行号，+#指定偏移量
         * /pat1/,/pat2/：从第一次被/pat1/匹配到的行至第一次被/pat2/匹配到的行
         * #,/pat1/：从#到第一次被/pat1/匹配到的行
        
   (4) #~#:起始行号#，~#步进行数

2. 编辑命令

   p:打印匹配的行
   d:删除匹配到的行
   a[\]TEXT:支持换行符\n进行多行追加
   i[\]TEXT:支持换行符\n进行多行插入
   c[\]TEXT:替换行内容  
   i.backup: 原地修改文件并备份原文件，备份文件名称后缀是.backup
   w /PATH/TO/SOMEFILE:匹配到的行储存至指定文件
   r /PATH/FROM/SOMEFILE: 读取指定文件内容至sed模式空间匹配到的行后
   =:为模式空间中行打印行号
   ！：非模式空间匹配到行
   查找替换：s@查找内容@替换内容@修饰符；查找替换使用同vim。
            (1) 要查找的内容：可使用正则表达式
            (2) 替换为的内容：不能使用正则表达式，但可以引用；
               如果“要查找内容”的模式中使用分组符号，则在“替换为的内容”中可以使用后向引用；
                  或者直接引用查找模式匹配到的全部文本，使用&符号实现；
            (3) 修饰符：        
                  g：全局替换，默认只替换每一行第一次匹配到的内容。
                  p:打印替换成功的行
                  w /PATH/TO/SOMEFILE:替换成功的行另存为指定文件
```
**sed高级编辑命令**
```bash
 高级编辑命令：
 + h:把模式空间中的内容覆盖至保持空间中；原有保持空间中内容将被覆盖
 - H：把模式空间中内容追加至保持空间
 * g:从保持空间取出数据覆盖至模式空间
 * G:从保持空间取出内容追加至模式空间
 + x：把模式空间中内容与保持空间中内容进行互换
 - n:读取匹配到的行的下一行至模式空间
 + N:追加匹配到的行的下一行至模式空间
 * d:删除模式空间中的行
 + D:删除多行模式空间中的行
```

## awk
awk是文本处理工具，awk也被认为是一个特殊用途的编程语言，可以快速和简单过滤文本并格式化输出；Gawk是GUN组织基于awk编程语言实现，是GNU版本的awk，linux系统使用的即使gawk。

**awk的工作模式**
```bash
一次读入一行文本，按指定的输入分隔符切割为n个字段(默认分隔符为空白字符)，每个字段保存在awk的内建变量中，依次为$1, $2, ...
  $0:整行文本
  $1:第一个字段
  $2:第二个字段
  ...
awk同样可以实现地址定界，仅处理指定范围的文本行；如果不指定，awk会遍历整个文本；而且可以通过循环遍历每一行的每个字段；awk有自己内建变量、数组、函数，可以自定义变量、数组、函数；awk的action可以实现格式化文本输出，条件判断和循环处理。

使用格式：
  gawk [options] 'program' FILE ...

     program的内容组成: PATTERN{ACTION STATEMENTS} ；语句之间用“;”分隔
   
```
**1. awk的变量**
```bash
1.1 内建变量
 (1) FS: input feild seperator,默认为空白字符
   指定输入分隔符
   # awk -v FS=':' '{print $1}' /etc/passwd 
 或
   # awk -F':'' {print $1}' /etc/passwd 
 (2) OFS:output feild seperator,默认为空白字符
    指定输出分隔符
   # awk -v FS=':' -v OFS=':' '{print $1,$2}' /etc/passwd 
 (3) RS：input record seperator,输入数据的指定换行符；
   # awk -v RS=' ' '{print}' /etc/passwd 
 (4) ORS:output record seperator, 输出指定换行符；
   # awk -v RS=' ' -v ORS=' ' '{print}' /etc/passwd

 (5) NF: Number of Feilds， 每行slice个数存储在变量NF中
    # awk -F':' '{print NF}' /etc/passwd 
         $NF:每行最后一个字段
         {print NF}
         {print $NF};
 (6) NR: Number of Records，文本行数储存在NR变量中
     FNR：各文件分别计数行数
      # awk '{print NR}' /etc/passwd /etc/fstab ；
      # awk '{print FNR}' /etc/passwd /etc/fstab; 两文件分别计数编号
            
 (7) FILENAME：当前文件名

 (8) ARGC：命令行中参数个数；
      # awk '{print ARGC}' /etc/fstab /etc/issue 
            3
            ...
 (9) ARGV：内建数组，保存命令行所给定的各参数；

   [root@centos7 ~]#awk 'BEGIN{print ARGC}' /etc/fstab /etc/issue
   3
   [root@centos7 ~]#awk 'BEGIN{print ARGV[0]}' /etc/fstab /etc/issue
   awk
   [root@centos7 ~]#awk 'BEGIN{print ARGV[1]}' /etc/fstab /etc/issue
   /etc/fstab
   [root@centos7 ~]#awk 'BEGIN{print ARGV[2]}' /etc/fstab /etc/issue
   /etc/issue

1.2 自定义变量
 (1) -v var=value 
   变量名区分字符大小写；
   # awk -v test="hello awk" '{print test, $0}' /etc/issue  
(2) 在program中直接定义
   # awk '{test="hello awk";print test,$1}' /etc/issue

   [root@centos7 ~]#awk -v test="hello awk" '{print test,$1}' /etc/issue
   hello awk \S
   hello awk Kernel
   hello awk 
   [root@centos7 ~]#awk '{test="hello awk";print test,$1}' /etc/issue
   hello awk \S
   hello awk Kernel
   hello awk 
注意：awk中引用变量不需要加$符
```
**2. awk的常用action**

2.1  print
```bash
使用格式：
   print item1, item2, ...
要点：
   (1)逗号分隔每个item
   (2)输出的各item可以是字符串，也可以是数值，或当前记录的字段、变量或awk的表达式；
   (3)如果省略item，相当于{print $0}
```
2.2 printf--实现格式化输出

```bash
使用格式
 printf FORMAT, item1, item2, ...

   FORMAT:定义格式，各item按FORMAT定义的格式输出；
         (1) FORMAT必须给出；
         (2) 不会自动换行，需要显示给出换行控制符，\n
         (3) FORMAT中需要分别为后面的每一个item指定一个格式化符号；
     格式符：
         %c:显示字符的ASCII码；
         %d，%i:显示十进制证书；
         %e,%E:科学记数法数字显示
         %f:显示为浮点数；
         %g,%G:以科学记数法或浮点形式显示数值；
         %s:显示字符串；
         %U:无符号整数；
         %%：%
      [root@centos7 ~]#awk -F':' '{printf "Username: %s, Uid: %d\n", $1,$3}' /etc/passwd
            Username: root, Uid: 0
            Username: bin, Uid: 1
            Username: daemon, Uid: 2
            Username: adm, Uid: 3
      修饰符：--用于格式符前面起到修饰作用
         #[.#]:第一个#数字控制显示的宽度；第二个#表示小数点后的精度；
            %3.1f 
            -:左对齐，默认为右对齐
            +：显示数值符号
        4. 操作符
           
         (1) 算数运算操作符：+, -, *, /, ^, %,
            -x：转换为负数
            +x:转换为数值； 
         (2) 字符串操作符：没有符号的操作符，字符串连接
         (3) 赋值操作符：
             =，+=, -=, *=, /=, %=, ^=, ++, --
         (4) 比较操作符：
             >, >=, <, <=, !=, == 
         (5) 模式匹配符：
            ~ :左侧字符串是否可以匹配右边模式
            !~:左侧字符串是否不匹配右边模式
         (6) 逻辑操作符：
              与 &&
              或 ||
              非 !
         (7) 函数调用：
             function_name(argu1,argu2,...)
         (8) 内建条件表达式：

             selector?if-true-expression:if-false-expression

            [root@centos7 ~]#awk -F':' '{$3 >= 1000?usertype="Comon user":usertype="Sysadmin or Sysuser";printf "%15s:%s\n",$1,usertype}' /etc/passwd
               root:Sysadmin or Sysuser
               bin:Sysadmin or Sysuser
            [root@centos7 ~]#awk -F':' '$NF=="/bin/bash"{printf "%15s, %s\n", $1,$NF}' /etc/passwd
               root, /bin/bash
               moon, /bin/bash
```
2.3 控制语句
```bash  
    
     if(condition) {statements}
     if(condition) {statements} else {statement}
     while(condition) {statements}
     do {statements} while(condition)
     for(expr1;expr2;expr3) {statements}
     break 
     continue 
     delete array[index]
     delete array
     exit 
     { statements }；多个语句组成组合代码使用{}扩起来；

 1. if-else 
       
   语法：if(condition) {statement} [else {statement}]
   使用场景：对awk取得字段或整行进行条件判断

   [root@centos7 ~]#awk -F: '{if($3>=1000) print $1,$3}' /etc/passwd
    moon 1000
   [root@centos7 ~]#awk -F: '{if($NF=="/sbin/nologin") print $1,$NF}' /etc/passwd
   bin /sbin/nologin
   [root@centos7 ~]#awk -F: '{if($3>=1000) {printf "common user: %s\n",$1} else {printf "root or sysuser: %s\n", $1}}' /etc/passwd
    root or sysuser: root
    root or sysuser: bin
   [root@centos7 ~]#df -h | awk -F"%" '{print $1}' | awk '/\/dev\/sd*/{if($NF>=7) {print $1,$NF}}'
    /dev/sda1 7
  
 2. while 循环

   语法： while(condition) statement 
         条件为“真”，进入循环；条件为“假”，退出循环；
   使用场景：对一行内的多个字段逐一类似处理时使用；对数组中各元素逐一处理时使用；

    [root@centos7 ~]#awk -F: 'NR>=1&&NR<=2{i=1;while(i<=NF) {print $i,length($i);i++}}' /etc/passwd
      root 4
      x 1
      0 1
      0 1
      root 4
      /root 5
      /bin/bash 9
    [root@centos7 ~]#awk -F: 'NR>=1&&NR<=2{i=1;while(i<=NF) {if(length($i)>=5) {print $i,length($i)};i++}}' /etc/passwd
      /root 5
      /bin/bash 9
      /sbin/nologin 13

3. do-while循环

   语法： do statement while(condition)
         不管条件是否为真，至少执行一次循环体 
4. for循环
     
   语法： for(expr1;expr2;expr3) statement 
               expr1:初始变量赋值
               expr2:条件判断表达式
               expr3:变量修正的表达式
        
    [root@centos7 ~]#awk -F: 'NR<=2{for(i=1;i<=NF;i++) {if(length($i)>=4){print $i, length($i)}}}' /etc/passwd
      root 4
      root 4
      /root 5
      /bin/bash 9
      /bin 4
      /sbin/nologin 13

   for循环特殊用法：--遍历数组中的元素，结合关联数组使用
     语法： for(var in array) {statement} 
    
5. switch语句
   语法： switch(expression) {case VALUE1 or /REGEXP/: statement; case VALUE2 or /REGEXP2/: statement;....;default:statement}
6. break和continue 

   break [n]：退出n层循环
   continue:退出本轮循环提前进入下一轮循环；提前结束对本字段处理而直接进入下一个字段处理

7. next  

   提前结束对本行的处理而直接进入下一行；控制awk的内生循环直接进入下一行处理
   [root@centos7 ~]#awk -F: '{if($3%2!=0) next; print $1,$3}' /etc/passwd  
                  

```
**3. awk的数组**
```bash

关联数组：array[index-expression]
   index-expression:
   (1) 可以使用任意字符串；字符串要使用双引号“string”
   (2) 如果某数组元素事先不存在，在引用时，awk会自动创建此元素，并将其值初始化为“空串”，如果当数值使用则当作0

   1. 若要判断数组中是否存在某元素，要使用“index in array”格式进行；
           weekdays["mon"]="Monday"
          
   2. 若要遍历数组中的每一个元素，要使用for循环
      for(var in array) print array[var]
      注意：var会遍历array的每个索引；
      state["LISTEN"]++
      state["ESTABLISHED"]++
     应用：
      统计数组元素出现次数
         # awk -F' FS' '{array[$i]++}END{for(i in array) {print i, array[i]}}'

         [root@centos7 ~]#netstat -tan | awk '/^tcp\>/{state[$NF]++}END{for(i in state) {print i,state[i]}}'
         LISTEN 1
         ESTABLISHED 2
         [root@centos7 ~]#awk '{ip[$1]++}END{for(i in ip) {print i,ip[i]}}' /var/log/httpd/access_log
         172.18.133.58 17
         [root@centos7 ~]#awk '/^UUID/{fs[$3]++}END{for(i in fs) {print i, fs[i]}}' /etc/fstab
         swap 1
         xfs 3
         [root@centos7 ~]#awk '{for(i=1;i<=NF;i++){word[$i]++}}END{for(i in word) {print i,word[i]}}' /etc/fstab
```
**4. awk地址定界--PATTERN**
```bash
 pattern：--地址定界 

     1. empty:空模式，匹配每一行；
     2. /regular expression/:模式匹配到的行
     3.!/regular expression/:处理匹配行以外的行
     4. relational expression:关系表达式；结果有“真”“假”，结果为真的才会被处理；
        真：结果为非0值；非空字符串
        假：0，空字符串；
     5. line range ;
         /pat1/,/pat2/
        [root@centos7 ~]#awk -F: 'NR>=1&&NR<=11{print $1}' /etc/passwd
       注意：不支持直接给出行号数字的格式
     6. BEGIN/END模式
        BEGIN{}:仅在开始处理文件文本前执行一次的操作；
        END{}:仅在文本处理完成之后命令完成之前执行1次的操作；

    模式匹配
      [root@centos7 ~]#awk -F':' '$NF~/nologin$/{printf "%15s: %s\n", $1,$NF}' /etc/passwd
        bin: /sbin/nologin
        daemon: /sbin/nologin
        adm: /sbin/nologin
```
