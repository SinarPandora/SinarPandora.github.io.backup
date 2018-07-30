---
layout: post
title: 一见倾心的 Scala I/O 开源库 —— Better-Files（官方文档翻译）
---

> 对于 Scala 开发者来说，I/O 操作，如文件的读写通常采用内置的 scala.io.Source API 来实现。但其功能欠缺，而且并不是那么方便（甚至有删除该 API 的提案）。而直接使用 Java 的 io/nio API 又要面对速度慢的问题，以及返回 null，抛出异常等非 Scala 风格设计。
>
> 在 Java 中，我们可以使用诸如 [java.nio.file.Files](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html)、[Guava](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/io/Files.html)、[Apache common-io](https://commons.apache.org/proper/commons-io/)、[jodd FileUtil](http://jodd.org/api/jodd/io/FileUtil.html) 等开源 I/O 库，虽然这些在 Scala 中也可以使用，但毕竟充满了 Java 风格的 API，用起来并不是那么的纯粹。
>
> 所以，为什么不尝试使用由 Scala 实现的 I/O 库，如：[sbt io](https://github.com/sbt/io)、[Ammonite-Ops](http://www.lihaoyi.com/Ammonite/#Ammonite-Ops)、[better-files](https://github.com/pathikrit/better-files)呢。
>
> Better-files 就是其中一个轻量，但功能强大的开源 I/O 库，支持丰富的 API 以及 Scala 风格的语法设计。

## 导入依赖
sbt:
```scala
libraryDependencies += "com.github.pathikrit" %% "better-files" % version
```
gradle:
```groovy
compile group: 'com.github.pathikrit', name: 'better-files_2.12', version: version
```
maven:
```xml

<dependency>
  <groupId>com.github.pathikrit</groupId>
  <artifactId>better-files_2.12</artifactId>
  <version>version</version>
</dependency>
```

## 多种方式实例化 better-files

```scala
import better.files._
import better.files.File._
// 如果引起同事的误会，可以修改导入模块的名称
// import better.files.{File => ScalaFile, _}
import java.io.{File => JFile}
val f = File("/User/johndoe/Documents")                     // 使用构造器
val f1: File = file"/User/johndoe/Documents"                // 字符串插值器
val f2: File = "/User/johndoe/Documents".toFile             // 讲一个字符串路径转换为文件
val f3: File = new JFile("/User/johndoe/Documents").toScala // 将一个 java File 对象转换为 better-files 对象
val f4: File = root/"User"/"johndoe"/"Documents"            // 从根目录下查找文件
val f5: File = `~` /"Documents"                             // 与 home / "Documents" 等价
val f6: File = "/User"/"johndoe"/"Documents"                // 使用路径分隔符 DSL
val f7: File = home/"Documents"/"presentations"/`..`        // 使用 `..` 定位到父级目录
```

## 简单文件写入

基本写法

```scala
val file = root/"tmp"/"test.txt"
file.overwrite("hello")           // 覆盖文件
file.appendLine().append("world") // 在末尾添加
```
类 C / Shell 脚本语法
```scala
file < "hello"     // 相当于 file.overwrite("hello")
file << "world"    // 相当于 file.appendLines("world")
```
如果你喜欢从右到左的风格
```scala 
"hello" `>:` file
"world" >>: file
```
甚至流式接口风格
```scala 
(root/"tmp"/"diary.txt")
  .createIfNotExists()
  .appendLine()
  .appendLines("My name is", "Inigo Montoya")
  .moveTo(home/"Documents")
  .renameTo("princess_diary.txt")
  .changeExtensionTo(".md")
  .lines
```

## 转换成 Java Resource APIs

```scala
val resource        : InputStream   = Resource.getAsStream("foo.txt") // 相当于 this.getClass().getResource("foo.txt")
val resourceURL     : java.net.URL  = Resource.getUrl("foo.txt")
val resourceAsStr   : String        = Resource.getAsString("foo.txt")
```
以上 API 也可以通过自定义的 ClassLoader 加载
```scala
val resource2       : InputStream   = Resource.at[MyClass].getAsStream("foo.txt")
```

## 转换为流
以便在不加载到内存的情况下将文件移动到各个地方
```scala
val bytes  : Iterator[Byte]            = file.bytes
val chars  : Iterator[Char]            = file.chars
val lines  : Iterator[String]          = file.lineIterator
```
注意！上述 API 最多可以遍历一次，要多次遍历而不创建新的迭代器实例，请考虑转换为其他集合，比如：file.bytes.toStream（不是 Java 的 Stream）

你也可以将字节/字符流回写到文件
```scala
file.writeBytes(bytes)
file.printLines(lines)
```
tee 函数可以合并多个流
```scala
val s3 = s1 tee s2
s3.printWriter.println(s"Hello world") // 将 s1 与 s2 合并后写入
```
可以为任何读/写操作提供字符集
```scala
val content: String = file.contentAsString  // 默认字符集
import java.nio.charset.Charset             // 自定义字符集
file.contentAsString(charset = Charset.forName("US-ASCII"))
file.write("hello world")(charset = "US-ASCII") // 也可以使用隐式参数提供字符集
```
默认情况下，better-file 在解码 UTF-8 时会正确的处理 BOM，如果想象 jdk 一样不正确的处理 bom，可以考虑以下方式
```scala
file.contentAsString(charset = Charset.forName("UTF-8"))    // 像 JDK 一样处理 UTF-8 (see: JDK-4508058)
```
如果需要输出带有 BOM 的文件，可以使用以下方式：
```scala
file.write("hello world")(charset = UnicodeCharset("UTF-8", writeByteOrderMarkers = true))
```

## 对象序列化/解序列化
```scala
case class Person(name: String, age: Int)
val person = new Person("Chris", 24)
```
写入文件
```scala
file.newOutputStream.asObjectOutputStream.serialize(obj).flush()
```
从文件中读取
```scala
val person2 = file.newInputStream.asObjectInputStream.deserialize[Person]
```
使用自定义的 ClassLoader 载入
```scala
file.newInputStream.asObjectInputStreamUsingClassLoader(classLoader = myClassLoader).deserialize[Person]
```
可以使用下面这两个简化的 API
```scala
val person2: Person = file.writeSerialized(person).readDeserialized[Person]
```
## 与 Java 互操作
可以很轻松的使用 Java 的 API
```scala
val file: File = tmp / "hello.txt"
val javaFile     : java.io.File                 = file.toJava
val uri          : java.net.URI                 = file.uri
val url          : java.net.URL                 = file.url
val reader       : java.io.BufferedReader       = file.newBufferedReader
val outputstream : java.io.OutputStream         = file.newOutputStream
val writer       : java.io.BufferedWriter       = file.newBufferedWriter
val inputstream  : java.io.InputStream          = file.newInputStream
val path         : java.nio.file.Path           = file.path
val fs           : java.nio.file.FileSystem     = file.fileSystem
val channel      : java.nio.channel.FileChannel = file.newFileChannel
val ram          : java.io.RandomAccessFile     = file.newRandomAccess
val fr           : java.io.FileReader           = file.newFileReader
val fw           : java.io.FileWriter           = file.newFileWriter(append = true)
val printer      : java.io.PrintWriter          = file.newPrintWriter
```
better-files 也提供了一些好用的隐式，比如：
```scala
file1.reader > file2.writer       // 管道式文件写入
System.in > file2.out             // 管道式输入输出流
src.pipeTo(sink)                  // 如果你不喜欢使用上面的符号的话
val bytes   : Iterator[Byte]        = inputstream.bytes
val bis     : BufferedInputStream   = inputstream.buffered
val bos     : BufferedOutputStream  = outputstream.buffered
val reader  : InputStreamReader     = inputstream.reader
val writer  : OutputStreamWriter    = outputstream.writer
val printer : PrintWriter           = outputstream.printWriter
val br      : BufferedReader        = reader.buffered
val bw      : BufferedWriter        = writer.buffered
val mm      : MappedByteBuffer      = fileChannel.toMappedByteBuffer
val str     : String                = inputstream.asString  // 从输入流读取字符串
val in      : InputStream           = str.inputStream
val reader  : Reader                = str.reader
val lines   : Seq[String]           = str.lines
```
并支持一些 [JDK 不支持的转换](https://stackoverflow.com/questions/62241/how-to-convert-a-reader-to-inputstream-and-a-writer-to-outputstream)

## 通配符支持
```scala
val dir = "src"/"test"
val matches: Iterator[File] = dir.glob("*.{java,scala}")
// 其等价于下面的代码
dir.listRecursively.filter(f => f.extension == Some(".java") || f.extension == Some(".scala"))
```
你也可以用正则表达式代替通配符
```scala
val matches = dir.globRegex("^\\w*$".r)
```
默认情况下 better-files 的通配符与 JDK 提供的并不相同，因为他始终包含当前路径
```scala
dir.glob("**/*.txt", includePath = false) // JDK 默认
dir.glob("*.txt", includePath = true) // better-files 默认
// 这两种方法是或关系
```
也可以扩展 File.PathMatcherSyntax 以实现自己的匹配模式，例如：
```scala
dir.collectChildren(_.isSymbolicLink) // 从目录中获取全部链接文件
```

## 文件系统操作
提供对：ls, cp, rm, mv, ln, md5, touch, cat 等的支持
```scala
file.touch()
file.delete()     // 与 Java API 不同，该方法可以作用在文件夹上，并递归删除文件夹下全部内容（而 Java API 只支持空文件夹的删除））
file.clear()      // 如果是文件夹，清除下面所有内容；如果是文件，清空文件内容
file.renameTo(newName: String)
file.moveTo(destination)
file.moveToDirectory(destination)
file.copyTo(destination)         // 与 Java API 不同，该方法可以作用在文件夹上，并递归复制文件夹下全部内容
file.copyToDirectory(destination)
file.linkTo(destination)         // 硬连接（即在目录下增加目标文件的目录项，或称：该文件同时存在于多个目录下（但并不是复制），修改他会影响目标文件）
file.symbolicLinkTo(destination) // 软连接（又称为符号链接）（即在当前目录下添加一个目标文件的快捷方式，可以根据快捷方式找到文件，他只是一个文本文件，修改他并不影响目标文件）
file.{checksum, md5, sha1, sha256, sha512, digest}   //同样可以作用在文件夹上
file.setOwner(user: String)      // 修改文件目录属主
file.setGroup(group: String)     // 改变文件所属的组
Seq(file1, file2) `>:` file3     // 相当于 cat file1 file2 > file3 (必须导入 better.files.Dsl.SymbolicOperations)
Seq(file1, file2) >>: file3      // 相当于 cat file1 file2 >> file3 (必须导入 better.files.Dsl.SymbolicOperations)
file.isReadLocked; file.isWriteLocked; file.isLocked // 文件的一些属性
File.numberOfOpenFileDescriptors // 打开的文件描述符数量
```

## 临时文件
创建临时文件/文件夹：
```scala
File.newTemporaryFile()
File.newTemporaryDirectory()
```
上述 API 支持设置前缀，后缀和父文件夹。
这些文件默认不会在 JVM 退出时删除，你需要设置 deleteOnExit 属性使其自动删除

更简洁的替代方法是使用代码块来完成临时文件操作
```scala
for {
  tempFile <- File.temporaryFile()
} doSomething(tempFile) // 当代码块结束时临时文件将被删除，即使中途出现了异常
```
或者使用下面的写法：
```scala
File.usingTemporaryFile() {tempFile =>
  // 处理
} // 当代码块结束时临时文件将被删除，即使中途出现了异常
```
也可以指定某些文件为临时文件
```scala
val foo = File.home / "Downloads" / "foo.txt"
for {
  temp <- foo.toTemporary
} doSomething(temp) // 当代码块结束时临时文件将被删除，即使中途出现了异常
```

## 文件属性
```scala
file.name // 比 java.io.File#getName 方法更简单
file.extension
file.contentType
file.lastModifiedTime // 返回 JSR-310 time 对象
file.owner
file.group
file.isDirectory; file.isSymbolicLink; file.isRegularFile
file.isHidden
file.hide(); file.unhide()
file.isOwnerExecutable; file.isGroupReadable // 等，参见 file.permissions
file.size // 作用在目录上时，会求出整个目录的占用大小
file.posixAttributes; file.dosAttributes // 参见 file.attributes
file.isEmpty      // 如果文件没有内容（或文件夹没有子项）或者文件/夹不存在时 返回真
file.isParentOf; file.isChildOf; file.isSiblingOf; file.siblings
file("dos:system") = true // 修改文件元数据 (类似于 Files.setAttribute)
```
所有上述 API 均支持直接指定 LinkOption
```scala
file.isDirectory(LinkOption.NOFOLLOW_LINKS)
```
或使用 File.LinkOptions 辅助设置
```scala
file.isDirectory(File.LinkOptions.noFollow)
```
对于 chmod 操作：
```scala
import java.nio.file.attribute.PosixFilePermission
file.addPermission(PosixFilePermission.OWNER_EXECUTE)      // chmod +X file
file.removePermission(PosixFilePermission.OWNER_WRITE)     // chmod -w file
assert(file.permissionsAsString == "rw-r--r--")
// 以下内容均等价:
assert(file.permissions contains PosixFilePermission.OWNER_EXECUTE)
assert(file.testPermission(PosixFilePermission.OWNER_EXECUTE))
assert(file.isOwnerExecutable)
```

## 文件比较
`==`操作符为基于路径的比较，`===`操作符为基于内容的比较
```scala
file1 == file2    // 判断是否为同一文件（路径）
file1 === file2   // 判断文件内容是否相同
file1 != file2    // 判断文件是否不是同一文件（路径）
file1 =!= file2   // 判断文件内容是否不同
```

## 文件排序
```scala
val files = myDir.list.toSeq
files.sorted(File.Order.byName)
files.max(File.Order.bySize)
files.min(File.Order.byDepth)
files.max(File.Order.byModificationTime)
files.sorted(File.Order.byDirectoriesFirst)
// 等
```

## ZIP压缩 API
> 你不再需要到 StackOverflow 查询 “在 Scala/Java 中如何压缩/解压文件？”！（原文）

解压:
```scala
val zipFile: File = file"path/to/research.zip"
val research: File = zipFile.unzipTo(destination = home/"Documents"/"research")
```
压缩:
```scala
val zipFile: File = directory.zipTo(destination = home/"Desktop"/"toEmail.zip")
```
压缩到（已有压缩包）:
```scala
val zipFile = File("countries.zip").zipIn(Iterator(file"usa.txt", file"russia.txt"))()
```
压缩/解压缩到临时文件/文件夹:
```scala
val someTempZipFile: File = directory.zip()
val someTempDir: File = someTempZipFile.unzip()
assert(directory === someTempDir)
```
同时支持 GZIP
```scala
File("big-data.csv").gzipTo(File("big-data.csv.gz"))
File("big-data.csv.gz").unGzipTo(File("big-data.csv"))
```
GZIP 流操作:
```scala
File("countries.gz").newInputStream.asGzipInputStream().lines.take(10).foreach(println)
def write(out: OutputStream, countries: Seq[String]) =
  out.asGzipOutputStream().printWriter().printLines(countries).close()
```

## UNIX DSL
以上的内容也可以通过类似 UNIX Shell 的方式操作：
```scala
import better.files.Dsl._   // 必须导入 Dsl._ 来开启这些功能

pwd / cwd // 当前目录（classpath）
cp(file1, file2)
mv(file1, file2)
rm(file) /*或者*/ del(file)
ls(file) /*或者*/ dir(file)
ln(file1, file2)   //硬链接
ln_s(file1, file2) //软链接（符号链接）
cat(file1)
cat(file1) >>: file
touch(file)
mkdir(file)
mkdirs(file) // mkdir -p
chown(owner, file)
chgrp(owner, file)
chmod_+(permission, files) // 添加权限
chmod_-(permission, files) // 删除权限
md5(file); sha1(file); sha256(file); sha512(file)
unzip(zipFile)(targetDir)
zip(file*)(targetZipFile)
ungzip(gzipFile)(targetFile)
gzip(file)(targetGZipFile)
```

## 轻量级 ARM（自动化资源管理）
自动关闭的 Java closeable 对象
```scala
for {
  in <- file1.newInputStream.autoClosed
  out <- file2.newOutputStream.autoClosed
} in.pipeTo(out) // 当代码块结束时，输入输出流会自动关闭
```
better-files 为所有的 Java closeable 对象提供了方便的托管版本
```scala
for {
  reader <- file.newBufferedReader.autoClosed
} foo(reader)
```
也可以写成：
```scala
for {
  reader <- file.bufferedReader // 返回 Dispose[BufferedReader]
} foo(reader)
```
或者：
```scala
file.bufferedReader.foreach(foo)
```
同样的
```scala
for {
  reader <- file.bufferedReader
} yield foo(reader)
```
或者
```scala
file.bufferedReader.map(foo).get()
```
甚至
```scala
file.bufferedReader.apply(foo)
```
如果 foo 这个对象本身是 lazy 的，切取决于 reader 是否打开，你需要使用 flatmap 代替 apply：
```scala
def lines(reader: BufferedReader): Iterator[String] = ???
for {
  reader <- file.bufferedReader
  line <- lines(reader)
} yield line
```
或者：
```scala
file.bufferedReader.flatMap(lines)
```
还可以定义自己的自定义可支配资源
```scala
trait Shutdownable {
  def shutdown(): Unit = ()
}

object Shutdownable {
  implicit val disposable: Disposable[Shutdownable] = Disposable(_.shutdown())
}

val s: Shutdownable = ....

for {
  instance <- new Dispose(s)
} doSomething(s) // s 在代码块结束后将被丢弃
```

## Scanner API
尽管 java.util.Scanner 有丰富的 API，但它只支持解析基本数据类型，并且出了名的慢，因为它使用正则表达式， 而且拥有 Java 的一贯作风：它会返回 null 或抛出异常

better-files 提供更快，更丰富，更安全，更范用且可组合的 Scala 替代版本，它不使用正则，允许窥视，访问行号，尽可能返回 Opinion 对象并可让用户自己选择解析器
```scala
val f1 = File("/tmp/temp.txt")
val data = f1.overwrite(s"""Hello World
                           | 1 true
                           | 2 3
                           | """.stripMargin)
val scanner: Scanner = data.newScanner()

assert(scanner.next[String] == "Hello")
assert(scanner.lineNumber == 1)
assert(scanner.next[String] == "World")
assert(scanner.next[(Int, Boolean)] == (1, true))
assert(scanner.nextLine() == " 2 3")
assert(!scanner.hasNext)
// 如果你对 tokens 感兴趣，可以使用 file.tokens()
```
定制你的 Scanner
```scala
sealed trait Animal
case class Dog(name: String) extends Animal
case class Cat(name: String) extends Animal

implicit val animalParser: Scannable[Animal] = Scannable {scanner =>
  val name = scanner.next[String]
  if (name == "Garfield") Cat(name) else Dog(name)
}

val scanner = file.newScanner()
println(scanner.next[Animal])
```
[shapeless-scanner](https://github.com/milessabin/shapeless) 允许你扫描 HList
```scala
val in = Scanner("""
                12 Bob True
                13 Mary False
                26 Rick True
                 """)
import shapeless._
type Row = Int :: String :: Boolean :: HNil
val out = Seq.fill(3)(in.next[Row])
assert(out == Seq(
  12 :: "Bob" :: true :: HNil,
  13 :: "Mary" :: false :: HNil,
  26 :: "Rick" :: true :: HNil
))
```
或者模式类
```scala
case class Person(id: Int, name: String, isMale: Boolean)
val out2 = Seq.fill(3)(in.next[Person])
```
一个扫描 CSV 文件的例子
```scala
val file = """
          23,foo
          42,bar
           """
val csvScanner = file.newScanner(StringSpliiter.on(','))
csvScanner.next[Int]    //=> 23
csvScanner.next[String] //=> foo
```

## 文件监控
Java 中文件监控例子：
```scala
import java.nio.file.{StandardWatchEventKinds => EventType}
val service: java.nio.file.WatchService = myDir.newWatchService
myDir.register(service, events = Seq(EventType.ENTRY_CREATE, EventType.ENTRY_DELETE))
```
上面的 API 使用起来很麻烦（涉及大量的类型转换和空检查），基于阻塞轮询的模型（耗费 CPU），不允许递归监控目录，在监控一般文件时需要编写大量样板文件

better-files 使用了简单的抽象解决了上面的问题
```scala
val watcher = new FileMonitor(myDir, recursive = true) {
  override def onCreate(file: File, count: Int) = println(s"$file got created")
  override def onModify(file: File, count: Int) = println(s"$file got modified $count times")
  override def onDelete(file: File, count: Int) = println(s"$file got deleted")
}
watcher.start()
```
一般情况下，比起重写三个方法，更方便的是直接重写调度程序本身
```scala
import java.nio.file.{Path, StandardWatchEventKinds => EventType, WatchEvent}
val watcher = new FileMonitor(myDir, recursive = true) {
  override def onEvent(eventType: WatchEvent.Kind[Path], file: File, count: Int) = eventType match {
    case EventType.ENTRY_CREATE => println(s"$file got created")
    case EventType.ENTRY_MODIFY => println(s"$file got modified $count")
    case EventType.ENTRY_DELETE => println(s"$file got deleted")
  }
}
```
还有一个专门提供高性能的文件监控和更好的文件插值的[外部模块](https://github.com/gmethvin/directory-watcher#better-files-integration-scala)

## 基于 AKKA 的文件监控
 better-files还提供了一个强大而简洁的基于Akka角色的响应式文件监视程序，该程序支持动态分派:
```scala
import akka.actor.{ActorRef, ActorSystem}
import better.files._, FileWatcher._
implicit val system = ActorSystem("mySystem")
val watcher: ActorRef = (home/"Downloads").newWatcher(recursive = true)
```
为事件注册偏函数
```scala
watcher ! on(EventType.ENTRY_DELETE) {
  case file if file.isDirectory => println(s"$file got deleted")
}
```
监控多个事件
```scala
watcher ! when(events = EventType.ENTRY_CREATE, EventType.ENTRY_MODIFY) {
  case (EventType.ENTRY_CREATE, file, count) => println(s"$file got created")
  case (EventType.ENTRY_MODIFY, file, count) => println(s"$file got modified $count times")
}
```
## 性能跑分
```scala
> sbt "core/testOnly better.files.benchmarks.*"
JavaScanner              : 2191 ms
StringBuilderScanner     : 1325 ms
CharBufferScanner        : 1117 ms
StreamingScanner         :  212 ms
IterableScanner          :  365 ms
IteratorScanner          :  297 ms
BetterFilesScanner       :  272 ms
ArrayBufferScanner       :  220 ms
FastJavaIOScanner2       :  181 ms
FastJavaIOScanner        :  179 ms
```
EOF
