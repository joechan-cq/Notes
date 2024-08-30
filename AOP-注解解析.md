# AOP-注解解析

注解Annotation是JDK5.0引入的一种注释机制。可以用来修饰类、方法、变量、参数和包，此外重要的是，这个注解，是可以通过反射进行获取的。因为利用注解技术，可以节省非常多的代码，所以注解在非常多的库中得到应用，其中就有著名的`Retrofit`和`ButterKnife`。如下：

```java
@BindView( R2.id.button)  
public Button button;
```

通过注解，自动绑定`button`变量和`R2.id.button`对应的控件，等价于代码`button = findViewById(R2.id.button);`。当然，虽然写的代码中没有写findViewById，但最终编译后的代码，仍然是通过findViewById来实现的。这段代码，就是`注解解析器`解析`@BindView`注解后，编译Java代码的时候，自动生成findViewById的代码插入到最终的class文件中去的。

### 注解Annotation

下面只说明最常用的东西。

#### 定义

注解的基本定义如下：

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.TYPE)
public @interface AName {
    int value() default 0;
}
```

由两个元注解和内容定义组成。

| 元注解        | 说明               | 取值                                                                                                                                                                                                                                                                                                          |
| ---------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| @Target    | 表示该注解可以用在什么地方    | ElementType.ANNOTATION_TYPE 可以应用于注释类型。<br />ElementType.CONSTRUCTOR 可以应用于构造函数。<br />ElementType.FIELD 可以应用于字段或属性。<br />ElementType.LOCAL_VARIABLE 可以应用于局部变量。<br />ElementType.METHOD 可以应用于方法级注释。<br />ElementType.PACKAGE 可以应用于包声明。<br />ElementType.PARAMETER 可以应用于方法的参数。<br />ElementType.TYPE 可以应用于类的定义。 |
| @Retention | 表示需要在什么级别保存该注解信息 | 1.SOURCE:在源文件中有效（即源文件保留）<br />2.CLASS:在class文件中有效（即class保留）<br />3.RUNTIME:在运行时有效（即运行时保留，CLASS类型，一旦APP运行起来，是无法反射获取注解的，RUNTIME可以）                                                                                                                                                                            |

上面的`AName`注解，就是`修饰类定义`，`可以设置类型为int的value变量`，`注解最终会保留在class文件中`。使用如下：

```java
@AName(value = 1)
public class AAA {
}
```

### 注解解析器

AnnotationProcessor，就是用来在编译阶段，解析注解，并做一些操作的插件。像`ButterKnife`之类用到注解的库，都会对应有一个注解解析器，使用的时候，都会在gralde文件中进行引入：`annotationProcessor 'xxx:xxxxxxx:1.0'`。

#### 定义

在Android Studio的项目根目录下创建解析器工程目录为`processor`，同时建立内部文件，目录架构如下：

```
-processor
    -src
        -main
            -java
            -resources
                -META-INF
                    -gradle
                        -incremental.annotation.processors
                    -services
                        -javax.annotation.processing.Processor
    -build.gradle
```

* java文件夹中，存储解析器代码（也可以改成groovy，使用groovy语言进行编写）
* resources文件夹中，声明定义解析器
  * incremental.annotation.processors 声明这个解析器是否支持增量编译
  * javax.annotation.processing.Processor 声明这个解析器的路径
* build.gradle 管理解析器工程的依赖

build.gradle的内容大致如下：

```groovy
apply plugin: 'java-library'

dependencies {
}
repositories {
    google()
    jcenter()
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"
```

#### 自定义解析器

##### 创建解析器

继承`AbstractProcessor`：

```java
class TestProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return false;
    }
}
```

在`javax.annotation.processing.Processor`中指定该解析器：

```
{packagename}.TestProcessor
```

自行填写TestProcessor所在packagename。

##### 声明解析器支持的注解

使用`@SupportedAnnotationTypes`来声明，这个解析器需要处理的注解：

```java
@SupportedAnnotationTypes({"{packagename}.AName"})
class TestProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return false;
    }
}
```

或者重写`AbstractProcessor`的`getSupportedOptions`方法，返回需要处理的注解数组。

##### init方法

```java
@Override
public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    mFiler = processingEnv.getFiler();
    mMessager = processingEnv.getMessager();
    mElementUtils = processingEnv.getElementUtils();
}
```

* mFiler —— 用于创建.java或.class文件
* mMessager —— 用于输出日志
* mElementUtils —— 用户遍历解析被注解修饰的对象

##### process方法

整个方法就是实际处理注解的地方了，常用的写法如下：

```java
 @Override
 public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
         if (set == null || set.isEmpty()) {
            return false;
    }
       Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(AName.class);
         if (elements != null) {
                for (Element e : elements) {
          //获取到注解
          AName aname = e.getAnnotation(AName.class);
          //遍历所有被注解修饰的对象
          if (e instanceof typeElement) {
            //被修饰的是类
          } else if (e instanceof ExecutableElement) {
            //被修饰的是方法
          } else if (e instanceof VariableElement) {
            //被修饰的是变量
          }

          ...//做之后的处理
          }
    }
 }
```

##### 动态生成代码

这部分的实现跟注解其实关系并不大，只是很多库会根据注解来生成代码而已。这里推荐采用Square的`javapoet`库，它是基于`javassist`库实现的，使用起来会比`javassist`更加的简单，也更加容易理解。注意javapoet库一般用于**生成新的类文件，而不是修改现有的类文件**。

在`build.gradle`中引入javapoet:

```groovy
api 'com.squareup:javapoet:1.9.0'
```

其大致的使用方法如下：

```java
//1. 准备类定义
Builder typeBuilder = TypeSpec.classBuilder("AName_Generate");

//2. 构造内部变量
typeBuilder.addField(TypeName.BOOLEAN, "isOk", Modifier.PRIVATE);

//3. 构造方法
MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder("init").returns(void.class).addStatement("System.out.println(123456)");
typeBuilder.addMethod(methodBuilder.build());

//4. 创建文件
JavaFile.Builder builder = JavaFile.builder("com.dw.temp", typeBuilder.build());
JavaFile javaFile = builder.build();
try {
    javaFile.writeTo(mFiler);
} catch (IOException e) {
    e.printStackTrace();
}
```

动态生成的代码，能够在{project}/build/generated/ap_generated_sources/中看到，最终参与打包。
