---
dg-publish: true
dg-home: false
---
## 1 分布式系统概述
### 1.1使用分布式系统的原因：
- 连接物体上分离的机器，共享系统资源
- 使用并行(parallelism)，提高性能
- 提高容错性(Tolerate faults)
- 设备物理隔离，提高安全性

### 1.2分布式系统发展：
1980s，局域网集群 --- 1990s，数据中心（搜索引擎存储数据，网站存储用户数据） ---  2000s，云计算(cloud computing)

### 1.3挑战：
- 大量并发场景时的资源调度
- 处理部分故障，接管故障机
- 实现性能优势

### 1.4支撑分布式应用程序开发的底层基础架构
常见的有：
- Storage，存储基础架构，比如键值服务器、文件系统等
- Computation，计算框架，用来编排或构件分布式应用程序，比如经典的mapreduce
- Comminication，网络通信，如RPC(远程过程调用)

### 1.5分布式系统的main topics:
1. Fault tolerance 容错性
	- availability 可用性，在故障时仍可提供服务
		- 关键技术：replication 复制
	- recoverability 可恢复性，当机器故障后，可以恢复到之前的工作状态
		- 关键技术：
			logging/transaction 日志，事务
			durable storage，将数据写入持久化的存储器，便于后续恢复工作
2. Consistency 一致性：多个服务应该提供一致的服务，客户端请求哪个服务端获取的响应需要一致。
3. Performance 性能：分布式系统希望能比单机系统要具备更高的性能，但是提高性能和提供容错性、一致性是冲突的。为保证性能，会对容错性和一致性做出牺牲。
	性能考虑的指标：
	- 吞吐量(throughput)：希望吞吐量随机器数量增加而增加
	- 延迟(latency)：
		- 尾部延迟(tail latency)：一台慢的机器会导致整个用户体验变慢

