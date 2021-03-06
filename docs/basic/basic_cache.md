# 基础Cache

## 类 sram 接口

在实现基础 cache 时，cache 与数据通路和转接桥之间都通过类 sram 接口进行通信，所以我们需要足够了解类 sram 接口的工作方式  
类 sram 接口包括以下信号:  

| 信号      | 位宽     | 方向            | 功能                                    |
| ------- | ------ | ------------- | ------------------------------------- |
| clk     | 1      | master->slave | 时钟信号                                  |
| req     | 1      | master->slave | 请求信号，置1时表示有读写请求                       |
| wr      | 1      | master->slave | 写使能信号，置1时表示该次请求为写                     |
| size    | [1:0]  | master->slave | 该次请求传输的字节数，0：1byte，1：2bytes， 2：4bytes |
| addr    | [31:0] | master->slave | 该次请求的地址                               |
| wdata   | [31:0] | master->slave | 该次请求写的数据                              |
| addr_ok | 1      | slave->master | 表示该次请求的地址传输 OK，读：地址传输完成，写：地址和数据均传输完成  |
| data_ok | 1      | slave->master | 该次请求的数据 OK，读：希望读的数据返回，写：数据写入完成        |
| rdata   | [31:0] | slave->master | 该次请求返回的数据                             |

类 sram 接口一次请求的大致过程如下：

* master 端拉高 `req` 信号并给出该次请求的各项参数（读或写，请求的地址，请求的字节数，请求写的数据等等）

* slave 端存储 master 端给出的请求参数后拉高 `addr_ok` （在 `addr_ok` 拉高后 master 可能不再保持请求各项参数有效）

* slave 处理请求，这可能需要数个周期（存储器的时钟速度往往慢于 CPU 故读写，可能需要多个周期完成读写）

* slave 处理完成，将请求处理结果返回，并拉高 `data_ok` （读：需要在 `rdata` 上给出读取的数据，写：仅拉高 `data_ok` 即可）

注意 sram 接口的地址信号（addr）以字节寻址，其指向读写数据的最低有效位。我们需要将 addr 和 size 信号配合起来使用：  

* `size`==0 读写单字节，`addr` 最低两位可以是任意值

* `size`==1 读写半字，`addr` 最低位需要为 0，否则发生地址错误异常

* `size`==2 读写一个字，`addr` 最低两位需要为 0，否则发生地址错误异常

下表列出了在各种有效的 addr 和 size 组合下，有效数据出现的位置

|                                    | `data[31:24]` | `data[23:16]` | `data[15:8]` | `data[7:0]` |
| ---------------------------------- | ------------- | ------------- | ------------ | ----------- |
| `size`==2'b00, `addr[1:0]`==2'b00  | -             | -             | -            | valid       |
| `size`==2'b00, `addr[1:0]`==2'b01  | -             | -             | valid        | -           |
| `size`==2'b00, `addr[1:0]`==2'b10  | -             | valid         | -            | -           |
| `size`==2'b00, `addr[1:0]`==2'b11  | valid         | -             | -            | -           |
| `size`==2'b01, `addr[1:0]`==2'b00  | -             | -             | valid        | valid       |
| `size`==2'b01,  `addr[1:0]`==2'b10 | valid         | valid         | -            | -           |
| `size`==2'b10, `addr[1:0]`==2'b00  | valid         | valid         | valid        | valid       |

## 类 sram 接口的状态机设计

在了解了类 sram 接口的工作机制之后，我们可以基于此设计一个状态机来控制类 sram 接口的数据传输，设计状态机需要关注状态机可能处于的状态和促使状态改变的标志信号或者标志信号组合   
我们首先来看看类 sram 接口可能处于的状态：  

* `IDLE`：等待请求到达

* `ADDR`：传输包括 addr 在内的请求参数

* `DATA`：传输数据

接下来我们需要决定类 sram 接口是如何在状态之间转移的

* 显然，当 master 端拉高 `req` 信号时，请求被发起，在 `req` 信号为高的第一个时钟上升沿状态机应当从 `IDLE` 转到 `ADDR`

* 当请求的参数被接受时，`addr_ok` 拉高，在此后的第一个时钟上升沿，状态机应当转到 `DATA`

* 当数据传输完成，`data_ok` 拉高，此后的第一个时钟上升沿，状态机回到 `IDLE`，等待下一个请求

以上给出的是一个比较简单但是可行的例子，我们也可以添加更多状态以实现对类 sram 接口更加精细的控制

