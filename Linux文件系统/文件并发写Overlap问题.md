**问题：Linux 中 write 系统调用具有原子性吗？比如两个进程同时 write 同一个文件会不会出现数据交错的情况？标准里面有规定吗？**

* 知乎上的回答：https://www.zhihu.com/question/20906432/answer/1687086286

  两个进程/线程同时write同一个文件要分两种情况，一是“文件“被共享， 二是不被共享。

  首先澄清一下概念，这里的“文件”指的是Linux 内核 in memory 文件表（file table)里的一个文件对象，即 file description。

  第一种情况，会在`dup`或者`fork`时发生。这时有多个 fd 同时指向“文件”，两个进程/线程同时调用write，即使不同的fd，可“文件”还是同一个，write操作的两个步骤（1. lseek 找写入文件位置；2. vfs_write 写入内容；）会被 file description 里的 `struct mutex  f_pos_lock` 加锁保护，保证其原子性，不会出现文件内容的overlap。

  第二种情况，一个文件被open多次，file table 里有对应的多个文件对象，有单独的文件 position 和 f_*pos*_lock，此时同时write，会出现内容overlap的情况。

  不过，第二种情况可以使用 open + O_APPEND 模式，保证文件内容不被overlap。O_APPEND 模式通过inode里的锁机制保证并发原子的。



JDK在打开FileOutStream时的模式选择：

```C++
private native void open0(String name, boolean append)
    throws FileNotFoundException;
// jdk/src/solaris/native/java/io/FileOutputStream_md.c
JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_open(JNIEnv *env, jobject this,
                                   jstring path, jboolean append) {
    // 使用O_WRONLY，O_CREAT模式打开文件，如果文件不存在会新建文件
    // 如果java中指定append参数为true，则使用O_APPEND追加模式
    // 如果java中指定append参数为false，则使用O_TRUNC模式，如果文件存在内容，会清空掉
    fileOpen(env, this, path, fos_fd,
             O_WRONLY | O_CREAT | (append ? O_APPEND : O_TRUNC));
}
```



:question:但是，模拟多线程并发写相同文件，不管（1）是否使用append；（2）是否使用一个或两个FileOutputStream对象（3）每次写入有无flush，都没有出现overlap现象。难道JDK同一个进程内还有文件锁？还是测试写入的长度太小了？

JDK关于文件读写的native实现可以参考：https://www.cnblogs.com/yungyu16/p/13053986.html