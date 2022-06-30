---
title: find命令
date: 2019-04-16 10:57:12
tags: 
 - java
categories: 
 - java
---
```
将当前目录及其子目录下所有文件后缀为 .c 的文件列出来:
# find . -name "*.c"
将当前目录及其子目录中的所有文件列出：

# find . -type f
将当前目录及其子目录下所有最近 20 天内更新过的文件列出:

# find . -ctime -20
查找 /var/log 目录中更改时间在 7 日以前的普通文件，并在删除之前询问它们：

# find /var/log -type f -mtime +7 -ok rm {} \;
查找当前目录中文件属主具有读、写权限，并且文件所属组的用户和其他用户具有读权限的文件：

# find . -type f -perm 644 -exec ls -l {} \;
查找系统中所有文件长度为 0 的普通文件，并列出它们的完整路径：

# find / -type f -size 0 -exec ls -l {} \;
```