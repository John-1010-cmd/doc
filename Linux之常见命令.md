---
title: Linux之常见命令
date: 2023-05-06
updated : 2023-05-06
categories: 
- Linux
tags: 
- Linux
description: 这是一篇关于Linux的常见命令的Blog。
---

## 基本操作命令

Tab 按键：命令补齐功能

Ctrl+c 按键：停掉正在运行的程序

Ctrl+d 按键：相当于exit，退出

Ctrl+l 按键：清屏

### 关机和重启

**关机命令：shutdown**

正确的关机流程为：sync > shutdown > reboot > halt

```shell
sync #将数据由内存同步到硬盘中。

shutdown #关机指令，你可以man shutdown 来看一下帮助文档。例如你可以运行如下命令关机：

shutdown –h 10 #‘This server will shutdown after 10 mins’ 这个命令告诉大家，计算机将在10分钟后关机，并且会显示在登陆用户的当前屏幕中。

shutdown –h now #立马关机

shutdown –h 20:25 #系统会在今天20:25关机

shutdown –h +10 #十分钟后关机

shutdown –r now #系统立马重启

shutdown –r +10 #系统十分钟后重启

reboot #就是重启，等同于 shutdown –r now

halt #关闭系统，等同于shutdown –h now 和 poweroff

shutdown -c #取消定时关机
```

不管是重启系统还是关闭系统，首先要运行 sync 命令，把内存中的数据写到磁盘中。

关机的命令有 shutdown –h now halt poweroff 和 init 0 , 重启系统的命令有 shutdown –r now reboot init 6。

**重启命令：reboot**

### 帮助命令

-help命令

```shell
shutdown --help
ifconfig --help #查看网卡信息
```

man命令（命令说明书）

```shell
man shutdown #man shutdown打开命令说明书后，按q键退出
```

## 目录操作命令

- 绝对路径
  路径的写法，由根目录 / 写起，例如： /usr/share/doc 这个目录。
- 相对路径
  路径的写法，不是由 / 写起，例如由 /usr/share/doc 要到 /usr/share/man 底下时，可以写成： cd …/man

### 目录切换cd

```shell
cd /        #切换到根目录
cd /usr     #切换到根目录下的usr目录
cd ../      #切换到上一级目录 或者  cd ..
cd ~        #切换到home目录
cd -        #切换到上次访问的目录
```

### 目录查看ls

ls 查看当前目录下的所有目录和文件
ls -a 查看当前目录下的所有目录和文件（包括隐藏的文件）
ls -l 或 ll 列表查看当前目录下的所有目录和文件（列表查看，显示更多信息）
ls /dir 查看指定目录下的所有目录和文件 如：ls /usr

### 目录操作

#### 创建目录mkdir

mkdir [-mp] 目录名称

- -m ：配置文件的权限
- -p ：将所需要的目录(包含上一级目录)递归创建

#### 删除目录或文件rm

rm [-fir] 文件或目录

- -f ：就是 force 的意思，忽略不存在的文件，不会出现警告信息；
- -i ：互动模式，在删除前会询问使用者是否动作
- -r ：递归删除！最常用在目录的删除！这是非常危险的选项！

**删除文件**

rm 文件 删除当前目录下的文件
rm -f 文件 删除当前目录的的文件（不询问）

**删除目录**

rm -r aaa 递归删除当前目录下的aaa目录
rm -rf aaa 递归删除当前目录下的aaa目录（不询问）

**全部删除**

