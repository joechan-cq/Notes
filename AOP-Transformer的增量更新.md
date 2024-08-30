# Transformer的增量更新

上节简单介绍了如何去实现Transformer，这节将会讲一下如何让Transformer去支持增量更新。

### 启用增量更新

首先要告知Gradle，这个Transformer支持增量更新。

```groovy
@Override
boolean isIncremental() {
    return true
}
```

返回true即可。

### 实现增量更新

之前不支持增量更新的时候，Transformer的实现中，都是直接将`DirectoryInput`和`JarInput`直接拷贝到指定的输出目录下的。而增量更新，会告知Transformer每个文件的变更状态，一共有四种：NOTCHANGED、CHANGED、ADDED、REMOVED。只需要针对这四种状态做好处理就可以了。简单的来讲：

* NOTCHANGED —— 不要做任何处理
* CHANGED、ADDED —— 重新拷贝，覆盖已有文件
* REMOVED —— 删除已有文件

#### JarInput

针对`JarInput`而言， 直接按照下面的写法进行处理即可：

```groovy
inputs.each { input ->
    input.jarInputs.each { it ->
    File destJar = outputProvider.getContentLocation(it.name, it.contentTypes, it.scopes, Format.JAR)
      if (isIncremental) {
      switch (it.status) {
        case Status.NOTCHANGED:
            //不用做任何处理
            break;
        case Status.CAHNGED:
        case Status.ADDED:
            //如果Jar文件有修改或是新增的Jar文件，则拷贝覆盖到指定位置
            FileUtils.copyFile(it.file, destJar)
            break;
        case Status.REMOVED:
            //如果是移除的，则删除指定位置上的Jar文件
            FileUtils.delete(destJar)
            break;
      }
    } else {
      //不是增量更新，则直接拷贝jar文件到指定输出位置
      FileUtils.copyFile(it.file, destJar)
    }
  }
}
```

非常的简单易懂。

#### DirectoryInput

而对于`DirectoryInput`的增量处理，也有两种方法，一种省事-性能一般的，一种精确-性能好的。

##### 省事的

因为对于`DirectoryInput`的增量更新，是针对单个文件的，但`DirectoryInput`却是文件夹级别的。所以简单的做法就是只要这个文件夹中内容有改变，就直接重新拷贝整个文件夹。

```groovy
inputs.each { input ->
    input.directoryInputs.each { it ->
    File destDir = outputProvider.getContentLocation(it.name, it.contentTypes, it.scopes, Format.DIRECTORY)
      if (isIncremental) {
      Map<File, Status> changedMap = it.getChangedFiles()
      if (changedMap == null || changedMap.isEmpty()) {
        //说明没有任何改变，则不错任何处理
      } else {
        //如果有改变，则删除原有的目录，重新拷贝
        FileUtils.deleteDirectoryContents(destDir)
        FileUtils.copyDirectory(it.file, destDir)
      }
    } else {
      //不是增量更新，则直接拷贝文件夹到指定输出位置
      FileUtils.copyDirectory(it.file, destDir)
    }
  }
}
```

##### 精确的

精确的，就是针对每个有变化的文件进行单独处理

```groovy
boolean isRelease = context.variantName.toLowerCase().contains('release')
inputs.each { input ->
    input.directoryInputs.each { it ->
    File destDir = outputProvider.getContentLocation(it.name, it.contentTypes, it.scopes, Format.DIRECTORY)
      if (!isIncremental) {
      //不是增量更新，则直接拷贝文件夹到指定输出位置
      FileUtils.copyDirectory(it.file, destDir)
    } else {
      Map<File, Status> changedMap = it.getChangedFiles()
      if (changedMap == null || changedMap.isEmpty()) {
        //说明没有任何改变，则不错任何处理
      } else {
        changedMap.each { entry ->
          //变化的文件
          File srcFile = entry.key
          //状态
          Status status = entry.value

          //因为Transformer的数据来源有两种：
          //1. java原文件直接编译。build/intermediates/javac/debug/classes
          //2. 其他Transformer的结果目录。build/intermediates/transforms/AAAAAA/debug/xxx
          //srcFile的路径中是包含包名目录的，例如
          //build/intermediates/javac/debug/classes/com/dw/btime/App.class
          //而destDir的路径是不包含包名的，例如
          //build/intermediates/transforms/BBBBBB/debug/xxx
          //而最终这个文件需要拷贝到的dstFile路径为
          //build/intermediates/transforms/BBBBBB/debug/xxx/com/dw/btime/App.class
          //所以需要想办法提取包名后，拼接出正确的目标路径
          int debugIndex = srcFile.path.indexOf(isRelease ? "release" : "debug")
          int nextSeparator = srcFile.path.indexOf(File.separator, debugIndex + (isRelease ? 8 : 6))
          String srcFileName = srcFile.path.substring(nextSeparator)
          File dstFile = new File(destDir.getAbsolutePath() + srcFileName)

          switch (status) {
            case Status.NOTCHANGED:
                    //不用做任何处理
                    break;
                case Status.CAHNGED:
                case Status.ADDED:
                //拷贝覆盖到指定位置
                    FileUtils.copyFile(srcFile, dstFile)
                    break;
                case Status.REMOVED:
                //删除指定位置上的文件
                    FileUtils.deletePath(dstFile)
                    break;
          }
        }
      }
    }
  }
}
```
