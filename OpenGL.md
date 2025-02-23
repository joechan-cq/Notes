# OpenGL 笔记

## Android下创建OpenGL环境

### 使用GLSurfaceView

**1. 继承GLSurfaceView**

``` java
public class MyGLSurfaceView extends GLSurfaceView {
    private final MyGLRenderer mRenderer;
 
    public MyGLSurfaceView(Context context) {
        super(context);
        // Create an OpenGL ES 2.0 context
        setEGLContextClientVersion(2);
        mRenderer = new MyGLRenderer();
        setRenderer(mRenderer);
    }
}
```

**2. 实现GLSurfaceView.Renderer接口**

``` java
public class MyGLRenderer implements GLSurfaceView.Renderer {
    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        // 设置清除屏幕所用的颜色
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        // 清除屏幕
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        // 设置视口
        GLES20.glViewport(0, 0, width, height);
    }
}
```

在自定义Render中就拥有了OpenGL环境，即可使用OpenGL进行开发。

### 手动创建EGL环境

EGL是一个接口层，用于连接渲染API（例如OpenGL ES）和本地窗口系统。

创建一个EGL环境，需要顺序调用以下的API：

1. eglGetDisplay
2. eglInitialize
3. eglChooseConfig
4. eglCreateContext
5. eglCreateWindowSurface或eglCreatePBufferSurface或eglCreatePixmapSurface
6. eglMakeCurrent
7. eglSwapBuffers

#### eglGetDisplay

用户获取屏幕显示设备句柄

```  java
EGLDisplay display = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
```

#### eglInitialize

初始化EGL

``` java
int[] version = new int[2]; //用于存储获取到的EGL版本，[0]主版本号，[1]次版本号
EGL14.eglInitialize(display, version, 0, version, 1);
```

#### eglChooseConfig

根据一些配置参数，获取当前EGL所支持的最佳配置

``` java
int[] configAttribs = {
    EGL14.EGL_RED_SIZE, 8,
    EGL14.EGL_GREEN_SIZE, 8,
    EGL14.EGL_BLUE_SIZE, 8,
    EGL14.EGL_ALPHA_SIZE, 8,
    EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,
    EGL14.EGL_NONE
}; //基础配置参数
EGLConfig[] configs = new EGLConfig[1];
int[] numConfigs = new int[1];
//获取满足基础配置参数的具体配置
EGL14.eglChooseConfig(display, configAttribs, 0, configs, 0, configs.length, numConfigs, 0);
EGLConfig eglConfig = configs[0];
```

#### eglCreateContext

创建EGL上下文。如果需要实现EGL线程之间资源共享，那么在创建上下文时，就需要传入其它需要共享的上下文。

``` java
int[] contextAttribs = {
    EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
    EGL14.EGL_NONE
};
EGLContext eglContext = EGL14.eglCreateContext(display, eglConfig, EGL14.EGL_NO_CONTEXT, contextAttribs, 0);
```

#### eglCreateWindowSurface

创建渲染数据载体，这里用windowSurface做例子，根据不同用户，也会用到其他的。

``` java
Surface surface = ...; // 获取或创建一个 Android Surface
int[] surfaceAttribs = {
    EGL14.EGL_NONE
};
// 创建一个 EGLSurface，关联到一个 Android Surface。
EGLSurface eglSurface = EGL14.eglCreateWindowSurface(display, eglConfig, surface, surfaceAttribs, 0);
```

这里就是将其他地方获取的surface关联到EGLSurface上去，这样EGL所绘制的内容也就能输出到这个surface上去了。

#### eglMakeCurrent

到上一步为止，EGL环境就已经配置完毕了。这个`eglMakeCurrent`是在渲染前，将`EGLSurface`、`EGLContext`、`EGLDisplay`进行绑定的。相当于告诉EGL，我接下来要在这些东西上面进行绘制和输出。

#### eglSwapBuffers

EGL的工作模式是**双缓冲**的，所以代码所绘制的内容，并不会直接到屏幕上去，而是在一个缓冲里，需要调用该方法，才能将缓冲里的内容，正式输出到屏幕上去。

## 着色器内建变量及含义

##### 顶点着色器内建变量

| 内建变量       | 作用                                                         | 精度  |
| -------------- | ------------------------------------------------------------ | ----- |
| gl_VertexID    | 当前被处理的顶点的索引                                       | highp |
| gl_InstanceID  | 当前被渲染的实例编号                                         | highp |
| gl_Position    | 输出顶点位置(如果在 vertex shader 中没有写入 gl_Position，那么它的值是未定义的) | highp |
| gl_PointSize   | 指定栅格化点的直径                                           | highp |
| gl_FrontFacing | 内建只读变量,根据顶点位置和图元类型自动生成,如果属于一个当前图元，那么这个值就为true |       |

##### 片元着色器内建变量

