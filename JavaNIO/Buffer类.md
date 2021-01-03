## 1.Buffer类指针

* Capacity

  Buffer容量，Buffer对象创建后，值不可被修改

* Position

  Buffer可读写下一个字节的索引位置，初始值为0

* Limit

  Buffer第一个不可读写的索引位置，初始值等于Capacity（limit处索引是不可读写的）

* Mark

  标记Position位置，方便重置Position索引值

Buffer索引值从0开始计算



## 2.Buffer操作公式

1. init(size)：capactity = size, position = 0, limit = capacity, mark未初始化
2. put()：position++
3. put(index, value)：position不变
4. put(array[], start, length)：array数据从start位置开始，拷贝length长度数据到buffer，position=position+length
5. put(array[])：等价于put(array[], 0, array.length)，如果buffer.remaining()<array.length，则触发overflow异常
6. put(srcBuffer)：如果buffer.remaining()<srcBuffer.remaining()，则触发overflow异常，position=position+srcBuffer.remaining()
7. flip()：limit=position, position=0，如果连续flip两次，position=limit=0，buffer不可读写
8. get()：position++
9. get(index, value)：position不变
10. get(array[], start, length)：buffer数据从start位置开始，拷贝length长度数据到外部数组
11. get(array[])：等价于get(array[], 0, array.length)，如果buffer.remaining()>array.length，则触发overflow异常
12. rewind()：position=0
13. compact()：limit=limit-position, position=0, 剩余数据拷贝到buffer开头
14. clear()：position=0, limit=capacity
15. mark()：mark=position
16. reset()：position=mark



## 3.视图Buffer

* duplicate()

  srcBuffer指针变量的完全拷贝

* asReadOnlyBuffer()

  视图Buffer，禁止put操作，其他指针类似duplicate

* slice()

  视图Buffer中，position=0，limit=srcBuffer.remaining()，capacity=srcBuffer.remaining()



## 4.DirectBuffer（直接缓存区）

使用堆外内存进行存储的Buffer，对jvm的gc性能影响不同，加上预分配技术，可以提升程序性能。