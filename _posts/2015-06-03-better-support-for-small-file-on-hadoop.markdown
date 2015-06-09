---
layout: post
title:  "让 HDFS 更好的支持小文件"
date:   2015-06-03 20:00:00
tags:
    - Hadoop
    - HBase
    - HDFS
---

### 背景
什么是小文件问题？我们知道 HDFS 的框架设计是来自 Google 的 GFS 的开源实现，适合于处理数据规模大的文件，其实它在小文件的处理上并不擅长，造成 HDFS 在处理大量小文件效率不高的原因主要在两个方面：首先 HDFS 将整个系统的命名空间存储在 FSImage 文件中，当集群启动时 NameNode 会将这个文件加载到内存中，所以当大量小文件存在时会导致命名空间文件变的十分庞大从而影响检索的效率使得系统性能降低；其次，HDFS 中文件是分块存储的，默认的数据块大小是 128M，这样每个小文件都会至少占据一个文件块，当客户端对小文件进行操作时都要经过 NameNode 的查找获得文件块所在的地址进行存取，这是一个耗时的操作，然后紧接着会进行大量的 I/O 操作，导致小文件数据流的传输时间远小于其他的耗时，这是一个严重的性能瓶颈。

### Hadoop 提供的解决方案

#### 1. Hadoop Archive(HAR)
HAR 是一种特殊的档案格式，一个 HAR 对应一个文件系统目录，其扩展名是 *.har。HAR 包含元数据（形式是_index和_masterindx）和数据（part-*）文件，_index文件包含了档案中的文件的文件名和位置信息。这样在减少 NameNode 的内存使用的同时允许对文件进行透明的访问。但 HAR 是不可改变的，所以重命名，删除和创建都会返回错误。而且需要系统管理员定期执行命令进行归档操作来维护。

#### 2. SequenceFile
SequenceFile 由二进制的 key/value 构成，使用 key 作为小文件名，value 为具体内容，可以将大批小文件合并成一个大文件。
![](/media/files/2015/06/03/sequence_file.png "Sequence File")

#### 3. CombineFileInputFormat
CombineFileInputFormat 是一种新的输入格式，用于将多个文件合并成一个单独的 split，另外它会考虑数据的存储位置。对于 HDFS 中已经存在大量小文件的情况比较适用。

### 如何改进
上面提到的所有方案均需要用户每隔一段时间对小文件进行合并来减轻 NameNode 的内存负担。所以为什么不能将小文件的处理模块作为一个黑盒嵌入到 HDFS 中，在用户上传文件的时候自动判断是否为小文件然后再自动进行合并呢？

### HDFS 读/写文件数据流
为此我们首先需要了解 HDFS 是怎么进行读/写文件的，

#### 1. HDFS 读文件数据流
![](/media/files/2015/06/03/hdfs_read.png "Read")
文件读取的过程如下：

* 客户端(HDFSclient) 用 FileSystem 的 open() 函数打开文件。
* DistributedFileSystem 用 RPC 调用元数据节点，得到文件的数据块信息。
* 对于每一个数据块，元数据节点返回保存数据块的数据节点的地址。
* DistributedFileSystem 返回 FSDataInputStream 给客户端，用来读取数据。
* 客户端调用 stream 的 read() 函数开始读取数据。
* DFSInputStream 连接保存此文件第一个数据块的最近的数据节点。
* Data 从数据节点读到客户端(HDFSclient)。
* 当此数据块读取完毕时，DFSInputStream 关闭和此数据节点的连接，然后连接此文件下一个数据块的最近的数据节点。
* 当客户端读取完毕数据的时候，调用 FSDataInputStream 的 close 函数。
* 在读取数据的过程中，如果客户端在与数据节点通信出现错误，则尝试连接包含此数据块的下一个数据节点。
* 失败的数据节点将被记录，以后不再连接。

#### 2. HDFS 写文件数据流
![](/media/files/2015/06/03/hdfs_write.png "Write")
写入文件的过程如下：