| 内建变量       | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| gl_FragCoord   | 内建只读变量，它保存了片元相对窗口的坐标位置：x, y, z, 1/w处理的顶点的索引 |
| gl_FrontFacing | 内建只读变量，如果当前片段是正向面的一部分那么就是true，否则就是false |
| gl_PointCoord  | 内建只读变量，它的值是当前片元所在**点图元**的二维坐标       |
| gl_FragDepth   | 内建，深度值                                                 |

## 如何实现对GLSurfaceView的录制(FBO)

使用FBO离屏渲染的方式，将所有绘制操作，都先会知道离屏缓存中去，然后将这份缓存，画一份到surfaceview上，再画一份到mediarecorder的surface上就可以。 

FBO的实现原理，就是将Surface的输出和一块纹理进行绑定，这块纹理使用FrameBuffer进行数据存储。那么后续Surface的输出，就会保存到这个FrameBuffer中。等内容渲染结束后，就可以去处理这一个纹理。

```java
//fbo的创建 (缓存)   Frame Buffer Object（FBO）即为帧缓冲对象，用于离屏渲染缓冲
//1、创建fbo （离屏屏幕）
mFrameBuffers = new int[1];
// 1、创建几个fbo 2、保存fbo id的数据 3、从这个数组的第几个开始保存
GLES20.glGenFramebuffers(mFrameBuffers.length, mFrameBuffers, 0);

//2、创建属于fbo的纹理
mFrameBufferTextures = new int[1]; //用来记录纹理id
//创建纹理
OpenGLUtils.glGenTextures(mFrameBufferTextures);

//将缓存和纹理进行绑定

//创建一个 2d的图像
// 目标 2d纹理+等级 + 格式 +宽、高+ 格式 + 数据类型(byte) + 像素数据
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mFrameBufferTextures[0]);
GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGBA, width, height, 0, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, null);
// 让fbo与纹理绑定起来 ， 后续的操作就是在操作fbo与这个纹理上了
GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, mFrameBuffers[0]);
GLES20.glFramebufferTexture2D(GLES20.GL_FRAMEBUFFER, GLES20.GL_COLOR_ATTACHMENT0, GLES20.GL_TEXTURE_2D, mFrameBufferTextures[0], 0);

//重置（这一步不会影响已经绑定好的buffer和texture的关系，只是移除与当前环境的绑定）
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
```

**使用FBO**

```java
//启动离屏渲染
//设置显示窗口
GLES20.glViewport(0, 0, mWidth, mHeight);
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mFrameBufferTextures[0]);
GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, mFrameBuffers[0]);
//渲染到离屏缓存中
_draw(gl);
//结束离屏渲染
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
```

## 如何使用OpenGL绘制Bitmap

OpenGL无法直接绘制Bitmap，需要先将Bitmap转化成纹理，然后使用OpenGL绘制纹理。代码如下：

```java
public class BitmapDrawer extends BaseDrawer {

    /**
     * 带有vPosition和aCoord的顶点着色器。
     */
    public static final String COORD_VERTEX_SHADER_CODE = "" +
            "attribute vec4 vPosition;\n" +
            "attribute vec2 vCoord;\n" +
            "varying vec2 aCoord;\n" +
            "void main(){\n" +
            "    gl_Position = vPosition;\n" +
            "    aCoord = vCoord;\n" +
            "}";

    /**
     * 用于渲染单层texture的片元着色器,
     * 需要和{@link #COORD_VERTEX_SHADER_CODE}配合使用
     */
    public static final String TEXTURE_FRAGMENT_SHADER_CODE = "" +
            "precision mediump float;\n" +
            "varying mediump vec2 aCoord;\n" +
            "uniform sampler2D vTexture;\n" +
            "void main() {\n" +
            "     lowp vec4 textureColor = texture2D(vTexture, aCoord);\n" +
            "     gl_FragColor = textureColor;\n" +
            "}";

    private BitmapMaterial mBitmap;
    private Integer imageTextureId;
    //位置
    private FloatBuffer mGLVertexBuffer;
    //纹理
    private FloatBuffer mGLTextureBuffer;

    public BitmapDrawer(@NonNull Bitmap bitmap) {
        super(COORD_VERTEX_SHADER_CODE, TEXTURE_FRAGMENT_SHADER_CODE);
        mBitmap = bitmap;
        // 4个点 x，y = 4*2 float 4字节 所以 4*2*4
        mGLVertexBuffer = ByteBuffer.allocateDirect(4 * 2 * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        mGLVertexBuffer.clear();
        // 参考OpenGL的坐标系
        float[] VERTEX = {
                -1.0f, -1.0f, //左下
                1.0f, -1.0f,  //右下
                -1.0f, 1.0f, //左上
                1.0f, 1.0f   //右上
        };
        mGLVertexBuffer.put(VERTEX);
        mGLTextureBuffer = ByteBuffer.allocateDirect(4 * 2 * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        mGLTextureBuffer.clear();

        //需要使用这个矩阵来修正上下，否则图片是颠倒的
        float[] TEXTURE = {
                0.0f, 1.0f,
                1.0f, 1.0f,
                0.0f, 0.0f,
                1.0f, 0.0f,
        };
        mGLTextureBuffer.put(TEXTURE);
    }

    @Override
    public void onSurfaceCreated() {
        super.onSurfaceCreated();
        if (imageTextureId == null) {
            imageTextureId = OpenGLUtils.createTexture(mBitmap);
        }
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        super.onSurfaceChanged(gl, width, height);
        mWidth = width;
        mHeight = height;
    }

    @Override
    protected void realDraw(GL10 gl, int[] args) {
        super.realDraw(gl, args);
        if (imageTextureId == null) {
            return;
        }
        GLES20.glViewport(mWidth, mHeight);
        GLES20.glUseProgram(mProgramId);

        //传递坐标
        GLES20.glEnableVertexAttribArray(vPosition);
        mGLVertexBuffer.position(0);
        GLES20.glVertexAttribPointer(vPosition, 2, GLES20.GL_FLOAT, false, 0, mGLVertexBuffer);

        GLES20.glEnableVertexAttribArray(vCoord);
        mGLTextureBuffer.position(0);
        GLES20.glVertexAttribPointer(vCoord, 2, GLES20.GL_FLOAT, false, 0, mGLTextureBuffer);

        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, imageTextureId);
        GLES20.glUniform1i(vTexture, 0);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);

        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);

        GLES20.glDisableVertexAttribArray(vPosition);
        GLES20.glDisableVertexAttribArray(vCoord);
    }
}
```

