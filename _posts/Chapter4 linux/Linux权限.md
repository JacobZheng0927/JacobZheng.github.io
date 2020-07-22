[TOC]

# Linux权限

## 前言

为了学习hdfs权限系统，先学习一下linux的权限



## 权限简介

Linux下文件的权限类型一般包括读，写，执行。对应字母为 r、w、x。

数字权限：

|      | Read  = 4 | Write = 2 | Excute  = 1 | Sum  |
| ---- | --------- | --------- | ----------- | ---- |
| rwx  | 1         | 1         | 1           | 7    |
| rw   | 1         | 1         | 0           | 6    |
| rx   | 1         | 0         | 1           | 5    |
| r    | 1         | 0         | 0           | 4    |
| wx   | 0         | 1         | 1           | 3    |
| w    | 0         | 1         | 0           | 2    |
| x    | 0         | 0         | 1           | 1    |

如果我们要表示一个文件的所有权限详情，有两种方式：

- 第一种是十位二进制表示法，(三个属组每个使用二进制位，再加一个最高位共十位)，可简化为三位八进制形式
- 另外一种十二位二进制表示法(十二个二进制位)，可简化为四位八进制形式



十位权限表示

常见的权限表示形式有：

```
-rw------- (600)  只有拥有者有读写权限。
-rw-r--r-- (644)  只有拥有者有读写权限；而属组用户和其他用户只有读权限。
-rwx------ (700)  只有拥有者有读、写、执行权限。
-rwxr-xr-x (755)  拥有者有读、写、执行权限；而属组用户和其他用户只有读、执行权限。
-rwx--x--x (711)  拥有者有读、写、执行权限；而属组用户和其他用户只有执行权限。
-rw-rw-rw- (666)  所有用户都有文件读、写权限。
-rwxrwxrwx (777)  所有用户都有读、写、执行权限。
```

每三位二进制用8进制标识，第一位的情况如下

```
d代表的是目录(directroy)
-代表的是文件(regular file)
s代表的是套字文件(socket)
p代表的管道文件(pipe)或命名管道文件(named pipe)
l代表的是符号链接文件(symbolic link)
b代表的是该文件是面向块的设备文件(block-oriented device file)
c代表的是该文件是面向字符的设备文件(charcter-oriented device file)
```

Linux下权限的粒度有 拥有者 、群组 、其它组 三种。每个文件都可以针对三个粒度，设置不同的rwx(读写执行)权限。通常情况下，一个文件只能归属于一个用户和组， 如果其它的用户想有这个文件的权限，则可以将该用户加入具备权限的群组，一个用户可以同时归属于多个组。



Linux 命令

**chmod [可选项] <mode> <file...>**

```
可选项：
  -c, --changes          like verbose but report only when a change is made (若该档案权限确实已经更改，才显示其更改动作)
  -f, --silent, --quiet  suppress most error messages  （若该档案权限无法被更改也不要显示错误讯息）
  -v, --verbose          output a diagnostic for every file processed（显示权限变更的详细资料）
       --no-preserve-root  do not treat '/' specially (the default)
       --preserve-root    fail to operate recursively on '/'
       --reference=RFILE  use RFILE's mode instead of MODE values
  -R, --recursive        change files and directories recursively （以递归的方式对目前目录下的所有档案与子目录进行相同的权限变更)
       --help		显示此帮助信息
       --version		显示版本信息
mode ：权限设定字串，详细格式如下：
[ugoa...][[+-=][rwxX]...][,...]，其中
[ugoa...]
u 表示该档案的拥有者，g 表示与该档案的拥有者属于同一个群体(group)者，o 表示其他以外的人，a 表示所有（包含上面三者）。
[+-=]
+ 表示增加权限，- 表示取消权限，= 表示唯一设定权限。
[rwxX]
r 表示可读取，w 表示可写入，x 表示可执行，X 表示只有当该档案是个子目录或者该档案已经被设定过为可执行。
```



**chown [可选项] user[:group] file...**

```
使用权限：root

说明：
[可选项] : 同上文chmod
user : 新的文件拥有者的使用者 
group : 新的文件拥有者的使用者群体(group)
```

EOF