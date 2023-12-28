### Java 有几种文件拷贝方式？哪一种最高效？

### 典型回答

Java 有多种比较典型的文件拷贝实现方式，

比如：利用 java.io 类库，直接为源文件构建一个 FileInputStream 读取，然后再为目标文件构建一个 FileOutputStream，完成写入工作。

```Java
public static void copyFileByStream(File source, File dest) throws
        IOException {
    try (InputStream is = new FileInputStream(source);
         OutputStream os = new FileOutputStream(dest);){
        byte[] buffer = new byte[1024];
        int length;
        while ((length = is.read(buffer)) > 0) {
            os.write(buffer, 0, length);
        }
    }
 }
```

或者，利用 java.nio 类库提供的 transferTo 或 transferFrom 方法实现。

```Java
public static void copyFileByChannel(File source, File dest) throws
        IOException {
    try (FileChannel sourceChannel = new FileInputStream(source)
            .getChannel();
         FileChannel targetChannel = new FileOutputStream(dest).getChannel
                 ();){
        for (long count = sourceChannel.size() ;count>0 ;) {
            long transferred = sourceChannel.transferTo(
                    sourceChannel.position(), count, targetChannel);            sourceChannel.position(sourceChannel.position() + transferred);
            count -= transferred;
        }
    }
 }
```

此外，还可以利用 Apache Commons IO 提供的 FileUtils.copyFile 方法实现。

```Java
public static void copyFileWithCommonsIO(File source, File dest) throws
        IOException {
    FileUtils.copyFile(source, dest);
}
```

最后，还可以利用 Java 7 提供的 Files.copy 方法实现。

```Java
public static void copyFileWithJava7(File source, File dest) throws
        IOException {
    Files.copy(source.toPath(), dest.toPath());
}
```

### 知识扩展

Java NIO（Java New IO）库提供了一些类和方法，如FileChannel.transferTo()和FileChannel.transferFrom()，它们可以利用操作系统底层的零拷贝特性来实现文件复制。这些方法可以在适当的条件下使用零拷贝进行数据传输。

因此，如果要实现零拷贝特性的文件复制，可以考虑使用Java NIO库提供的相关方法，而不是Files.copy方法。，是通过利用操作系统底层的 sendfile 系统调用实现的。

sendfile 系统调用是一个高效的文件传输系统调用，它可以在内核空间和用户空间之间传输数据，而不需要在内核空间和用户空间之间复制数据。

因此，利用 sendfile 系统调用可以避免不必要的内存拷贝，从而提高效率。

但是，需要注意的是，sendfile 系统调用只能在 Linux 系统上使用，而且只能用于文件之间的传输，不能用于网络传输。

因此，如果要实现跨平台的零拷贝文件复制，可以考虑使用 Java NIO 库提供的 transferTo 和 transferFrom 方法，它们会根据不同的操作系统和文件系统选择最佳的实现方式。

此外，还需要注意的是，零拷贝特性的实现并不一定比普通的文件复制方式效率更高。

因为，零拷贝特性的实现需要利用操作系统底层的系统调用，这会增加系统调用的次数，从而增加了系统调用的开销。

因此，如果要实现高效的文件复制，还需要考虑到文件的大小、系统的负载、系统的配置等因素。

最后，需要注意的是，零拷贝特性的实现并不一定比普通的文件复制方式更安全。

因为，零拷贝特性的实现需要利用操作系统底层的系统调用，这会增加系统调用的次数，从而增加了系统调用的风险。

因此，如果要实现安全的文件复制，还需要考虑到文件的权限、系统的安全配置等因素。