OpenGL绘制图片时，有两个区域需要指定，一个是绘制Bitmap的哪个区域（a），还有一个是将区域a会知道Surface的哪个区域中去。分别对应上述代码中的`TEXTURE`和`VERTEX`。

## OpenGL上下文共享（纹理共享）

EGLContext是OpenGL的上下文环境，一个线程只能有一个OpenGL的上下文。一般来讲，使用不同上下文之间互相隔离，资源不共享。因此采用MediaRecorder+Camera+GLSurface方式录制特效视频时，需要采用共享上下文的方式，才能将绘制到屏幕上的数据，同时发送给MediaRecorder。

共享上下文的方式也非常简单，例如上下文A、B，只需要在创建B时，传入A作为sharedContext参数即可。

```java
B = EGL14.eglCreateContext(mEglDisplay, mEglConfig, A, ctx_attrib_list, 0);
```

## 纹理Texture的使用

纹理Texture的使用，一般分为创建纹理+填充纹理→激活纹理单元→绑定纹理单元→shader处理纹理。

**创建纹理**

```java
int[] textureHandles = new int[1];
// 1.生成纹理（1表示创建几个纹理，textureHandles为接收纹理id的容器）
GLES20.glGenTextures(1, textureHandles, 0);
// 2.绑定纹理
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureHandles[0]);
// 3.设置纹理属性
// 设置纹理的缩小过滤类型（1.GL_NEAREST ; 2.GL_LINEAR）
GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_NEAREST);
// 设置纹理的放大过滤类型（1.GL_NEAREST ; 2.GL_LINEAR）
GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_NEAREST);
// 设置纹理的X方向边缘环绕
GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_REPEAT);
// 设置纹理的Y方向边缘环绕
GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_REPEAT);
```

**填充纹理**

一般来说，是把bitmap绑定到纹理上

```java
//在调用这个之前，一定要先使用glBindTexture，将纹理进行绑定
android.opengl.GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bitmap, 0);
//解绑纹理单元
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
```

如果不是bitmap，而是像素数据，则使用如下代码

```java
GLES20.glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width,height, 0, GL_RGB, GL_UNSIGNED_BYTE, pData);
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
```

**使用纹理**

激活+绑定+shader处理

```java
//激活纹理单元（未必GL_TEXTURE1）
GLES20.glActiveTexture(GLES20.GL_TEXTURE1);
//绑定纹理
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureHandles[0]);
//将纹理单元设置给vTexture，vTexture为shader中的uniform sampler2D vTexture;
//这里的参数1，对应GL_TEXTURE1，如果是0，对应GL_TEXTURE0，2对应GL_TEXTURE2
GLES20.glUniform1i(vTexture, 1);
```

## PS.

* OpenGL部件一般由OpenGL程序+着色器组成。

* 着色器中必须要知道，分为处理坐标的**顶点着色器**和处理颜色的**片元着色器**。

* 使用顶点着色器接收顶点坐标（FloatBuffer）时，注意指针置0：`floatBuffer.position(0);`，不然只有第一次能读取到坐标。

* 必须创建和使用**多个openGL程序**来进行不同图层的绘制，否则一些变量对象的变化，会影响之前的绘制。

* OpenGL的坐标，是以绘制区域中心为原点，上、右为正方向，范围为[-1, -1, 1 ,1]的坐标轴。 
