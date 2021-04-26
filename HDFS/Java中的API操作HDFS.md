# 1、使用到的类说明

---

## 1、FileSystem：文件系统的抽象基类
* FileSystem的实现取决于fs.defaultFS的配置！
    * 有两种配置方式：
      * LocalFileSystem：本地文件系统--》fs.defaultFS=file:///
      * DistributedFileSystem：分布式文件系统--》fs.defaultFS=hdfs://xxx:9000
    * 声明用户身份：
      * FileSystem fs = FileSystem.get(new URI("hdfs://hp1:9000"), conf, "hjc");
  
## 2、Configuration：功能是读取配置文件中的参数信息

* Configuration在读取配置文件的参数是，根据文件名，从类路径按照顺序读取配置文件。先读取 xxx.default.xml，再读取xxx-site.xml
* Configuration类一加载，就会默认读取8个配置文件，将这8个配置文件中的所有参数，读取到一个Map集合中。
* 同时，Configuration也提供了set(name, value)，来手动设置用户自定义的参数。

## 3、FileStatus：代表一个文件的状态（文件的属性信息）

## 4、offset和length
* offset是偏移量：指块在文件中的起始位置
* length是文件的长度：指块大小

## 5、LocatedFileStatus：是FileStatus的子类，除了文件的属性，还有块的位置信息。

---

# 2、具体代码展示
## **常用API的实现**
``` java {.line-numbers}
public class TestHDFS {	
	private FileSystem fs;	
	private Configuration conf = new Configuration();	
	@Before
	public void init() throws IOException, URISyntaxException {		
		//创建一个客户端对象
		 fs=FileSystem.get(new URI("hdfs://hadoop101:9000"),conf);		
	}	
	@After
	public void close() throws IOException {		
		if (fs !=null) {
			fs.close();
		}		
	}
	//  hadoop fs(运行一个通用的用户客户端)   -mkdir /xxx
	//  创建一个客户端对象 ，调用创建目录的方法，路径作为方法的参数掺入
	@Test
	public void testMkdir() throws IOException, InterruptedException, URISyntaxException {	
		fs.mkdirs(new Path("/eclipse2"));
	}
	// 上传文件： hadoop fs -put 本地文件  hdfs
	@Test
	public void testUpload() throws Exception {	
		fs.copyFromLocalFile(false, true, new Path("e:/sts.zip"), new Path("/"));	
	}
	// 下载文件：  hadoop fs -get hdfs  本地路径
	@Test
	public void testDownload() throws Exception {
		
		fs.copyToLocalFile(false, new Path("/wcinput"), new Path("e:/"), true);
		
	}
	// 删除文件：  hadoop fs -rm -r -f  路径
	@Test
	public void testDelete() throws Exception {
		fs.delete(new Path("/eclipse"), true);	
	}
	// 重命名：  hadoop fs -mv  源文件   目标文件
	@Test
	public void testRename() throws Exception {
		fs.rename(new Path("/eclipse1"), new Path("/eclipsedir"));
	}
	// 判断当前路径是否存在
	@Test
	public void testIfPathExsits() throws Exception {	
		System.out.println(fs.exists(new Path("/eclipsedir1")));
	}
	// 判断当前路径是目录还是文件
	@Test
	public void testFileIsDir() throws Exception {	
		//Path path = new Path("/eclipsedir");
		Path path = new Path("/wcoutput1");	
		// 不建议使用此方法，建议好似用Instead reuse the FileStatus returned 
		//by getFileStatus() or listStatus() methods.

	/*	System.out.println(fs.isDirectory(path));
		System.out.println(fs.isFile(path));*/
		//FileStatus fileStatus = fs.getFileStatus(path);
		FileStatus[] listStatus = fs.listStatus(path);
		for (FileStatus fileStatus : listStatus) {	
			//获取文件名 Path是完整的路径 协议+文件名
			Path filePath = fileStatus.getPath();
			System.out.println(filePath.getName()+"是否是目录："+fileStatus.isDirectory());
			System.out.println(filePath.getName()+"是否是文件："+fileStatus.isFile());
		}		
	}
	// 获取到文件的块信息
	@Test
	public void testGetBlockInfomation() throws Exception {
		
		Path path = new Path("/sts.zip");
		
		RemoteIterator<LocatedFileStatus> status = fs.listLocatedStatus(path);
		
		while(status.hasNext()) {
			
			LocatedFileStatus locatedFileStatus = status.next();
			
			System.out.println("Ownner:"+locatedFileStatus.getOwner());
			System.out.println("Group:"+locatedFileStatus.getGroup());
			
			//---------------块的位置信息--------------------
			BlockLocation[] blockLocations = locatedFileStatus.getBlockLocations();
			
			for (BlockLocation blockLocation : blockLocations) {
				System.out.println(blockLocation);
				System.out.println("------------------------");	
			}	
		}	
	}
}
```

