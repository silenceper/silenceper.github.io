# 用vim保存文件和echo命令到底有什么不同？



## 现象
最近在调试一个filebeat程序时需要制造一些log，我是直接使用vim直接对文件打开然后直接保存的。

但是有个**奇怪的现象**：每次写入一行新的日志，filebeat都会将整个文件的内容又重新进行上报一遍导致日志上传重复，同时观察到filebeat的文件采集状态文件`registry`都会进行增加一个重复的文件(source相同，inode不同)。

操作一番后以为是程序的问题，最后才反应过来（一开始没注意到inode不同 :<），最终使用`echo "test some log">>test.log`追加的方式发现不会有此问题。

同时看了下两者的`inode`信息，确认了是因为`vim`会改变文件的inode(一个新的文件)，echo则不会：
> 2020-05-14更新：也跟文件权限有关，如果文件没有o没有w权限，则inode信息会改变，如果`chmod o+w test.log`则inode信息不会变

**1、查看文件inode为908488**

```shell
[root@localhost work]# stat test.log
  文件："test.log"
  大小：0         	块：0          IO 块：4096   普通空文件
设备：fd00h/64768d	Inode：908488      硬链接：1
权限：(0644/-rw-r--r--)  Uid：(    0/    root)   Gid：(    0/    root)
环境：unconfined_u:object_r:admin_home_t:s0
最近访问：2020-05-10 04:38:39.399791808 -0400
最近更改：2020-05-10 04:38:39.399791808 -0400
最近改动：2020-05-10 04:38:39.399791808 -0400
创建时间：-
```

**2、当我用vim/vi工具进行编辑保存之后，变为了：908490**

```
[root@localhost work]# vi test.log
[root@localhost work]# stat test.log
  文件："test.log"
  大小：7         	块：8          IO 块：4096   普通文件
设备：fd00h/64768d	Inode：908490      硬链接：1
权限：(0644/-rw-r--r--)  Uid：(    0/    root)   Gid：(    0/    root)
环境：unconfined_u:object_r:admin_home_t:s0
最近访问：2020-05-10 04:39:01.950055784 -0400
最近更改：2020-05-10 04:39:01.950055784 -0400
最近改动：2020-05-10 04:39:01.953305882 -0400
创建时间：-
```

**3、而用echo命令修改看到inode信息并没有改变, 908490**

```
[root@localhost work]# echo "test some log">>test.log
[root@localhost work]# stat test.log
  文件："test.log"
  大小：21        	块：8          IO 块：4096   普通文件
设备：fd00h/64768d	Inode：908490      硬链接：1
权限：(0644/-rw-r--r--)  Uid：(    0/    root)   Gid：(    0/    root)
环境：unconfined_u:object_r:admin_home_t:s0
最近访问：2020-05-10 04:39:01.950055784 -0400
最近更改：2020-05-10 04:41:16.794459151 -0400
最近改动：2020-05-10 04:41:16.794459151 -0400
创建时间：-
```

## 验证

其实通过inode的改变以及可以猜测出在使用vi/vim进行编辑的时候，文件以及变为了一个新的文件，这里我主要通过`inotifywait`来看一下，在进行vim编辑的时候，文件会产生什么时间
> 一开始本来打算使用golang中`github.com/fsnotify/fsnotify`去看的，但是发现包中对event进行了合并，不够原始。所以才直接使用`inotifywait `来看


inotifywait命令：
```
inotifywait -rm test.log
```

文件权限：

```
-rw-r--r--. 1 root root   49 5月  14 05:40 test.log
```

1、使用vim进行编辑的结果：

```
[root@localhost work]# inotifywait -rm test.log
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
test.log OPEN
test.log ACCESS
test.log CLOSE_NOWRITE,CLOSE
test.log MOVE_SELF
test.log ATTRIB
test.log DELETE_SELF
```
可以看到对文件进行了`DELETE_SELF`

