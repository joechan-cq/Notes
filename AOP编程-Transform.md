# AOP编程-Transform

Android代码在Android Studio上编译时的大致流程：

**Java -> Class字节码 -> Dex分包 -> 打包APK**

在Android Studio上可以通过编写Gradle插件，在代码编译期间，进行一些修改处理，这里的`Transform`就是其中一种方式，它是在编译出Class字节码之后工作，可以对Class字节码做出一些修改。

### 建立工程

使用Android Studio创建一个普通的工程。然后创建`buildSrc`目录及内部文件：

--app

--buildSrc (Gradle插件目录)

​    --src

​        --main

​            --resources (插件申明定义)

​                --META-INF

​                    --gradle-plugins

​                        --{PACKAGE_NAME}.properties (定义文件，{PACKAGE_NAME}为包名，根据需要修改)

​            --groovy (java/kotlin，根据插件编写语言决定)

​    --build.gradle

创建完成后，Android Studio会自动识别`buildSrc`文件夹，作为插件文件夹。

在`build.gradle `中填入内容：

```groovy
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation 'com.android.tools.build:gradle:4.1.3'
}
```

### 创建Gradle插件

在`groovy`目录下创建包名目录，并创建插件文件`TestPlugin.groovy`，编写`TestPlugin.groovy`：

```groovy
class TestPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
    }
}
```

在`gralde-plugins`目录的`{PACKAGE_NAME}.properties`文件中填写以下内容：

```properties
implementation-class={PACKAGE_NAME}.TestPlugin
```

然后在主工程的`build.grale`中使用该插件

```groovy
apply plugin: {PACKAGE_NAME}.TestPlugin
```

### 编写Transform类

创建`TestTransform.groovy`文件，继承`Transform`类：

```groovy
class TestTransform extends Transform {

    @Override
    String getName() {
        return 'TestTransform'
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.PROJECT_ONLY
    }

    @Override
    boolean isIncremental() {
        return true
    }

    @Override
    void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs, TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
        super.transform(context, inputs, referencedInputs, outputProvider, isIncremental)
    }
}
```

* getName
  
  这个Transform的名称

* getInputTypes
  
  这个Transform需要处理的文件类型，一般为`CONTENT_CLASS`字节码，也可以使用`CONTENT_RESOURCES`处理资源。

* getScopes
  
  Transform需要处理的项目范围。`PROJECT_ONLY`表示只在引用插件的工程中执行，不会影响依赖的子Module。如需处理所有工程，则使用`SCOPE_FULL_PROJECT`。

* isIncremental
  
  这个Transform是否支持增量编译。简单处理的话，就`return false`不支持即可。

* transform
  
  这个就是实际进行字节码修改的方法。

在`TestPlugin`中注册该`Transform`：

```groovy
 @Override
 void apply(Project project) {
     def android = project.extensions.findByName('android')
     android.registerTransform(new TestTransform())
 }
```

### transform方法实现

先来简单介绍下这个方法中的参数含义：

##### inputs

文件输入源，分为两种：目录和Jar包。**目录**其实就是插件所在工程中的自己的Java源码编译后的字节码保存的地方，**Jar**包则是依赖引入的三方Jar包。

##### referencedInputs

引用型Input，不要对其做transform，因为这些input并不会打包。

##### outputProvider

输出管理类，用于管理transform之后字节码文件的保存输出。

##### isIncremental

当前这次，是否是增量更新

#### 基础实现

一旦接入`Transform`后，`transform`方法中一定要有输出。也就是说一定要将`inputs`通过`outputProvider`进行输出，如果`transform`中什么都不做的话，直接编译打包，将会发现，所有的代码、引入的Jar包，最终都没有打包到APK中。所以`transform`方法的最低限度，要实现如下代码：

```groovy
    @Override
    void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs, TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
        inputs.each { it ->
            it.directoryInputs.each { dir ->
                File destDir = outputProvider.getContentLocation(dir.name, dir.contentTypes, dir.scopes, Format.DIRECTORY)
                FileUtils.copyDirectory(dir.file, destDir)
            }
            it.jarInputs.each { jar ->
                File jarDest = outputProvider.getContentLocation(jar.name, jar.contentTypes, jar.scopes, Format.JAR)
                FileUtils.copyFile(jar.file, jarDest)
            }
        }
    }
```