* 客户端调用 create() 来创建文件
* DistributedFileSystem 用 RPC 调用元数据节点，在文件系统的命名空间中创建一个新的文件。
* 元数据节点首先确定文件原来不存在，并且客户端有创建文件的权限，然后创建新文件。
* DistributedFileSystem 返回 DFSOutputStream，客户端用于写数据。
* 客户端开始写入数据，DFSOutputStream 将数据分成块，写入 Data queue。
* Data queue 由 Data Streamer 读取，并通知元数据节点分配数据节点，用来存储数据块(每块默认复制3块)。分配的数据节点放在一个 pipeline 里。
* Data Streamer 将数据块写入 pipeline 中的第一个数据节点。第一个数据节点将数据块发送给第二个数据节点。第二个数据节点将数据发送给第三个数据节点。
* DFSOutputStream 为发出去的数据块保存了 ack queue，等待 pipeline 中的数据节点告知数据已经写入成功。
* 如果数据节点在写入的过程中失败：

	* 关闭 pipeline，将 ack queue 中的数据块放入 data queue 的开始。
	* 当前的数据块在已经写入的数据节点中被元数据节点赋予新的标示，则错误节点重启后能够察觉其数据块是过时的，会被删除。
	* 失败的数据节点从 pipeline 中移除，另外的数据块则写入 pipeline 中的另外两个数据节点。
	* 元数据节点则被通知此数据块是复制块数不足，将来会再创建第三份备份。

* 当客户端结束写入数据，则调用 stream 的 close 函数。此操作将所有的数据块写入 pipeline 中的数据节点，并等待 ack queue 返回成功。最后通知元数据节点写入完毕。

### 我的解决方法
所以针对此，我的解决办法是在原有 HDFS 基础上添加一个小文件处理模块，当一个文件到达时，判断该文件是否属于小文件，如果是，则交给小文件处理模块处理，否则，交给通用文件处理模块处理。小文件处理模块的设计思想是，先将很多小文件合并成一个大文件，然后为这些小文件建立索引，以便进行快速存取和访问。

### 如何实现
我们先来看下一段常规的读/写文件代码，

	public void read() throws IOException {
        Configuration conf = new Configuration();
        FileSystem fileSystem = FileSystem.get(URI.create("hdfs://localhost:9000/usr/ivanchou/file"), conf);
        Path path = new Path("hdfs://localhost:9000/usr/ivanchou/file");
        FSDataInputStream is = fileSystem.open(path);
        BufferedReader br = new BufferedReader(new InputStreamReader(is));

        String line;
        while ((line = br.readLine()) != null) {
            System.out.println(line);
        }
        br.close();
    }

    public void write() throws IOException{
        InputStream in = new BufferedInputStream(new FileInputStream("~/Documents/local_file"));
        Configuration conf = new Configuration();
        FileSystem fileSystem = FileSystem.get(URI.create("hdfs://localhost:9000/usr/ivanchou/file"), conf);
        Path path = new Path("hdfs://localhost:9000/usr/ivanchou/file");
        OutputStream out = fileSystem.create(path);
        IOUtils.copyBytes(in, out, 4096, true);
    }

这是一段常规的从 HDFS 读/写文件的代码，我们在前面分析了 HDFS 读/写文件的数据流。那么有个疑问，这里的 FileSystem 是什么？系统又是如何最终得到 DistributedFileSystem 的呢？继续看代码，
	
	public static FileSystem get(URI uri, Configuration conf) throws IOException {
        String scheme = uri.getScheme();
        String authority = uri.getAuthority();

        if (scheme == null && authority == null) {     // use default FS
            return get(conf);
        }

        if (scheme != null && authority == null) {     // no authority
            URI defaultUri = getDefaultUri(conf);
            if (scheme.equals(defaultUri.getScheme())    // if scheme matches default
                    && defaultUri.getAuthority() != null) {  // & default has authority
                return get(defaultUri, conf);              // return default
            }
        }

        String disableCacheName = String.format("fs.%s.impl.disable.cache", scheme);
        if (conf.getBoolean(disableCacheName, false)) {
            return createFileSystem(uri, conf);
        }

        return CACHE.get(uri, conf);
    }

    private static FileSystem createFileSystem(URI uri, Configuration conf) throws IOException {
        Class<?> clazz = getFileSystemClass(uri.getScheme(), conf);
        if (clazz == null) {
            throw new IOException("No FileSystem for scheme: " + uri.getScheme());
        }
        FileSystem fs = (FileSystem) ReflectionUtils.newInstance(clazz, conf);
        fs.initialize(uri, conf);
        return fs;
    }