**注意当再次进行编辑的时候，已经无法监听到了，因为原有文件已经被删除，需要重新监听**

2、使用echo进行追加写入的结果

```
[root@localhost work]# inotifywait -rm test.log
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
test.log OPEN
test.log MODIFY
test.log CLOSE_WRITE,CLOSE
```

3、监听整个目录，可以看到更多信息

```
[root@localhost work]# inotifywait -rm ./dir/
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.

# 当进行文件写入时
./dir/ OPEN test.log
./dir/ CREATE .test.log.swp
./dir/ OPEN .test.log.swp
./dir/ CREATE .test.log.swx
./dir/ OPEN .test.log.swx
./dir/ CLOSE_WRITE,CLOSE .test.log.swx
./dir/ DELETE .test.log.swx
./dir/ CLOSE_WRITE,CLOSE .test.log.swp
./dir/ DELETE .test.log.swp
./dir/ CREATE .test.log.swp
./dir/ OPEN .test.log.swp
./dir/ MODIFY .test.log.swp
./dir/ ATTRIB .test.log.swp
./dir/ ACCESS test.log
./dir/ CLOSE_NOWRITE,CLOSE test.log
./dir/ MODIFY .test.log.swp

# 当进行文件保存是

./dir/ CREATE 4913
./dir/ OPEN 4913
./dir/ ATTRIB 4913
./dir/ CLOSE_WRITE,CLOSE 4913
./dir/ DELETE 4913
./dir/ MOVED_FROM test.log
./dir/ MOVED_TO test.log~
./dir/ CREATE test.log
./dir/ OPEN test.log
./dir/ MODIFY test.log
./dir/ CLOSE_WRITE,CLOSE test.log
./dir/ ATTRIB test.log
./dir/ MODIFY .test.log.swp
./dir/ DELETE test.log~
./dir/ CLOSE_WRITE,CLOSE .test.log.swp
./dir/ DELETE .test.log.swp
```

可以看到在进行文件保存时，将原有test.log 移动为test.log~，并新创建一个文件test.log，最后对test.log~备份文件和.test.log.swp缓冲区文件进行删除

4、当进行`chmod o+w test.log`时，看到的信息如下：

```
./dir/ CREATE 4913
./dir/ OPEN 4913
./dir/ ATTRIB 4913
./dir/ CLOSE_WRITE,CLOSE 4913
./dir/ DELETE 4913
./dir/ OPEN test.log
./dir/ CREATE test.log~
./dir/ OPEN test.log~
./dir/ ATTRIB test.log~
./dir/ ACCESS test.log
./dir/ MODIFY test.log~
./dir/ CLOSE_WRITE,CLOSE test.log~
./dir/ ATTRIB test.log~
./dir/ CLOSE_NOWRITE,CLOSE test.log
./dir/ MODIFY test.log
./dir/ OPEN test.log
./dir/ MODIFY test.log
./dir/ CLOSE_WRITE,CLOSE test.log
./dir/ ATTRIB test.log
./dir/ MODIFY .test.log.swp
./dir/ DELETE test.log~
./dir/ CLOSE_WRITE,CLOSE .test.log.swp
./dir/ DELETE .test.log.swp
```
对比3中的结果没有将test.log move to test.log~的操作，而是直接对test.log文件进行更改，所以此时inode信息并没有改变

## 总结

- vim编辑是产生了一个新的文件，所以看到inode信息变化了，而echo仅仅只是对文件进行写入。
- 在vim编辑过程中，会生成swp缓冲区文件，在进行vim编辑时时刻将内容保存进入swp，防止程序退出未保存
- 产生后缀带 ~ 的备份文件，这个是在对文件进行保存时产生，如果文件被正常保存则该文件会被立即删除，正常情况下是看不到这个文件
- 如果文件other有write权限（`chmod o+w xxx`），则当进行vi写入内容的时候文件inode不会改变。