通过`outputProvider.getContentLocation`方法，获取input对应的输出路径。将`目录`的input，直接输出到对应的目录中，将`Jar`的input，直接输出到对应的文件地址上。这个就是什么都不修改，将input直接output的做法。此时再进行编译打包，然后查看`build/intermediates/transforms/TestTransform/debug`目录下，就能看到一堆文件夹和Jar文件。这些就是使用`FileUtils`copy过来的输出。

#### 字节码修改

有了上述的基础实现后，我们就能遍历这些input，然后找到我们需要修改的class文件或jar包，然后借助`ASM`或`javassist`三方库来对字节码文件进行修改。这里不做具体的使用方法，仅仅贴出`javassist`的教程地址:

[javassist tutorial](http://www.javassist.org/tutorial/tutorial.html)

需要说明的是对于Jar包内字节码的修改，需要将Jar包解压缩出来，修改其中具体的字节码后，重新打包成Jar包作为输出才行。下面会给出Jar文件解压缩和将文件夹压缩成Jar的方法：

```groovy
static void unZip(File file, File outDir) {
  JarFile jarFile = new JarFile(file)
        Enumeration enumeration = jarFile.entries()
        while (enumeration.hasMoreElements()) {
            JarEntry jarEntry = (JarEntry) enumeration.nextElement()
            String entryName = jarEntry.getName()
            if (!jarEntry.directory) {
                String outFileName = outDir.absolutePath + "/" + entryName
                File outEntryFile = new File(outFileName)
                outEntryFile.getParentFile().mkdirs()
                InputStream input = jarFile.getInputStream(jarEntry)
                FileOutputStream output = new FileOutputStream(outEntryFile)
                output << input
                output.close()
                input.close()
            }
        }
        jarFile.close()
}


/**
 * 将某一个目录压缩成jar包
 * @param packagePath 需要压缩的目录
 * @param dstPath 压缩成的jar包的路径
 * @param deleteSrc 是否在压缩完成后删除源文件目录
 */
static void zipDir(File dirFile, File dstFile, boolean deleteSrc) {
    JarOutputStream jarOutputStream = new JarOutputStream(new FileOutputStream(dstFile))
    dirFile.eachFileRecurse { File f ->
        String entryName = f.absolutePath.substring(dirFile.absolutePath.length() + 1)
        ZipEntry entry
        if (f.directory) {
            entry = new ZipEntry(entryName + "/")
        } else {
            entry = new ZipEntry(entryName)
        }
        jarOutputStream.putNextEntry(entry)
        if (!f.directory) {
            InputStream inputStream = new FileInputStream(f)
            jarOutputStream << inputStream
            inputStream.close()
        }
    }
    jarOutputStream.close()
    if (deleteSrc) {
        dirFile.deleteDir()
    }
}
```

### 插件发布

Gradle Plugin编写完成后，一般是要发布到maven仓库，让别人也能够使用的。这个发布就和普通的库发布一样，下面给出发布的gradle task：

```groovy
//maven插件，在Gradle7.0上被移除，不能使用了。所以这里使用7.0以下的Gradle版本
apply plugin: 'maven'

// 判断版本是Release or Snapshots
static def isReleaseBuild(String version) {
    return !version.contains('SNAPSHOT')
}
// 获取仓库url
static def getRepositoryUrl(String version) {
    def MAVEN_RELEASE_URL = 'http://114.55.125.195:8081/repository/maven-releases/'
    def MAVEN_SNAPSHOT_URL = 'http://114.55.125.195:8081/repository/maven-snapshots/'
    return isReleaseBuild(version) ? MAVEN_RELEASE_URL : MAVEN_SNAPSHOT_URL
}

def userName = 'xxxx'
def pwd = 'xxxxx'
def VERSION = '1.0'
def artifactId = 'test_plugin'
def group = 'plugins'

uploadArchives {
    configuration = configurations.archives
    repositories {
        mavenDeployer {
            def NAME = userName
            def PASSWORD = pwd

            pom.version = VERSION
            pom.artifactId = artifactId
            pom.groupId = group

            repository(url: getRepositoryUrl(pom.version)) {
                authentication(userName: NAME, password: PASSWORD) // maven授权信息
            }
        }
    }
}
```

## PS

groovy中可以使用`println`方法来打印日志。