这是 FileSystem 的源码，可以看到会根据 URI 来判断 scheme，然后通过 Java 的反射机制获取到对应的实体类。所以我们大概知道这是为什么了，注意到 FileSystem 是个抽象类，所以查看文档得知 DistributedFileSystem 是它的实体类，那么为什么 Hadoop 知道是 DistributedFileSystem 而不是其他的实体类呢？Hadoop 有一个抽象的文件系统概念，HDFS 只是其中的一个实现，Java 抽象类 org.apache.hadoop.fs.FileSystem 定义了 Hadoop 中的一个文件系统接口，并且该抽象类有几个具体实现，如下,

文件系统|URI 方案|Java 实现(均包含在 org.apache.hadoop 包中)|描述|
---|---|---|---
Local|file|fs.LocalFileSystem|支持有客户端校验和本地文件系统。带有校验和的本地系统文件在 fs.RawLocalFileSystem 中实现。
HDFS|hdfs|hdfs.DistributionFileSystem|Hadoop 的分布式文件系统。
HFTP|hftp|hdfs.HftpFileSystem|支持通过 HTTP 方式以只读的方式访问 HDFS，distcp 经常用在不同的 HDFS 集群间复制数据。
HSFTP|hsftp|hdfs.HsftpFileSystem|支持通过 HTTPS 方式以只读的方式访问 HDFS。
HAR|har|fs.HarFileSystem|构建在Hadoop文件系统之上，对文件进行归档。Hadoop 归档文件主要用来减少 NameNode 的内存使用。
KFS|kfs|fs.kfs.KosmosFileSystem|Cloudstore (其前身是 Kosmos 文件系统) 文件系统是类似于 HDFS 和 Google 的 GFS 文件系统，使用 C++ 编写。
FTP|ftp|fs.ftp.FtpFileSystem|由 FTP 服务器支持的文件系统。
S3(本地)|s3n|fs.s3native.NativeS3FileSystem|基于 Amazon S3 的文件系统。
S3(基于块)|s3|fs.s3.NativeS3FileSystem|基于 Amazon S3 的文件系统，以块格式存储解决了 S3 的 5GB 文件大小的限制。

这些都是配置在 Hadoop 系统的配置文件中的，`createFileSystem(URI uri, Configuration conf)` 的两个参数中的 conf 就是读取配置文件信息。同样如果你分析 shell 命令 `hdfs dfs -put <local> <dst>`，查看 `hdfs` 的实现，你会发现其调用的是 `org.apache.hadoop.fs.FsShell` 的 `copyFromLocal` 方法，然后 `copyFromLocal` 同样调用的是 `org.apache.hadoop.fs.FileSystem`，至此，我们的思路应该是相当清楚了。从前面对 HDFS 读文件的分析我们知道 DistributedFileSystem 与 NameNode 之间的通信是采取 RPC 并封装在底层的，所以我们应该写一套自己的 FileSystem 去完成我们要做的事情。这里直接继承 DistributedFileSystem 并复写其基本的读/写方法。

#### 1. improve HDFS