状态机的存在使得我们的模块能够知道请求当前的状态，我们需要依据状态机的当前状态来生成其余的控制信号

值得注意的是，在类 sram 接口完成请求参数传输并拉高 `addr_ok` 后， **如果继续拉高 `req` 信号，会被认为是发出了一个新的请求** ，这可能导致一系列的 bug  
比如，当 I-cache 取指完成，但 EXE 阶段仍有一条乘法指令造成流水线阻塞，此时如果继续拉高 req 信号，类 sram 接口将会开始一个新的请求处理过程，如果在类 sram 接口完成请求处理之前乘法结束，流水线继续流动，那么类 sram 接口返回的指令将会在之后的某个周期被读取，于是错误的取指便发生了

```
t           ------------------------------------------------------------->

sram-l      <--------req process-------><--------req process----->
         开始取指                    取指完成 此时错误发起了新的取指
mul                <----------executing mul--------->
              开始执行乘法                       乘法执行结束
stall-if    <----------------------high--------------------------><-low-->

req         <----------------------high--------------------------><-low--> 
       
```



另外，我们需要考虑流水线因为分支预测错误（在使用了分支预测的情况下）和发生异常而清空的情况，如果在发生流水线清空的时候取指阶段的类 sram 读请求尚未完成，而类 sram 接口并没有取消请求的机制，那么我们就需要忽略该请求返回的结果，以防止后续的指令取指错误  
这个问题的另一种解决方案是在取指的读类 sram 接口请求尚未完成时，暂停全部流水级，这种解决方案对性能的影响较小（无论是否暂停全部流水级，进入流水线的指令数都没有发生改变）同时也能够简化设计

## 写回 cache

不同于写直达 cache，写回 cache 在接受写请求时将数据先写入 cache，并将该 cache 块标记为 dirty，当之后的某次请求需要覆盖该 cache 块时，再将脏块写回到内存中  



我们需要注意，由于类 sram 接口支持字节和半字的读写，在向一个空 cache 块执行写半字或者写字节操作之前，我们需要提前将写操作地址对应的整个字先读入 cache，此后再将新数据写入该 cache 块，并将 cahce 块标记为 dirty  
假设我们希望写入的是某个字的最低的字节，将该字表示为 `XXXXXXXX`，（`X`可能是任何十六进制数），如果我们不先将写操作对应的字读入，那么在我们向 cache 中写入数据（`AA`）后，该 cache 块为 `XXXXXXAA`，在后续写回该块时，我们会将高位的所有 `X` 写入内存中对应的位置，从而覆盖正确的数据，这显然是不可接受的  
一般而言，如果需要对空 cache 块写入的数据大小 **小于** 一个 cache 块，我们都需要事先从内存中读入待写数据对应的块。在使用块单字时，如果写入一整个字，则不需要从内存中读入对应的块，但在使用块多字时，由于一次最多写一个字，故任何对空 cache 块的写操作都需要先把对应的 cache 块从内存中读出  
为了简化设计，我们可以对任何针对空 cache 块的写操作都要求事先读入对应的块，这样会损失一定的性能，但是有助于简化写回 cache 的状态机设计

## 写回 cache 的状态机设计

在了解了写回 cache 的基本工作流程后，我们可以设计出相应的状态机，写回 cache 可能处于以下几个状态：

* `IDLE`：等待请求

* `WRIBK`：正在向内存写回数据

* `READIN`：正在从内存读入数据

* `FIN：cache` 读写完成  
  
  * 对于读 cache 请求，进入此状态待读数据出现在 cache 与数据通路交互的接口上
  
  * 对于写 cache 请求，进入此状态后的第一个时钟上升沿将数据写入 cache 块相应的位置

写回 cache 的状态转移流程：

* 初始为 `IDLE`

* 在来自数据通路的请求到达时，首先判断是否命中  
  
  * 如果命中，则直接执行 cache 内的读写操作，保持 `IDLE` 状态
  
  * 如果缺失，则判断需要覆盖的 cache 块是否有效且包含脏数据  
    
    * 如果待覆盖的 cache 块无效或不包含脏数据，则转入 `READIN` 状态，从内存中读入数据
    
    * 如果待覆盖的 cache 块有效且包含脏数据，则转入 `WRBK` 状态，向内存写回数据，在完成写回后，转入 `READIN` 状态，从内存中读入数据
    
    * 在完成写回和读入后，进入 `FIN` 状态，完成 cache 读写，最后回到 IDLE 状态

## 