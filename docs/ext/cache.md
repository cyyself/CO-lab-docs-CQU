# Cache

## 多路组相联 cache

多路 cache 在状态机设计上基本没有区别，但是多路 cache 内部存储数据的方式与直接映射 cache 有所不同，内存中的块在 cache 中对应多个位置，但 cache 最多存储内存中数据的一份有效镜像

```
              way 1       way 2       way3        way4
        +-+-+-+-----+-+-+-+-----+-+-+-+-----+-+-+-+-----+
set1    |D|V|T|     |D|V|T|     |D|V|T|     |D|V|T|     |
        +-+-+-+-----+-+-+-+-----+-+-+-+-----+-+-+-+-----+
set2    |D|V|T|     |D|V|T|     |D|V|T|     |D|V|T|     |
        +-+-+-+-----+-+-+-+-----+-+-+-+-----+-+-+-+-----+
set3    |D|V|T|     |D|V|T|     |D|V|T|     |D|V|T|     |
        +-+-+-+-----+-+-+-+-----+-+-+-+-----+-+-+-+-----+
set4    |D|V|T|     |D|V|T|     |D|V|T|     |D|V|T|     |
        +-+-+-+-----+-+-+-+-----+-+-+-+-----+-+-+-+-----+
```



在处理多路 cache 读写请求时，我们需要同时读取对应 cache set 中所有 way 的 `valid`，`dirty`，和 `tag` 字段，并将请求的地址的 `tag` 段与所有 `valid` 为1的 cache way 的 `tag` 进行比对，如果有相等者则说明 cache 命中，反之则说明 cache 缺失，需要将对应的块从内存中读入  
在将缺失块从内存中换入时，优先选择无效的 way，如果没有无效的 way，则需要通过特定的替换算法进行替换，这里我们推荐伪 LRU 算法，可以通过比较简单的机制达到较好的效果（参考《计算机系统结构》实验指导书）  
在进行块的覆盖时，如果待覆盖的块为脏，则需要执行写回操作



注意，我们需要对 cache 的各个 way 进行并行读写，用一个周期完成对应 cache set 所有 way 的读写

### PLRU

（未完待续）

## 块多字

（未完待续）

### AXI Burst

（未完待续）
