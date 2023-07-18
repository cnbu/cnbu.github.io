步骤1：新document首先写入内存Buffer缓存中。

步骤2：每隔一段时间，执行“commitpoint”操作，buffer写入新Segment中。

步骤3：新segment写入文件系统缓存 filesystem cache。

步骤4：文件系统缓存中的index segment被fsync强制刷到磁盘上，确保物理写入。

此时，新egment被打开供search操作。

步骤5：清空内存buffer，可以接收新的文档写入


实际ES为保证实时性，会做refresh操作


1、当新的文档写入后，写入 index buffer的同时会写入translog。
2、refresh操作使得写入文档搜索可见；
3、flush操作使得filesystem cache写入磁盘，以达到持久化的目的
