# HDFS 的组成架构

![image-20210524214928537](/Users/guowenfeng/Library/Application Support/typora-user-images/image-20210524214928537.png)

## NameNode

存储文件的元信息，包括名字、目录、大小等。

1. 管理 HDFS 的名称空间
2. 配置副本策略
3. 管理数据块（Block）映射信息
4. 处理客户端读写请求

## DataNode

主要用来存储数据，NameNode 下达命令，DataNode 执行实际的操作。

1. 存储实际的数据块
2. 执行数据块的读写操作

## Client

客户端

1. 文件切分，文件上传 HDFS 的时候，Client 将文件切分成一个一个的 Block，一般是 128M，然后进行上传
2. 与 NameNode 交互，获取文件的位置信息
3. 与 DataNode 交互，读取或写入数据
4. Client 提供一些命令来管理 HDFS
5. Client通过一些命令来访问 HDFS，eg：HDFS 的增删查改

## Secondary NameNode

并非 NameNode 的热备。当 NameNode 挂掉的时候，它并不能马上替换 NameNode 并提供服务

1. 辅助 NameNode，分担工作量。eg：定期合并 Fsimage(静态文件)和 Edits(编辑日志)，并推送给 NameNode
2. 在紧急情况下，可辅助恢复 NameNode