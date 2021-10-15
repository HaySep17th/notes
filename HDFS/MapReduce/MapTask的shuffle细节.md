# MapTask的shuffle细节
### 1、记录输出收集器的赋值
``` java
if (job.getNumReduceTasks() == 0) {
      output = 
        new NewDirectOutputCollector(taskContext, job, umbilical, reporter);
    } else {
      output = new NewOutputCollector(taskContext, job, umbilical, reporter);
    }
```
* 如果没有Reduce阶段，使用直接的记录收集器，它不会读取数据并进行排序。只会按照Mapper的输出顺序写出。如果有Reduce阶段，使用NewOutputCollector来收集记录。

---

### 2、MapTask记录输出收集器初始化
``` java
NewOutputCollector(org.apache.hadoop.mapreduce.JobContext jobContext,
                       JobConf job,
                       TaskUmbilicalProtocol umbilical,
                       TaskReporter reporter
                       ) throws IOException, ClassNotFoundException {
      //真正干活的收集器，缓存区对象，这个缓冲区对其会对收集的记录进行排序
      collector = createSortingCollector(job, reporter);
      // 获取当前job所有的reduceTask数量，以此数量作为总的分区数，默认JobReduceTask的数量为1
partitions = jobContext.getNumReduceTasks();
// 为MapTask使用的分区器进行赋值
if (partitions > 1) {
        partitioner = (org.apache.hadoop.mapreduce.Partitioner<K,V>)
          ReflectionUtils.newInstance(jobContext.getPartitionerClass(), job);
      } else {
        // 默认使用此分区器进行分区，这个分区器将所有的key-value都分到0号区
        partitioner = new org.apache.hadoop.mapreduce.Partitioner<K,V>() {
          @Override
          public int getPartition(K key, V value, int numPartitions) {
            return partitions - 1;
          }
        };
      }
    }
```

---

### 3、获取Partitioner

``` java
@SuppressWarnings("unchecked")
  public Class<? extends Partitioner<?,?>> getPartitionerClass() 
     throws ClassNotFoundException {
    return (Class<? extends Partitioner<?,?>>) 
      conf.getClass(PARTITIONER_CLASS_ATTR, HashPartitioner.class);
  }
```

* 从配置中获取mapreduce.job.partitioner.class参数，如果没有设置，默认使用HashPartitioner作为分区器。

> HashPartitioner是如何进行分区的：
> ``` java
> public class HashPartitioner<K, V> extends Partitioner<K,V> { 
> /** Use {@link Object#hashCode()} to partition. */
>   public int getPartition(K key, V value, int numReduceTasks) {
>       return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
>   }
>
> }
> ```

---

### 4、缓冲区对象的初始化
如果有Reducer阶段，在MapTask中使用MapOutputBuffer作为缓冲区的实现类。
``` java
public void init(MapOutputCollector.Context context
                    ) throws IOException, ClassNotFoundException {
     

      //sanity checks
     // 获取当前缓冲区溢写的阀值，默认读取配置中mapreduce.map.sort.spill.percent
没有配置，默认为0.8
 final float spillper =        job.getFloat(JobContext.MAP_SORT_SPILL_PERCENT, (float)0.8);
     //获取缓冲区的初始大小，读取mapreduce.task.io.sort.mb，如果没有配置，默认为100
 final int sortmb = job.getInt(JobContext.IO_SORT_MB, 100);
      indexCacheMemoryLimit = job.getInt(JobContext.INDEX_CACHE_MEMORY_LIMIT,
                                         INDEX_CACHE_MEMORY_LIMIT_DEFAULT);
      if (spillper > (float)1.0 || spillper <= (float)0.0) {
        throw new IOException("Invalid \"" + JobContext.MAP_SORT_SPILL_PERCENT +
            "\": " + spillper);
      }
      if ((sortmb & 0x7FF) != sortmb) {
        throw new IOException(
            "Invalid \"" + JobContext.IO_SORT_MB + "\": " + sortmb);
      }
// 实例化排序器，默认使用快排，只排索引
      sorter = ReflectionUtils.newInstance(job.getClass("map.sort.class",
            QuickSort.class, IndexedSorter.class), job);
      

      // k/v serialization
      comparator = job.getOutputKeyComparator();
  // 根据Mapper输出的Key-value类型，获取序列化器
// 如果Mapper的输出Key-value实现了Wriable接口，Hadoop自动提供序列化器
// 如果Mapper输出的key-value没有实现Wriable接口，需要自定提供序列化器，设置到Job中
      keyClass = (Class<K>)job.getMapOutputKeyClass();
      valClass = (Class<V>)job.getMapOutputValueClass();
      serializationFactory = new SerializationFactory(job);
      keySerializer = serializationFactory.getSerializer(keyClass);
      keySerializer.open(bb);
      valSerializer = serializationFactory.getSerializer(valClass);
      valSerializer.open(bb);

   //MapTask输出的记录可以使用压缩格式，到ReduceTask时，再解压缩
  // 压缩可以节省磁盘IO和网络IO，提供MR的运行效率
      // compression
      if (job.getCompressMapOutput()) {
        Class<? extends CompressionCodec> codecClass =
          job.getMapOutputCompressorClass(DefaultCodec.class);
        codec = ReflectionUtils.newInstance(codecClass, job);
      } else {
        codec = null;
      }

      // combiner
      final Counters.Counter combineInputCounter =
        reporter.getCounter(TaskCounter.COMBINE_INPUT_RECORDS);
      combinerRunner = CombinerRunner.create(job, getTaskID(), 
                                             combineInputCounter,
                                             reporter, null);
      if (combinerRunner != null) {
        final Counters.Counter combineOutputCounter =
          reporter.getCounter(TaskCounter.COMBINE_OUTPUT_RECORDS);
        combineCollector= new CombineOutputCollector<K,V>(combineOutputCounter, reporter, job);
      } else {
        combineCollector = null;
      }
      spillInProgress = false;
      minSpillsForCombine = job.getInt(JobContext.MAP_COMBINE_MIN_SPILLS, 3);
      spillThread.setDaemon(true);
      spillThread.setName("SpillThread");
      spillLock.lock();
      try {
        spillThread.start();
        while (!spillThreadRunning) {
          spillDone.await();
        }
      } catch (InterruptedException e) {
        throw new IOException("Spill thread failed to initialize", e);
      } finally {
        spillLock.unlock();
      }
      if (sortSpillException != null) {
        throw new IOException("Spill thread failed to initialize",
            sortSpillException);
      }
    }
```

