自定义InputFormat类
``` java
/*
 * 1. 改变切片策略，一个文件固定切1片，通过指定文件不可切
 * 2. 提供RR ，这个RR读取切片的文件名作为key,读取切片的内容封装到bytes作为value
 */
public class MyInputFormat extends FileInputFormat {

	@Override
	public RecordReader createRecordReader(InputSplit split, TaskAttemptContext context)
			throws IOException, InterruptedException {
		
		return new MyRecordReader();
	}
	
	// 重写isSplitable
	@Override
	protected boolean isSplitable(JobContext context, Path filename) {
		
		return false;
	}
	
}
```

自定义RecordReader类
``` java
/*
 * RecordReader从MapTask处理的当前切片中读取数据
 * 
 * XXXContext都是Job的上下文，通过XXXContext可以获取Job的配置Configuration对象
 */
public class MyRecordReader extends RecordReader {
	
	private Text key;
	private BytesWritable value;
	
	private String filename;
	private int length;
	
	private FileSystem fs;
	private Path path;
	
	private FSDataInputStream is;
	
	private boolean flag=true;

	// MyRecordReader在创建后，在进入Mapper的run()之前，自动调用
	// 文件的所有内容设置为1个切片，切片的长度等于文件的长度
	@Override
	public void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {

		FileSplit fileSplit=(FileSplit) split;
		
		filename=fileSplit.getPath().getName();
		
		length=(int) fileSplit.getLength();
		
		path=fileSplit.getPath();
		
		//获取当前Job的配置对象
		Configuration conf = context.getConfiguration();
		
		//获取当前Job使用的文件系统
		fs=FileSystem.get(conf);
		
		 is = fs.open(path);
		
	}

	// 读取一组输入的key-value，读到返回true,否则返回false
	// 将文件的名称封装为key，将文件的内容封装为BytesWritable类型的value，返回true
	// 第二次调用nextKeyValue()返回false
	@Override
	public boolean nextKeyValue() throws IOException, InterruptedException {
		
		if (flag) {
			
			//实例化对象
			if (key==null) {
				key=new Text();
			}
			
			if (value==null) {
				value=new BytesWritable();
			}
			
			//赋值
			//将文件名封装到key中
			key.set(filename);
			
			// 将文件的内容读取到BytesWritable中
			byte [] content=new byte[length];
			
			IOUtils.readFully(is, content, 0, length);
			
			value.set(content, 0, length);
			
			flag=false;
			
			return true;
			
		}
		
		return false;
	}

	//返回当前读取到的key-value中的key
	@Override
	public Object getCurrentKey() throws IOException, InterruptedException {
		return key;
	}

	//返回当前读取到的key-value中的value
	@Override
	public Object getCurrentValue() throws IOException, InterruptedException {
		return value;
	}

	//返回读取切片的进度
	@Override
	public float getProgress() throws IOException, InterruptedException {
		return 0;
	}

	// 在Mapper的输入关闭时调用，清理工作
	@Override
	public void close() throws IOException {
		if (is != null) {
			IOUtils.closeStream(is);
		}
		
		if (fs !=null) {
			fs.close();
		}
	}

}
```