好了，还是没讲怎么实现。下面就简单的把思路整理成代码吧，
	

	public class SmallFileSystem extends DistributedFileSystem {

	    @Override
	    public String getScheme() {
	        return "sdfs";
	    }

	    @Override
	    public void initialize(URI uri, Configuration conf) throws IOException {
	        URI hdfsUri = URI.create("hdfs" + "://" + uri.getAuthority() + "/" + uri.getPath());
	        super.initialize(hdfsUri, conf);
	    }

	    @Override
	    public FSDataInputStream open(Path f, int bufferSize) throws IOException {

	        URI uri = f.toUri(); f -> sdfd://localhost/9000/usr/ivanchou/file
	        String filePath = f.toString();
	        if (uri.getScheme() != null) {
	            // change f -> hdfs://...
	        }
	        // check the file status
	        FileStatus status = checkFileStatus(filePath);

	        if (status == FileStatus.EXIST) {
	            // if small file is still on HBase, then direct read and return it.

	            return ...;
	        } else if (status == FileStatus.REMOVE) {
	            // if small file is merged and store to HDFS, then use Seekable interface.
	            
	            return ...;
	        } else {
	            // status == FileStatus.NOT_EXIST
	            // it stands for the file never store to HBase, direct handle to HDFS.
	            return super.open(f, bufferSize);
	        }
	    }

	    public FSDataOutputStream create(String local, Path dst) throws IOException {
	        File file = new File(local);
	        if (file.exists() && file.length() < SMALL_FILE_BYTES) {
	            // is a small file, store to HBase.
	            // key:dst | value:content

	            if (smallFileServer.getHBaseSize() >= MERGE_FILE_BYTES) {
	            	// merge all the files on HBase to one and then store to HDFS.
	            }
	        } else {
	            // not a small file, handle to HDFS.
	            return this.create(dst);
	        }
	    }
	}

其中，大致就做了这几件事：读文件时，判断 HBase 中是否存在；写文件时，判断文件的大小是否超过阀值，以及每次写一个文件到 HBase 中都要检查一下 HBase 中的所有文件大小是否超过该合并的大小。当然可能注意到 `create` 方法并没有直接复写，因为这里需要判断用户上传的文件大小后才能做出决定，所以这里小小修改了下上传的代码。于是乎，在程序中你就可以直接将 URI 改为自定义的 `sdfs://localhost:9000/usr/ivanchou/file`，再加上一句 `conf.set("fs.sdfs.impl", com.ivanchou.SmallFileSystem.class.getName());` 或者直接在配置文件中配置你的 jar 路径就搞定了。

于是，我们的读/写文件数据流是这样的，

* 读文件
![](/media/files/2015/06/03/sdfs_read.png "Read")
* 写文件
![](/media/files/2015/06/03/sdfs_write.png "Write")
写文件的 SmallFileNode 画的不够完善，大概就是其中同样实现了一个传统的 HDFS client。

#### 2. HBase 小文件表的设计
这里我分成了两张表，一张是索引表，一张就是小文件数据表。

* Index Table
![](/media/files/2015/06/03/table_index.png "Index")
其中，`exist=0` 表示小文件曾经存在过，但被合并到 HDFS 中了，`exist=1` 表示小文件依然在 HBase 中，这时候其他的 column 其实是不存在的。filepath 指的是合并后的文件的 URI，offset 表示在新文件中的偏移量，length 表示文件的长度。

* Data Table
![](/media/files/2015/06/03/table_data.png "Data")
这里面一个文件被划分为好几个块，每个分块的 key 用分块编号&分块长度标识。后来我在数据表中添加了一个特别的 row，也就是 length，每次写/删数据表中的小文件时，同时加/减 length 字段。

#### 3. 使用 RPC 协议来通信小文件处理模块（黑盒）
什么是 RPC？网络通信模块是分布式系统中最底层的模块，它支撑了上层分布式环境下复杂的进程间通信（Inter-Process Communication，IPC）逻辑，是所有分布式系统的基础。如何隐藏底层通信的细节，是实现分布式系统访问透明性的重要议题。Hadoop RPC 是 RPC 的一种简单实现，它是 Hadoop 中多个分布式子系统（如HDFS、MapReduce、Yarn等）公用的网络通信模块。RPC 中的访问透明性，是指当用户在一台计算机的程序调用另外一台计算机上的子程序时，用户自身不应感觉到其间涉及跨机器间的通信，而是感觉像是在执行一个本地调用。

在 SDFS Client 与 SmallFileNode 之间同样利用 Hadoop RPC 搭建了一个高效的 C/S 通信模型，所有的 HBase 相关操作均在服务端进行。

### 性能提升

未来如果继续的话，需要提升的几个方面：

#### 1. 流式读取

#### 2. 回滚支持

#### 3. HBase 文件系统权限

#### 4. 二进制文件支持