---

### 5、获取Mapper输出的Key的比较器

``` java
public RawComparator getOutputKeyComparator() {
// 从配置中获取mapreduce.job.output.key.comparator.class的值，必须是RawComparator类型，如果没有配置，默认为null
    Class<? extends RawComparator> theClass = getClass(
      JobContext.KEY_COMPARATOR, null, RawComparator.class);
// 一旦用户配置了此参数，实例化一个用户自定义的比较器实例
    if (theClass != null)
      return ReflectionUtils.newInstance(theClass, this);
//用户没有配置，判断Mapper输出的key的类型是否是WritableComparable的子类，如果不是，就抛异常，如果是，系统会自动为我们提供一个key的比较器
    return WritableComparator.get(getMapOutputKeyClass().asSubclass(WritableComparable.class), this);
}
```

如何自定义比较器？

* 1）、自定义一个类，这个类必须是RawComparator的子类，并设置参数mapreduce.job.output.key.comparator.class = 自定义的类。 
  * 自定义类是，可以继承WritableComparator接口，也可以实现RawComparator
  * 调用方法时，先调用RawComparator.compare(byte[] b1, int s1, int l1, byte[] b2, int s2, int l2),在调用RawComparator.compare()

* 2）、定义Mapper输出的key，让这个Key类实现WritableComparator接口，实现CompareTo()方法。

---

### 6 Combiner

什么是Combiner？
* Combiner实际上本质是一个Reducer类，只有在设置了之后才会运行。

Combiner和Reducer的区别：
* map---sort---copy---sort---reduce
  * Reducer是在reduce阶段调用的，Combiner是在shuffle阶段调用的
  * 本质上都是Reducer类，作用都是对相同的key的key-value进行合并

Combiner的意义：
* 在shuffle阶段对相同key的key-value提前进行合并，可以减少磁盘IO和网络IO

Combiner在MapTask中使用的场景：
> 每次内存溢写前会调用Combiner对溢写的数据进行局部合并
> 在merge时，如果溢写的片段数大于三个，且设置了Combiner，Combiner会再次对数据进行Combiner

Combiner在ReduceTask中使用的场景：
> shuffle线程复制多个MapTask同一分区的数据，复制后执行merge和sort命令，如果复制的数据量过大，需要将部分数据先合并排序后溢写到磁盘，如果设置了Combiner，数据全部复制后，会执行Combiner把磁盘中所有的小部分数据整合排序成一个完整的数据

---

### 7、总结

分区：
* 总的分区数取决于reduceTask的数量
  * 一个Job要启动几个reducetask，取决于用户期望产生几个分区，每个分区最后都会生成一个结果文件

* 分区的确定：
  * reducetask数量大于一，尝试获取用户设置的Partitioner，如果没有设置，默认使用HashPartitioner
  * reducetask数量小于等于一，系统会默认提供一个Partitioner，这个Partitioner会将所有的记录分到0号区

排序：
* 每次内存中的数据溢写到磁盘中时，使用快速排序
* 最后merge数据时，使用归并排序。

比较器： 
* 排序时根据比较器比较的结果进行排序
  * 如果用户自定义了比较器，MapReducer就会使用用户自定义的比较器
  * 如果用户没有自定义比较器，那么Mapper输出的key需要实现WritableComparator接口，只要实现了这个接口，系统会自动提供比较器

> 总结：不管是自己实现比较器还是实现WritableComparator接口，最后在进行比较时，都是调用自己实现的CompareTo()方法的。

Combiner：
* Combiner在shuffle阶段执行
  * 每次溢写前会调用Combiner对溢写的数据进行局部合并
  * 在merge时，如果溢写的片段数 > 3，并设置了Combiner，Combiner会再次对数据进行Combiner

执行流程：
* Partitioner计算分区
* 满足溢写条件，所有数据进行排序，排序时使用比较器对比key
  * 每次溢写前的排序默认使用快速排序
  * 如果设置了Combiner，在溢写前，排好序的结果会先被Combiner进行combine之后再溢写
* 第二个步骤可能会重复很多次，直至所有数据读取完毕
* 所有的溢写片段需要merge为一个总的文件
  * 合并时使用的是归并排序，对key进行排序
  * 如果溢写片段数量超过3，在溢写成一个最终的文件时，Combiner再次调用，执行Combine，combine后再溢写