## 2 mapreduce
论文翻译：[谷歌MapReduce经典论文翻译(中英对照) - 小熊餐馆 - 博客园 (cnblogs.com)](https://www.cnblogs.com/xiaoxiongcanguan/p/16724085.html)
### 2.1 概述
MapReduce简化了大规模集群上的分布式计算操作。用户通过编写Map和Reduce函数来处理大量数据。
![[mapreduce.png]]
通过这种模型和实现，Google能有效地处理数十TB大小的数据集，运行在成千上万的普通计算机上，而且具有很高的扩展性和容错性。
### 2.2 执行流程

![[MapReduce执行流程.png]]
1. **输入数据分片 (Splitting Input Data)**：
    - 对要处理的数据进行分片，将输入文件分成M个分片，每个分片的大小一般在16M到64M之间。
    - 在集群中启动MapReduce实例，包括一个Master节点和多个Worker节点。
2. **任务分配 (Task Assignment)**：
    - 主节点Master负责将M个Map任务和R个Reduce任务分配给空闲的Worker节点。每个Worker节点会被分配一个任务（Map或Reduce任务）。
3. **Map任务执行 (Executing Map Tasks)**：
    - 被分配Map任务的Worker读取对应的输入片段，并解析出键/值对，将其传递给用户定义的Map函数。
    - Map函数处理后输出中间键/值对，将这些中间结果缓存在内存中，并会被定期保存在map结点的本地磁盘上。
4. **Shuffle和排序 (Shuffling and Sorting)**：
	- 内存中的中间键/值对通过Partition函数（如`hash(key) mod R`）分成R个部分，并写入本地磁盘。
    - Map Worker将这些键值对在磁盘中的位置上传给Master节点，Master节点再将这些位置信息传给相应的Reduce Worker。
    - 当Reduce Worker收到数据存储位置信息后，使用RPC读取Map Worker磁盘中的数据。
    - Reduce Worker将读取的数据按中间键进行排序，将相同键的数据聚合到一起。如果中间数据过多，无法全部载入内存，则使用外部排序。同一个reduce任务经常会分到有着不同key的数据，因此这个排序很有必要。
5. **Reduce任务执行 (Executing Reduce Tasks)**：
    - Reduce Worker遍历排序后的中间键/值对，按照键进行分组聚合（key, {values}）。
    - 将聚合后的数据传递给用户定义的Reduce函数，Reduce函数处理后输出最终结果，追加到输出文件中。每个reduce任务产生1个文件。
6. **结果返回**：
    - 当所有Map和Reduce任务都完成后，Master节点唤醒用户程序，用户程序返回最终结果。

- **Master节点的数据结构 (Master Node Data Structures)**：
    - Master节点维护任务状态（空闲、处理中、完成）和每台Worker节点的ID（对应非空闲任务）。
    - Master节点存储每个已完成的Map任务生成的R个中间文件的位置和大小信息。这些信息在Map任务完成时接收，并逐步发送到正在处理的Reduce任务节点。

### 2.3应用场景
1. 词频统计(Word Count)
 map函数输出文档中的每个词以及它的频率(word, 1)，reduce函数把所有同样词的频率累加起来得到(word, total count)
	![[词频统计示例.png]]
2. URL访问频次统计：map函数处理网页请求的日志，对每个URL输出(URL, 1)的键值对。reduce函数将相同URL的所有值相加并输出(URL, 总访问数)。
3. 分布式Grep：在大量文档中搜索匹配某一特定模式的行。
	map函数在匹配到给定的模式时输出一行。reduce函数是个恒等函数，将中间数据原封不动地输出。

### 2.4 容错
- worker 故障
	master会周期性地ping每个worker，规定时间内没有返回信息，则master将其标记为fail。
	- 对于执行map任务的结点：master将所有由这个失效的worker做完的(completed)、正在做的(in-progress) map任务标记为初始状态(idle)等待其他worker认领任务。
		当一个map任务被第一次由工作节点A执行，然后因A故障后由工作节点B重新执行时，所有执行reduce任务的工作节点都会被通知重新执行。任何还未读取工作节点A数据的reduce任务都会从工作节点B读取数据。
	- 对于执行reduce任务的结点：已完成的reduce任务存储在全局文件系统中，不需要重新执行，其他任务将重新调度执行。
		当reduce任务完成时，其结果首先写入临时文件，然后临时文件原子性地重命名为最终输出文件。原子性重命名操作由底层文件系统提供，确保即使在并发重命名的情况下，最终只有一个文件存在。
		如reduce任务R1在节点A上执行，并将结果写入临时文件`temp_R1`，任务完成后，节点A将`temp_R1`原子性地重命名为`output_R1`。如果节点A在任务完成前出现故障，reduce任务R1会被重新分配给节点B。节点B执行任务并将结果写入`temp_R1_B`，然后重命名为`output_R1`。由于重命名操作的原子性，最终只有一个`output_R1`文件存在，包含reduce任务R1的结果。

- master 故障
	从头开始mapreduce操作。或加入检查点，周期性地将master节点存储的数据保存至磁盘。

### 2.5 加速并行处理
#### 局部性
在MapReduce的任务中，至少需要$M*R$次的网络传输，才能将中间文件发送给reduce所在的worker节点上。同时把输入文件发送给map任务所在的worker 也是非常消耗网络带宽的事情。调度map任务时应让执行任务的机器尽量靠近任务所需输入数据所在的机器。
#### 任务粒度
设置M的大小使得每个独立任务所需的输入数据大约在16MB至64MB之间，设置R的大小为预期使用worker机器的小几倍
### 备用任务(Backup Tasks)
可能存在slow worker拖慢整个程序，所以当剩下任务很少时，可以启动备用任务,即同时在两个节点上执行相同的任务。这样只要其中一个先返回即可结束整个任务,同时释放未完成的任务所占用的资源。

### 2.5 MapReduce调优技巧
#### 分区函数
分区函数一般是`Hash(key) % R`，但也可以定义自己的分区方法，如在URL访问频次统计时，可以使用`hash(Hostname(urlkey)) % R`将同一主机的url分配到同一个reduce任务
#### 顺序保证
在一个分区R中，MapReduce保证所有中间k/v对都是按key排序的。
#### 组合器函数(Combiner Function)
以单词频数统计为例，一个map任务可能产生成百上千的形如<the,1>的记录，可以利用combiner函数先将本地的中间结果合并一下，如合并成<the, 100>，就可以降低数据传输占用的宽带。reduce再继续将多个,<the, xxx>进行合并。
Combiner函数会在执行map任务的机器上执行一次，通常Combiner函数和Reduce函数实现的代码一致。
#### 输入输出
MapReduce库支持不同的格式的输入数据。如文本模式，key是行数，value是该行内容。程序员可以定义Reader接口来适应不同的输入类型。

#### 2.6单词频数统计程序
```cpp
#include "mapreduce/mapreduce.h"
// User’s map function 
class WordCounter : public Mapper { 
    public: 
	    virtual void Map(const MapInput& input) { 
	        const string& text = input.value(); 
	        const int n = text.size(); for (int i = 0; i < n; ) { 
	            // Skip past leading whitespace 
	            while ((i < n) && isspace(text[i])) 
	               i++;
	            // Find word end 
	            int start = i; 
	            while ((i < n) && !isspace(text[i]))
	               i++;
	            if (start < i) 
	               Emit(text.substr(start,i-start),"1");
        }
    }
};
REGISTER_MAPPER(WordCounter);

// User’s reduce function 
class Adder : public Reducer { 
    virtual void Reduce(ReduceInput* input) { 
        // Iterate over all entries with the 
        // same key and add the values 
        int64 value = 0; 
        while (!input->done()) { 
            value += StringToInt(input->value()); 
            input->NextValue();
        }
        // Emit sum for input->key() 
        Emit(IntToString(value));
    }
};

REGISTER_REDUCER(Adder);
int main(int argc, char** argv) { 
    ParseCommandLineFlags(argc, argv);
    MapReduceSpecification spec;
    // Store list of input files into "spec" 
    for (int i = 1; i < argc; i++) { 
        MapReduceInput* input = spec.add_input(); 
        input->set_format("text"); 
        input->set_filepattern(argv[i]); 
        input->set_mapper_class("WordCounter");
    }
    // Specify the output files: 
    // /gfs/test/freq-00000-of-00100 
    // /gfs/test/freq-00001-of-00100 
    // ... 
    MapReduceOutput* out = spec.output();
    out->set_filebase("/gfs/test/freq"); 
    out->set_num_tasks(100); 
    out->set_format("text"); 
    out->set_reducer_class("Adder");
    // Optional: do partial sums within map 
    // tasks to save network bandwidth 
    out->set_combiner_class("Adder");
    
    // Tuning parameters: use at most 2000 
    // machines and 100 MB of memory per task 

    spec.set_machines(2000); 
    spec.set_map_megabytes(100); 
    spec.set_reduce_megabytes(100);

    // Now run it 
    MapReduceResult result; 
    if (!MapReduce(spec, &result)) abort();
    
    // Done: ’result’ structure contains info
    // about counters, time taken, number of 
    // machines used, etc.
    return 0;
}

```