rm -rf * 将当前目录下的所有目录和文件全部删除
rm -rf /* **【自杀命令！慎用！】**将根目录下的所有文件全部删除

无论删除任何目录或文件，都直接使用 rm -rf 目录/文件/压缩包

#### 删除空目录rmdir 

rmdir [-p] 目录名称

-p ：连同上一级空的目录也一起删除

 rmdir 仅能删除空的目录，可以使用 rm 命令来删除非空目录。

#### 目录修改mv和cp

**mv (移动文件与目录，或修改名称)**

mv [-fiu] source destination

mv [options] source1 source2 source3 .... directory

- -f ：force 强制的意思，如果目标文件已经存在，不会询问而直接覆盖；
- -i ：若目标文件 (destination) 已经存在时，就会询问是否覆盖！
- -u ：若目标文件已经存在，且 source 比较新，才会升级 (update)

**cp (复制文件或目录)**

cp [-adfilprsu] source destination

cp [options] source1 source2 source3 .... directory

-a：相当於 -pdr 的意思；(常用)
-d：若来源档为连结档的属性(link file)，则复制连结档属性而非文件本身；
-f：为强制(force)的意思，若目标文件已经存在且无法开启，则移除后再尝试一次；
-i：若目标档(destination)已经存在时，在覆盖时会先询问动作的进行(常用)
-l：进行硬式连结(hard link)的连结档创建，而非复制文件本身；
-p：连同文件的属性一起复制过去，而非使用默认属性(备份常用)；
-r：递归持续复制，用於目录的复制行为；(常用)
-s：复制成为符号连结档 (symbolic link)，亦即捷径文件；
-u：若 destination 比 source 旧才升级 destination ！

**重命名目录**

mv 当前目录 新目录

mv aaa bbb 将目录aaa改为bbb

mv的语法不仅可以对目录进行重命名而且也可以对各种文件，压缩包等进行 重命名的操作

**剪切目录**

mv 目录名称 目录的新位置

将/usr/tmp目录下的aaa目录剪切到 /usr目录下面 mv /usr/tmp/aaa /usr

mv语法不仅可以对目录进行剪切操作，对文件和压缩包等都可执行剪切操作

**拷贝目录**

cp -r 目录名称 目录拷贝的目标位置 -r代表递归

将/usr/tmp目录下的aaa目录复制到 /usr目录下面 cp /usr/tmp/aaa /usr

cp命令不仅可以拷贝目录还可以拷贝文件，压缩包等，拷贝文件和压缩包时不 用写-r递归

### 搜索目录find

find 目录 参数 文件名称

```shell
find . -name "*.c" #将目前目录及其子目录下所有延伸档名是 c 的文件列出来。
find . -type f     #将目前目录其其下子目录中所有一般文件列出
find . -ctime -20  #将目前目录及其子目录下所有最近 20 天内更新过的文件列出
```

#### 当前目录显示pwd

pwd [-P]

- **-P** ：显示出确实的路径，而非使用连结 (link) 路径。

## 文件操作命令

### 文件操作

#### 新建文件touch

touch命令用于修改文件或者目录的时间属性，包括存取时间和更改时间。若文件不存在，系统会建立一个新的文件。

touch [-acfm] [-d<日期时间>] [-r<参考文件或目录>] [-t<日期时间>] [--help] [--version] [文件或目录…]

- a 改变档案的读取时间记录。
- m 改变档案的修改时间记录。
- c 假如目的档案不存在，不会建立新的档案。与 --no-create 的效果一样。
- f 不使用，是为了与其他 unix 系统的相容性而保留。
- r 使用参考档的时间记录，与 --file 的效果一样。
- d 设定时间与日期，可以使用各种不同的格式。
- t 设定档案的时间记录，格式与 date 指令相同。
- –no-create 不会建立新档案。
- –help 列出指令格式。
- –version 列出版本讯息。

```shell
touch testfile  #修改文件"testfile"的时间属性为当前系统时间
touch file      #指定的文件不存在，则将创建一个新的空白文件
```

#### 删除文件rm

rm [-fir] 文件或目录

- -f ：就是 force 的意思，忽略不存在的文件，不会出现警告信息；
- -i ：互动模式，在删除前会询问使用者是否动作
- -r ：递归删除，这是非常危险的选项！！！

#### 修改文件vi或vim

**命令模式**

- **i** 切换到输入模式，以输入字符。
- **x** 删除当前光标所在处的字符。
- **:** 切换到底线命令模式，以在最底一行输入命令。

**输入模式**

- 字符按键以及Shift组合，输入字符
- ENTER，回车键，换行
- BACK SPACE，退格键，删除光标前一个字符
- DEL，删除键，删除光标后一个字符
- 方向键，在文本中移动光标
- HOME/END，移动光标到行首/行尾
- Page Up/Page Down，上/下翻页
- Insert，切换光标为输入/替换模式，光标将变成竖线/下划线
- ESC，退出输入模式，切换到命令模式

**底线命令模式**

在命令模式下按下:（英文冒号）就进入了底线命令模式。

- q 退出程序
- w 保存文件

**打开文件**

vi 文件名

打开当前目录下的aa.txt文件 vi aa.txt 或者 vim aa.txt

**编辑文件**

使用vi编辑器打开文件后点击按键：i ，a或者o即可进入编辑模式。

- i:在光标所在字符前开始插入
- a:在光标所在字符后开始插入
- o:在光标所在行的下面另起一新行插入

**保存文件**

第一步：ESC 进入命令行模式
第二步：: 进入底行模式
第三步：wq 保存并退出编辑

**取消编辑**

第一步：ESC 进入命令行模式
第二步：: 进入底行模式
第三步：q! 撤销本次修改并退出编辑

#### 文件的查看

- cat 由第一行开始显示文件内容-A ：相当於 -vET 的整合选项，可列出一些特殊字符而不是空白而已；
  -b ：列出行号，仅针对非空白行做行号显示，空白行不标行号！
  -E ：将结尾的断行字节 $ 显示出来；
  -n ：列印出行号，连同空白行也会有行号，与 -b 的选项不同；
  -T ：将 [tab] 按键以 ^I 显示出来；
  -v ：列出一些看不出来的特殊字符

- tac 从最后一行开始显示，可以看出 tac 是 cat 的倒着写！

- nl 显示的时候，顺道输出行号！
  -b ：指定行号指定的方式，主要有两种：
  -b a ：表示不论是否为空行，也同样列出行号(类似 cat -n)；
  -b t ：如果有空行，空的那一行不要列出行号(默认值)；
  -n ：列出行号表示的方法，主要有三种：
  -n ln ：行号在荧幕的最左方显示；
  -n rn ：行号在自己栏位的最右方显示，且不加 0 ；
  -n rz ：行号在自己栏位的最右方显示，且加 0 ；
  -w ：行号栏位的占用的位数。

- more 一页一页的显示文件内容

  空白键 (space)：代表向下翻一页；
  Enter ：代表向下翻『一行』；
  /字串 ：代表在这个显示的内容当中，向下搜寻『字串』这个关键字；
  :f ：立刻显示出档名以及目前显示的行数；
  q ：代表立刻离开 more ，不再显示该文件内容。
  b 或 [ctrl]-b ：代表往回翻页，不过这动作只对文件有用，对管线无用。

- less 与 more 类似，但是比 more 更好的是，他可以往前翻页！

  空白键 ：向下翻动一页；
  [pagedown]：向下翻动一页；
  [pageup] ：向上翻动一页；
  /字串 ：向下搜寻『字串』的功能；
  ?字串 ：向上搜寻『字串』的功能；
  n ：重复前一个搜寻 (与 / 或 ? 有关！)
  N ：反向的重复前一个搜寻 (与 / 或 ? 有关！)
  q ：离开 less 这个程序；

- head 只看头几行

  -n ：后面接数字，代表显示几行的意思

- tail 只看尾巴几行

  -n ：后面接数字，代表显示几行的意思
  -f ：表示持续侦测后面所接的档名，要等到按下[ctrl]-c才会结束tail的侦测

#### 权限修改

chmod [-cfvR] [--help] [--version] mode file...

mode : 权限设定字串，格式：[ugoa...] [[+-=] [rwxX]...] [,...]

- u 表示该文件的拥有者，g 表示与该文件的拥有者属于同一个群体(group)者，o 表示其他以外的人，a 表示这三者皆是。

+ +表示增加权限、- 表示取消权限、= 表示唯一设定权限。
+ r 表示可读取，w 表示可写入，x 表示可执行，X 表示只有当该文件是个子目录或者该文件已经被设定过为可执行。

- -c : 若该文件权限确实已经更改，才显示其更改动作
- -f : 若该文件权限无法被更改也不要显示错误讯息
- -v : 显示权限变更的详细资料
- -R : 对目前目录下的所有文件与子目录进行相同的权限变更(即以递回的方式逐个变更)
- –help : 显示辅助说明
- –version : 显示版本

权限的设定方法有两种， 分别可以使用数字或者是符号来进行权限的变更。

## 压缩文件操作

Linux 常用的压缩与解压缩命令有：tar、gzip、gunzip、bzip2、bunzip2、compress 、uncompress、 zip、 unzip、rar、unrar 等。

### 打包和压缩和解压

Windows的压缩文件的扩展名 .zip/.rar
linux中的打包文件：aa.tar
linux中的压缩文件：bb.gz
linux中打包并压缩的文件：.tar.gz
Linux中的打包文件一般是以.tar结尾的，压缩的命令一般是以.gz结尾的。
而一般情况下打包和压缩是一起进行的，打包并压缩后的文件的后缀名一般.tar.gz。

### tar

最常用的打包命令是 tar，使用 tar 程序打出来的包常称为 tar 包，tar 包文件的命令通常都是以 .tar 结尾的。生成 tar 包后，就可以用其它的程序来进行压缩了。

```shell
tar -cf all.tar *.jpg #将所有 .jpg 的文件打成一个名为 all.tar 的包。-c 是表示产生新的包，-f 指定包的文件名。
tar -rf all.tar *.gif #将所有 .gif 的文件增加到 all.tar 的包里面去，-r 是表示增加文件的意思。
tar -uf all.tar logo.gif #更新原来 tar 包 all.tar 中 logo.gif 文件，-u 是表示更新文件的意思。
tar -tf all.tar #列出 all.tar 包中所有文件，-t 是列出文件的意思。
tar -xf all.tar #解出 all.tar 包中所有文件，-x 是解开的意思。
tar -czf all.tar.gz *.jpg #将所有 .jpg 的文件打成一个 tar 包，并且将其用 gzip 压缩，生成一个 gzip 压缩过的包，包名为 all.tar.gz。
tar -xzf all.tar.gz #将上面产生的包解开。
tar -cjf all.tar.bz2 *.jpg #将所有 .jpg 的文件打成一个 tar 包，并且将其用 bzip2 压缩，生成一个 bzip2 压缩过的包，包名为 all.tar.bz2
tar -xjf all.tar.bz2 #将上面产生的包解开。
tar -cZf all.tar.Z *.jpg #将所有 .jpg 的文件打成一个 tar 包，并且将其用 compress 压缩，生成一个 uncompress 压缩过的包，包名为 all.tar.Z。
tar -xZf all.tar.Z #将上面产生的包解开。
```

```shell
#对于.tar结尾的文件
tar -xf all.tar
#对于.gz结尾的文件
gzip -d all.gz
gunzip all.gz
#对于.tgz或.tar.gz结尾的文件
tar -xzf all.tar.gz
tar -xzf all.tgz
#对于.bz2结尾的文件
bzip2 -d all.bz2
bunzip2 all.bz2
#对于tar.bz2结尾的文件
tar -xjf all.tar.bz2
#对于.Z结尾的文件
uncompress all.Z
#对于.tar.Z结尾的文件
tar -xZf all.tar.z
#对于.zip
zip all.zip *.jpg
unzip all.zip
#对于.rar
# tar -xzpvf rarlinux-x64-5.6.b5.tar.gz # 安装 RAR for Linux
# cd rar 
# make
rar a all *.jpg
unrar e all.rar
```

### 扩展内容

**tar**

-c: 建立压缩档案 
-x：解压 
-t：查看内容 shell
-r：向压缩归档文件末尾追加文件 
-u：更新原压缩包中的文件

这五个是独立的命令，压缩解压都要用到其中一个，可以和别的命令连用但只能用其中一个。下面的参数是根据需要在压缩或解压档案时可选的。

-z：有gzip属性的 
-j：有bz2属性的 
-Z：有compress属性的 
-v：显示所有过程 
-O：将文件解开到标准输出 

下面的参数 -f 是必须的:

-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。 

**压缩**

```shell
tar –cvf jpg.tar *.jpg       # 将目录里所有jpg文件打包成 tar.jpg 
tar –czf jpg.tar.gz *.jpg    # 将目录里所有jpg文件打包成 jpg.tar 后，并且将其用 gzip 压缩，生成一个 gzip 压缩过的包，命名为 jpg.tar.gz 
tar –cjf jpg.tar.bz2 *.jpg   # 将目录里所有jpg文件打包成 jpg.tar 后，并且将其用 bzip2 压缩，生成一个 bzip2 压缩过的包，命名为jpg.tar.bz2 
tar –cZf jpg.tar.Z *.jpg     # 将目录里所有 jpg 文件打包成 jpg.tar 后，并且将其用 compress 压缩，生成一个 umcompress 压缩过的包，命名为jpg.tar.Z 
rar a jpg.rar *.jpg          # rar格式的压缩，需要先下载 rar for linux 
zip jpg.zip *.jpg            # zip格式的压缩，需要先下载 zip for linux
```

**解压**

```shell
tar –xvf file.tar         # 解压 tar 包 
tar -xzvf file.tar.gz     # 解压 tar.gz 
tar -xjvf file.tar.bz2    # 解压 tar.bz2 
tar –xZvf file.tar.Z      # 解压 tar.Z 
unrar e file.rar          # 解压 rar 
unzip file.zip            # 解压 zip 
```

1、* .tar 用 tar –xvf 解压 
2、* .gz 用 gzip -d或者gunzip 解压 
3、* .tar.gz和* .tgz 用 tar –xzf 解压 
4、* .bz2 用 bzip2 -d或者用bunzip2 解压 
5、* .tar.bz2用tar –xjf 解压 
6、* .Z 用 uncompress 解压 
7、* .tar.Z 用tar –xZf 解压 
8、* .rar 用 unrar e解压 
9、* .zip 用 unzip 解压

## 查找命令

### grep

```shell
ps -ef | grep sshd  #查找指定ssh服务进程 
ps -ef | grep sshd | grep -v grep #查找指定服务进程，排除gerp身 
ps -ef | grep sshd -c #查找指定进程个数 
grep "被查找的字符串" 文件名 #从文件内容查找匹配指定字符串的行
grep "thermcontact" /.in #在当前目录里第一级文件夹中寻找包含指定字符串的 .in 文件
grep –e "正则表达式" 文件名 #从文件内容查找与正则表达式匹配的行
grep –i "被查找的字符串" 文件名 #查找时不区分大小写
grep -c “被查找的字符串” 文件名 #查找匹配的行数
grep –v "被查找的字符串" 文件名 #从文件内容查找不匹配指定字符串的行
```

### find

find命令在目录结构中搜索文件，并对搜索结果执行指定的操作。

find 默认搜索当前目录及其子目录，并且不过滤任何结果（也就是返回所有文件），将它们全都显示在屏幕上。

```shell
find . -name "*.log" -ls  #在当前目录查找以.log结尾的文件，并显示详细信息。 
find /root/ -perm 600   #查找/root/目录下权限为600的文件 
find . -type f -name "*.log"  #查找当目录，以.log结尾的普通文件 
find . -type d | sort   #查找当前所有目录并排序 
find . -size +100M  #查找当前目录大于100M的文件
find / -type f -name "*.log" | xargs grep "ERROR" #从根目录开始查找所有扩展名为 .log 的文本文件，并找出包含 “ERROR” 的行
find . -name "*.in" | xargs grep "thermcontact" #从当前目录开始查找所有扩展名为 .in 的文本文件，并找出包含 “thermcontact” 的行
```

### locate

locate 让使用者可以很快速的搜寻某个路径。默认每天自动更新一次，所以使用locate 命令查不到最新变动过的文件。为了避免这种情况，可以在使用locate之前，先使用updatedb命令，手动更新数据库。如果数据库中没有查询的数据，则会报出locate: can not stat () `/var/lib/mlocate/mlocate.db’: No such file or directory该错误！updatedb即可！

yum -y install mlocate 如果是精简版CentOS系统需要安装locate命令

```shell
updatedb
locate /etc/sh #搜索etc目录下所有以sh开头的文件 
locate pwd #查找和pwd相关的所有文件
```

### whereis

whereis命令是定位可执行文件、源代码文件、帮助文件在文件系统中的位置。这些文件的属性应属于原始代码，二进制文件，或是帮助文件。

```shell
whereis ls    #将和ls文件相关的文件都查找出来
```

### which

which命令的作用是在PATH变量指定的路径中，搜索某个系统命令的位置，并且返回第一个搜索结果。

```shell
which pwd   #查找pwd命令所在路径 
which java  #查找path中java的路径
```

## su、sudo

### su

Linux su命令用于变更为其他使用者的身份，除 root 外，需要键入该使用者的密码。

su [-fmp] [-c command] [-s shell] [--help] [--version] [-] [USER [ARG]] 

- -f 或 --fast 不必读启动档（如 csh.cshrc 等），仅用于 csh 或 tcsh
- -m -p 或 --preserve-environment 执行 su 时不改变环境变数
- -c command 或 --command=command 变更为帐号为 USER 的使用者并执行指令（command）后再变回原来使用者
- -s shell 或 --shell=shell 指定要执行的 shell （bash csh tcsh 等），预设值为 /etc/passwd 内的该使用者（USER） shell
- –help 显示说明文件
- –version 显示版本资讯

- -l 或 --login 这个参数加了之后，就好像是重新 login 为该使用者一样，大部份环境变数（HOME SHELL USER等等）都是以该使用者（USER）为主，并且工作目录也会改变，如果没有指定 USER ，内定是 root
- USER 欲变更的使用者帐号
- ARG 传入新的 shell 参数

su用于用户之间的切换。但是切换前的用户依然保持登录状态。如果是root 向普通或虚拟用户切换不需要密码，反之普通用户切换到其它任何用户都需要密码验证。

### sudo

sudo是为所有想使用root权限的普通用户设计的。可以让普通用户具有临时使用root权限的权利。只需输入自己账户的密码即可。

进入sudo配置文件命令：vi /etc/sudoer或者visudo

## 下载与安装yum

yum提供了查找、安装、删除某一个、一组甚至全部软件包的命令，而且命令简洁而又好记。

yum [options] [command] [package ...]

- options：可选，选项包括-h（帮助），-y（当安装过程提示选择全部为"yes"），-q（不显示安装的过程）等等。
- command：要进行的操作。
- package操作的对象。

yum常用命令

1. 列出所有可更新的软件清单命令：yum check-update
2. 更新所有软件命令：yum update
3. 仅安装指定的软件命令：yum install <package_name>
4. 仅更新指定的软件命令：yum update <package_name>
5. 列出所有可安裝的软件清单命令：yum list
6. 删除软件包命令：yum remove <package_name>
7. 查找软件包 命令：yum search
8. 清除缓存命令:
   - yum clean packages: 清除缓存目录下的软件包
   - yum clean headers: 清除缓存目录下的 headers
   - yum clean oldheaders: 清除缓存目录下旧的 headers
   - yum clean, yum clean all (= yum clean packages; yum clean oldheaders) :清除缓存目录下的软件包及旧的headers

## Linux三剑客-awk、sed、grep

- grep 更适合单纯的查找或匹配文本
- sed 更适合编辑匹配到的文本
- awk 更适合格式化文本，对文本进行较复杂格式处理

### grep

Linux grep 命令用于查找文件里符合条件的字符串。

grep 指令用于查找内容包含指定的范本样式的文件，如果发现某文件的内容符合所指定的范本样式，预设 grep 指令会把含有范本样式的那一列显示出来。若不指定任何文件名称，或是所给予的文件名为 -，则 grep 指令会从标准输入设备读取数据。

grep [-abcEFGhHilLnqrsvVwxy] [-A<显示列数>] [-B<显示列数>] [-C<显示列数>] [-d<进行动作>] [-e<范本样式>] [-f<范本文件>] [--help] [范本样式] [文件或目录...]

- -a 或 --text : 不要忽略二进制的数据。
- -A<显示行数> 或 --after-context=<显示行数> : 除了显示符合范本样式的那一列之外，并显示该行之后的内容。
- -b 或 --byte-offset : 在显示符合样式的那一行之前，标示出该行第一个字符的编号。
- -B<显示行数> 或 --before-context=<显示行数> : 除了显示符合样式的那一行之外，并显示该行之前的内容。
- -c 或 --count : 计算符合样式的列数。
- -C<显示行数> 或 --context=<显示行数>或-<显示行数> : 除了显示符合样式的那一行之外，并显示该行之前后的内容。
- -d <动作> 或 --directories=<动作> : 当指定要查找的是目录而非文件时，必须使用这项参数，否则grep指令将回报信息并停止动作。
- -e<范本样式> 或 --regexp=<范本样式> : 指定字符串做为查找文件内容的样式。
- -E 或 --extended-regexp : 将样式为延伸的正则表达式来使用。
- -f<规则文件> 或 --file=<规则文件> : 指定规则文件，其内容含有一个或多个规则样式，让grep查找符合规则条件的文件内容，格式为每行一个规则样式。
- -F 或 --fixed-regexp : 将样式视为固定字符串的列表。
- -G 或 --basic-regexp : 将样式视为普通的表示法来使用。
- -h 或 --no-filename : 在显示符合样式的那一行之前，不标示该行所属的文件名称。
- -H 或 --with-filename : 在显示符合样式的那一行之前，表示该行所属的文件名称。
- -i 或 --ignore-case : 忽略字符大小写的差别。
- -l 或 --file-with-matches : 列出文件内容符合指定的样式的文件名称。
- -L 或 --files-without-match : 列出文件内容不符合指定的样式的文件名称。
- -n 或 --line-number : 在显示符合样式的那一行之前，标示出该行的列数编号。
- -o 或 --only-matching : 只显示匹配PATTERN 部分。
- -q 或 --quiet或–silent : 不显示任何信息。
- -r 或 --recursive : 此参数的效果和指定"-d recurse"参数相同。
- -s 或 --no-messages : 不显示错误信息。
- -v 或 --revert-match : 显示不包含匹配文本的所有行。
- -V 或 --version : 显示版本信息。
- -w 或 --word-regexp : 只显示全字符合的列。
- -x --line-regexp : 只显示全列符合的列。
- -y : 此参数的效果和指定"-i"参数相同。

```shell
grep test *file #在当前目录中，查找后缀有 file 字样的文件中包含 test 字符串的文件，并打印出该字符串的行
grep test test* #查找前缀有“test”的文件包含“test”字符串的文件  
grep -r update /etc/acpi #找指定目录/etc/acpi 及其子目录（如果存在子目录的话）下所有文件中包含字符串"update"的文件，并打印出该字符串所在行的内容
grep -v test *test* #查找文件名中包含 test 的文件中不包含test 的行
```

### sed

Linux sed 命令是利用脚本来处理文本文件。

sed 可依照脚本的指令来处理、编辑文本文件。

Sed 主要用来自动编辑一个或多个文件、简化对文件的反复操作、编写转换程序等。

sed [-hnv] [-e< script >] [-f<script文件>] [文本文件]

-e< script >或--expression=< script > 以选项中指定的script来处理输入的文本文件。
-f<script文件>或–file=<script文件> 以选项中指定的script文件来处理输入的文本文件。
-h或–help 显示帮助。
-n或–quiet或–silent 仅显示script处理后的结果。
-V或–version 显示版本信息。

a ：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)～
c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
d ：删除，因为是删除啊，所以 d 后面通常不接任何咚咚；
i ：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
p ：打印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
s ：取代，可以直接进行取代的工作哩！通常这个 s 的动作可以搭配正规表示法

### awk

awk [选项参数] 'script' var=value file(s) 或 

awk [选项参数] -f scriptfile var=value file(s)

```shell
-F fs or --field-separator fs
#指定输入文件折分隔符，fs是一个字符串或者是一个正则表达式，如-F:。
-v var=value or --asign var=value
#赋值一个用户定义变量。
-f scripfile or --file scriptfile
#从脚本文件中读取awk命令。
-mf nnn and -mr nnn
#对nnn值设置内在限制，-mf选项限制分配给nnn的最大块数目；-mr选项限制记录的最大数目。这两个功能是Bell实验室版awk的扩展功能，在标准awk中不适用。
-W compact or --compat, -W traditional or --traditional
#在兼容模式下运行awk。所以gawk的行为和标准的awk完全一样，所有的awk扩展都被忽略。
-W copyleft or --copyleft, -W copyright or --copyright
#打印简短的版权信息。
-W help or --help, -W usage or --usage
#打印全部awk选项和每个选项的简短说明。
-W lint or --lint
#打印不能向传统unix平台移植的结构的警告。
-W lint-old or --lint-old
#打印关于不能向传统unix平台移植的结构的警告。
-W posix
#打开兼容模式。但有以下限制，不识别：/x、函数关键字、func、换码序列以及当fs是一个空格时，将新行作为一个域分隔符；操作符** 和 **=不能代替^ 和 ^=；fflush无效。
-W re-interval or --re-inerval
#允许间隔正则表达式的使用，参考(grep中的Posix字符类)，如括号表达式[[:alpha:]]。
-W source program-text or --source program-text
#使用program-text作为源代码，可与-f命令混用。
-W version or --version
#打印bug报告信息的版本。
```