---

## **自定义上传下载文件**

``` java
/*
 * 1. 上传文件时，只上传这个文件的一部分
 * 
 * 2. 下载文件时，如何只下载这个文件的某一个块？ 
 * 			或只下载文件的某一部分？
 */
public class TestCustomUploadAndDownload {

   private FileSystem fs;
   private FileSystem localFs;
	
	private Configuration conf = new Configuration();
	
	@Before
	public void init() throws IOException, URISyntaxException {
		
		//创建一个客户端对象
		 fs=FileSystem.get(new URI("hdfs://hadoop101:9000"),conf);
		 
		 localFs=FileSystem.get(new Configuration());
		
	}
	
	@After
	public void close() throws IOException {
		
		if (fs !=null) {
			fs.close();
		}
		
	}
	
	// 只上传文件的前10M
	/*
	 * 官方的实现
	 * InputStream in=null;
      OutputStream out = null;
      try {
        in = srcFS.open(src);
        out = dstFS.create(dst, overwrite);
        IOUtils.copyBytes(in, out, conf, true);
      } catch (IOException e) {
        IOUtils.closeStream(out);
        IOUtils.closeStream(in);
        throw e;
      }
	 */
	
	@Test
	public void testCustomUpload() throws Exception {
		
		//提供两个Path，和两个FileSystem
		Path src=new Path("e:/悲惨世界(英文版).txt");
		Path dest=new Path("/悲惨世界(英文版)10M.txt");
		
		// 使用本地文件系统中获取的输入流读取本地文件
		FSDataInputStream is = localFs.open(src);
		
		// 使用HDFS的分布式文件系统中获取的输出流，向dest路径写入数据
		FSDataOutputStream os = fs.create(dest, true);
		
		// 1k
		byte [] buffer=new byte[1024];
		
		// 流中数据的拷贝
		for (int i = 0; i < 1024 * 10; i++) {
			
			is.read(buffer);
			os.write(buffer);
			
		}
		//关流
		 IOUtils.closeStream(is);
	     IOUtils.closeStream(os);
	}
	@Test
	public void testFirstBlock() throws Exception {
		//提供两个Path，和两个FileSystem
		Path src=new Path("/sts.zip");
		Path dest=new Path("e:/firstblock");
		
		// 使用HDFS的分布式文件系统中获取的输入流，读取HDFS上指定路径的数据
		FSDataInputStream is = fs.open(src);
		// 使用本地文件系统中获取的输出流写入本地文件
		FSDataOutputStream os = localFs.create(dest, true);
		
		// 1k
		byte [] buffer=new byte[1024];
				
		// 流中数据的拷贝
		for (int i = 0; i < 1024 * 128; i++) {
					
			is.read(buffer);
			os.write(buffer);
					
		}		
		//关流
		IOUtils.closeStream(is);
		IOUtils.closeStream(os);
	
	}
	
	@Test
	public void testSecondBlock() throws Exception {
		//提供两个Path，和两个FileSystem
		Path src=new Path("/sts.zip");
		//Path dest=new Path("e:/secondblock");
		Path dest=new Path("e:/thirdblock");
		
		// 使用HDFS的分布式文件系统中获取的输入流，读取HDFS上指定路径的数据
		FSDataInputStream is = fs.open(src);
		// 使用本地文件系统中获取的输出流写入本地文件
		FSDataOutputStream os = localFs.create(dest, true);
		
		//定位到流的指定位置
		is.seek(1024*1024*128*2);
		
		// 1k
		byte [] buffer=new byte[1024];
				
		// 流中数据的拷贝
		for (int i = 0; i < 1024 * 128; i++) {
					
			is.read(buffer);
			os.write(buffer);
					
		}				
		//关流
		IOUtils.closeStream(is);
		IOUtils.closeStream(os);		
	}
	
	@Test
	public void testFinalBlock() throws Exception {
		//提供两个Path，和两个FileSystem
		Path src=new Path("/sts.zip");
		//Path dest=new Path("e:/secondblock");
		Path dest=new Path("e:/fourthblock");		
		// 使用HDFS的分布式文件系统中获取的输入流，读取HDFS上指定路径的数据
		FSDataInputStream is = fs.open(src);
		// 使用本地文件系统中获取的输出流写入本地文件
		FSDataOutputStream os = localFs.create(dest, true);		
		//定位到流的指定位置
		is.seek(1024*1024*128*3);
		//buffSize 默认不能超过4096
		IOUtils.copyBytes(is, os, 4096, true);		
	}
}
```

