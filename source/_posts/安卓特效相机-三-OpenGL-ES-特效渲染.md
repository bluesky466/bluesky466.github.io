title: 安卓特效相机(三) OpenGL ES 特效渲染
date: 2019-09-22 11:48:12
tags:
  - 技术相关
  - Android
  - 音视频
---

系列文章:

[安卓特效相机(一) Camera2的使用](http://blog.islinjw.cn/2019/08/27/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%B8%80-Camera2%E7%9A%84%E4%BD%BF%E7%94%A8/)
[安卓特效相机(二) EGL基础](http://blog.islinjw.cn/2019/09/13/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%BA%8C-EGL%E5%9F%BA%E7%A1%80/)
[安卓特效相机(三) OpenGL ES 特效渲染](http://blog.islinjw.cn/2019/09/22/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%B8%89-OpenGL-ES-%E7%89%B9%E6%95%88%E6%B8%B2%E6%9F%93/)
[安卓特效相机(四) 视频录制](http://blog.islinjw.cn/2019/10/09/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E5%9B%9B-%E8%A7%86%E9%A2%91%E5%BD%95%E5%88%B6/)

# 特效的实现原理

接下来这篇文章我们讲下特效的具体实现原理。

由于预览画面的渲染是将Surface传给CameraDevice由它去绘制的,而且我没有找到什么可以接管或者添加渲染效果的接口,所以并不能直接去处理摄像头的画面。

于是这里我们只能用一种游戏中常用的手段去处理,这种手段的名字叫做RTT(render to texture),中文名叫做渲染到纹理。

玩法是先将我们想要处理的画面,不直接绘制到屏幕,而是绘制成一张图片,然后我们再拿这张图片去做一些特殊的处理,或者特殊的用途:

{% img /安卓特效相机三/5.png %}

例如游戏中水面的倒影一种比较古老的实现方法就是先将岸上的画面绘制成一张图片,然后倒过来然后做一些扭曲、模糊、淡化等处理,然后贴到水面上。

又例如下面这种狙击镜的实现原理就是先将摄像头位置调到远处,将远处的画面绘制到一张贴图上,然后将摄像头位置再调回角色处,把刚刚得到的远处的画面的图片直接贴到狙击镜上:

{% img /安卓特效相机三/4.jpeg %}

所以在这个特效相机的例子里面我们的实现原理如下:

{% img /安卓特效相机三/6.png %}

# OpenGL实现

我们使用OpenGL ES 2.0版本,这个版本要求我们用GLSL实现顶点着色器和片元着色器。这两个着色器其实是两个运行在GPU的程序。

GLSL全称是OpenGL Shading Language即OpenGL着色语言,它在语法上和C语言有点像。只是看的话相信大家都能看懂,我就不仔细介绍语法了。

OpenGL可编程渲染管线的整个流程比较复杂,作为初学者我们只要理解其中的顶点着色器和和片元着色器就可以了。简单来讲就是OpenGL会在顶点着色器确定顶点的位置,然后这些顶点连起来就是我们想要的图形。接着在片元着色器里面给这些图形上色:

{% img /安卓特效相机三/1.png %}

我们直接看看两个着色器的代码。

## 顶点着色器

OpenGL会将每个顶点的坐标传递给顶点着色器,我们可以在这里改变顶点的位置。例如我们给每个顶点都加上一个偏移,就能实现整个图形的移动。

在这个demo里面我们不改变顶点的坐标,只是简单的将它从二维转换成四维。现实世界里面都是三维的,那为什么要装换成四维呢?原因是我们可以用4\*4的矩阵对坐标进行旋转、缩放、平移等变换,但是4\*4的矩阵只能和四维向量相乘,所以需要在xyz之外加多一个维度,我们一般情况下直接把这个维度的值设成1就好。然后将计算得到的四维坐标放到gl\_Position作为最终结果值:

```
attribute vec2 vPosition;
attribute vec2 vCoord;

varying vec2 vPreviewCoord;
uniform mat4 matTransform;

void main() {
    gl_Position = vec4(vPosition, 0, 1);
    vPreviewCoord = (matTransform * vec4(vCoord.xy, 0, 1)).xy;
}
```

然后除了vPosition这个顶点的坐标,大家还会看到vCoord,它是纹理坐标。什么是纹理坐标呢?

纹理其实可以理解成图片,我们将图片的左下角定义成原点(0,0),左上角、右上角、右下角分别为(0,1)、(1,1)、(1,0):

{% img /安卓特效相机三/2.png %}

我们的每个顶点,除了携带顶点坐标之外,还携带了纹理坐标的信息,顶点坐标确定了这个图形的形状,而纹理坐标则确定贴图要怎么样贴到这个图形上。然后在片元着色器里面就可以根据这个纹理坐标去给图形贴上贴图了:

{% img /安卓特效相机三/3.png %}

不过看到代码可以看到,我们这里还用matTransform这个矩阵对纹理坐标进行了变换。这里是由于我们的图片不是普通的图片,而是将摄像头的画面画到另外一个surface之后拿过来的,需要进行变换。这块等下再仔细讲解。

## 片元顶点着色器

```
#extension GL_OES_EGL_image_external : require

precision highp float;

varying vec2 vPreviewCoord;

uniform samplerExternalOES texPreview;
uniform mat4 uColorMatrix;

void main() {
    gl_FragColor = uColorMatrix * texture2D(texPreview, vPreviewCoord).rgba;
}
```

片元着色器就比较简单了,第一行是由于我们使用了samplerExternalOES需要开启特殊配置,这个是由于在安卓上我们只能用samplerExternalOES类型的纹理去接收摄像头的画面,而使用samplerExternalOES需要开启GL\_OES\_EGL\_image\_external功能。

然后这个texPreview就是我们摄像头画面绘制成的那张图片了,我们用texture2D这个方法去读取图片某个像素的颜色值,它的第一个参数就是我们的纹理,第二个参数就是我们的纹理坐标,也就是上一步顶点着色器计算的到的纹理坐标:

```
vPreviewCoord = (matTransform * vec4(vCoord.xy, 0, 1)).xy;
```

这里有同学可能会疑问我们在顶点着色器不是只计算了顶点的纹理坐标吗?那图形边上和内部的纹理坐标又是怎么来的呢?

没错顶点着色器只是处理顶点的,有多少个顶点,顶点着色器就会执行多少次,处理完所有的顶点之后,我们将值传给varying类型的变量,OpenGL就会帮我们对varying变量做插值,计算出图像上每个像素对应的纹理坐标,然后每个像素都会调用片元着色器去处理。于是运行完所有像素的片元着色器之后整个图像就显示出来了:

{% img /安卓特效相机三/7.png %}

通过texture2D函数获得这个像素在预览画面对应的颜色值之后我们再用一个特效处理矩阵去和它相乘做特效处理。例如黑白、怀旧、反相的处理就是不同的矩阵去和这个颜色相乘,得到最终显示出来的颜色。

例如一个颜色(r,g,b)反相效果其实就是(1.0\-r, 1.0\-b, 1.0\-f),所以我们可以用这个矩阵去和像素颜色相乘:

```
-1.0f  0.0f  0.0f 1.0f
 0.0f -1.0f  0.0f 1.0f
 0.0f  0.0f -1.0f 1.0f
 0.0f  0.0f  0.0f 1.0f
```

至于原理的话不知道大家记不记得线性代数的知识:

```
-1.0f  0.0f  0.0f 1.0f         r       -r + a
 0.0f -1.0f  0.0f 1.0f    *    g   =   -g + a
 0.0f  0.0f -1.0f 1.0f         b       -b + a
 0.0f  0.0f  0.0f 1.0f         a       a
```

然后我们把alpha通道设置成1,0,就是[1.0-r, 1.0-g, 1.0-b, 1.0]就是(r,g,b,1)的反相颜色了。

其他的效果类似的,我这边列出两个特效矩阵给大家用:

```
// 去色效果矩阵
0.299f 0.587f 0.114f 0.0f
0.299f 0.587f 0.114f 0.0f
0.299f 0.587f 0.114f 0.0f
0.0f   0.0f   0.0f   1.0f

// 怀旧效果矩阵
0.393f 0.769f 0.189f 0.0f
0.349f 0.686f 0.168f 0.0f
0.272f 0.534f 0.131f 0.0f
0.0f   0.0f   0.0f   1.0f
```

## 创建渲染器

我们写好顶点着色器和片元着色器之后要让他们在我们的OpenGL程序里面运行。

我们可以用下面代码创建着色器

```
public int loadShader(int shaderType, InputStream source) {
        // 读取着色器代码
        String sourceStr;
        try {
            sourceStr = readStringFromStream(source);
        } catch (IOException e) {
            throw new RuntimeException("read shaderType " + shaderType + " source failed", e);
        }

        // 创建着色器并且编译
        int shader = GLES20.glCreateShader(shaderType); // 创建着色器程序
        GLES20.glShaderSource(shader, sourceStr); // 加载着色器源码
        GLES20.glCompileShader(shader); // 编译着色器程序

        // 检查编译是否出现异常
        int[] compiled = new int[1];
        GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, compiled, 0);
        if (compiled[0] == 0) {
            String log = GLES20.glGetShaderInfoLog(shader);
            GLES20.glDeleteShader(shader);
            throw new RuntimeException("create shaderType " + shaderType + " failed : " + log);
        }
        return shader;
    }
```

它最关键的其实就是中间这三行代码:

```
int shader = GLES20.glCreateShader(shaderType); // 创建着色器程序
GLES20.glShaderSource(shader, sourceStr); // 加载着色器源码
GLES20.glCompileShader(shader); // 编译着色器程序
```

在GLES20.glCreateShader的时候需要指定着色器类型,顶点着色器(GLES20.GL\_VERTEX\_SHADER)或者片元着色器(GLES20.GL\_FRAGMENT\_SHADER)创建出来的着色器程序需要链接到我们的渲染程序当中:

```
public int createProgram(InputStream vShaderSource, InputStream fShaderSource) {
    // 创建渲染程序
    int program = GLES20.glCreateProgram();
    GLES20.glAttachShader(program, loadShader(GLES20.GL_VERTEX_SHADER, vShaderSource));
    GLES20.glAttachShader(program, loadShader(GLES20.GL_FRAGMENT_SHADER, fShaderSource));
    GLES20.glLinkProgram(program);

    // 检查链接是否出现异常
    int[] linked = new int[1];
    GLES20.glGetProgramiv(program, GLES20.GL_LINK_STATUS, linked, 0);
    if (linked[0] == 0) {
        String log = GLES20.glGetProgramInfoLog(program);
        GLES20.glDeleteProgram(program);
        throw new RuntimeException("link program failed : " + log);
    }
    return program;
}
```


最后调用GLES20.glUseProgram方法使用创建的渲染程序:

```
AssetManager asset = context.getAssets();
try {
    mProgram = createProgram(asset.open(VERTICES_SHADER), asset.open(FRAGMENT_SHADER));
} catch (IOException e) {
    throw new RuntimeException("can't open shader", e);
}
GLES20.glUseProgram(mProgram);
```

## glViewport

这里有个比较重要的方法要先讲一下,GLES20.glViewport定义了视窗的位置。

OpenGL虽然是在Surface上绘制,但我们可以不铺满整个Surface,可以只在它的某部分绘制，例如我们可以用下面代码只用TextureSurface的左下角的四分之一去显示OpenGL的画面:

```
//width、height是TextureView的宽高
GLES20.glViewport(0, 0, width/2, height/2);
```

{% img /安卓特效相机三/8.png %}

当然一般情况下我们都是铺满整个Surface

```
GLES20.glViewport(0, 0, width, height);
```

## 填充顶点信息

从顶点着色器代码来看,我们的顶点携带了两种信息,一个是顶点的坐标、一个是纹理坐标:

```
attribute vec2 vPosition;
attribute vec2 vCoord;
```

在java代码中,glUseProgram之后我们可以这样拿到他们的id:

```
mPositionId = GLES20.glGetAttribLocation(mProgram, "vPosition");
mCoordId = GLES20.glGetAttribLocation(mProgram, "vCoord");
```

然后就可以通过这两个id,去给这两个变量填充值了。那具体要填充些什么值呢?

在OpenGL中,三角形是基本图形,任何的图形都可以由三角形组合而来。我们的TextureView其实是一个矩形,它可以由两个三角形组成。但是这个矩形的坐标应该设置成多少呢?

默认情况下当我们设置一个顶点的x=0,y=0的时候它就在OpenGL画面的中心,x轴正方向在右边,y轴正方向在上边,画面的上下左右分别是y=1、y=-1、x=-1、x=1:

{% img /安卓特效相机三/9.png %}

无论z坐标是多少都会忽略,只会管x,y坐标。有同学可能会疑惑,OpenGL不是可以处理三维图形运算的吗?
没错,但是OpenGL ES 2.0将整个三维运算都交给了我们,我们需要自己去乘观察矩阵和投影矩阵才能得到三维的效果,这块比较复杂这里就不讲了。我们不去乘的话OpenGL就变成了上面说的这样。

好了现在可以定义我们的顶点的坐标了:

{% img /安卓特效相机三/10.png %}

我们当然可以用六个点去定义两个三角形:

```
float[] VERTICES = {
        // 左下角三角形
        -1.0f, 1.0f,
        -1.0f, -1.0f,
        1.0f, -1.0f,

        // 右上角三角形
        1.0f, -1.0f,
        1.0f, 1.0f,
        -1.0f, 1.0f
};
```

但是这样的话有两个交点被重复定义了,占用内存比较多,更多情况下我们会用四个点,然后再加一个序号数组去标识三角形的顶点:

```
private static final float[] VERTICES = {
        -1.0f, 1.0f,
        -1.0f, -1.0f,
        1.0f, -1.0f,
        1.0f, 1.0f
};

private static final short[] ORDERS = {
        0, 1, 2, // 左下角三角形

        2, 3, 0  // 右上角三角形
};
```

设置顶点坐标的代码如下:

```
mVertices = CommonUtils.toFloatBuffer(VERTICES);
GLES20.glVertexAttribPointer(mPositionId, 2, GLES20.GL_FLOAT, false, 0, mVertices);
GLES20.glEnableVertexAttribArray(mPositionId);

...

public static FloatBuffer toFloatBuffer(float[] data) {
    ByteBuffer buffer = ByteBuffer.allocateDirect(data.length * 4);
    buffer.order(ByteOrder.nativeOrder());
    FloatBuffer floatBuffer = buffer.asFloatBuffer();
    floatBuffer.put(data);
    floatBuffer.position(0);
    return floatBuffer;
}
```

纹理坐标同理:

```
private static private float[] TEXTURE_COORDS = {
        0.0f, 1.0f,
        0.0f, 0.0f,
        1.0f, 0.0f,
        1.0f, 1.0f
};

...

mCoords = CommonUtils.toFloatBuffer(TEXTURE_COORDS);
GLES20.glVertexAttribPointer(mCoordId, 2, GLES20.GL_FLOAT, false, 0, mCoords);
GLES20.glEnableVertexAttribArray(mCoordId);
```

## 填充颜色特效矩阵

片元着色器中的uColorMatrix的设置类似,只不过由于它是uniform类型的变量,我们用GLES20.glUniformMXXXX去设置:

```
private static float[] COLOR_MATRIX = {
    -1.0f, 0.0f, 0.0f, 1.0f,
    0.0f, -1.0f, 0.0f, 1.0f,
    0.0f, 0.0f, -1.0f, 1.0f,
    0.0f, 0.0f, 0.0f, 1.0f
};
    
mColorMatrixId = GLES20.glGetUniformLocation(mProgram, "uColorMatrix");

GLES20.glUniformMatrix4fv(mColorMatrixId, 1, true, COLOR_MATRIX, 0);
```

glUniformMatrix4fv方法的第三个参数比较值得注意,这里我们填了true，代表需要转置,这是由于OpenGL的矩阵是列优先的:

{% img /安卓特效相机三/11.png %}

因为我们的COLOR\_MATRIX是一个一维数组,其实实际上是这样的:

```
private float[] COLOR_MATRIX = {-1.0f, 0.0f, 0.0f, 1.0f, 0.0f, -1.0f, 0.0f, 1.0f, 0.0f, 0.0f, -1.0f, 1.0f, 0.0f, 0.0f, 0.0f, 1.0f};
```

它去到GPU之后设置给uColorMatrix得到了这个4\*4的矩阵:

```
-1.0f,  0.0f,  0.0f, 0.0f,
 0.0f, -1.0f,  0.0f, 0.0f,
 0.0f,  0.0f, -1.0f, 0.0f,
 1.0f,  1.0f,  1.0f, 1.0f
```

所以我们需要给他做转置操作得到：

```
-1.0f, 0.0f, 0.0f, 1.0f,
0.0f, -1.0f, 0.0f, 1.0f,
0.0f, 0.0f, -1.0f, 1.0f,
0.0f, 0.0f, 0.0f, 1.0f
```


## 纹理变换矩阵

在顶点着色器里面我们讲到了matTransform这个变换矩阵用于变换纹理坐标,它是从SurfaceTexture里面拿到的:

```
private float[] mTransformMatrix = new float[16];
...
mPreviewTexutre.getTransformMatrix(mTransformMatrix);
...
mTransformMatrixId = GLES20.glGetUniformLocation(mProgram, "matTransform");
...
GLES20.glUniformMatrix4fv(mTransformMatrixId, 1, false, matrix, 0);
```

SurfaceTexture从哪里来的我们等下再说,我们的摄像头就是往这里绘制画面。可以用getTransformMatrix方法得到变换矩阵:

```
/**
    * Retrieve the 4x4 texture coordinate transform matrix associated with the texture image set by
    * the most recent call to updateTexImage.
    *
    * This transform matrix maps 2D homogeneous texture coordinates of the form (s, t, 0, 1) with s
    * and t in the inclusive range [0, 1] to the texture coordinate that should be used to sample
    * that location from the texture.  Sampling the texture outside of the range of this transform
    * is undefined.
    *
    * The matrix is stored in column-major order so that it may be passed directly to OpenGL ES via
    * the glLoadMatrixf or glUniformMatrix4fv functions.
    *
    * @param mtx the array into which the 4x4 matrix will be stored.  The array must have exactly
    *     16 elements.
    */
   public void getTransformMatrix(float[] mtx) {
       ...
   }
```

它返回4\*4的纹理坐标变换矩阵:

> Retrieve the 4x4 texture coordinate transform matrix associated with the texture image

然后它是列优先的可以直接使用不用转置:

> The matrix is stored in column-major order so that it may be passed directly to OpenGL ES via the glLoadMatrixf or glUniformMatrix4fv functions.

所以第三个参数我们设置成false:

```
GLES20.glUniformMatrix4fv(mTransformMatrixId, 1, false, matrix, 0);
```

## 创建纹理绘制摄像头画面

我们一直说要将摄像头的画面画到图片上,那图片是怎么来的呢?并不是用安卓上常见的Bitmap去画,而是用GLES20.glGenTextures创建一张OpenGL的纹理:

```
public int getTexture() {
    if (mGLTextureId == -1) {
        int[] textures = new int[1];
        GLES20.glGenTextures(1, textures, 0);
        mGLTextureId = textures[0];
    }

    return mGLTextureId;
}
```

但是创建出来就只是一个id,要怎么给摄像机去用呢?不知道大家还就不记得第一篇[博客](http://blog.islinjw.cn/2019/08/27/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%B8%80-Camera2%E7%9A%84%E4%BD%BF%E7%94%A8/)里面讲到如何设置摄像机画面的接收Surface:


```
public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
    ...
    mPreviewSurface = new Surface(surface);
    ...
}

...

CaptureRequest.Builder builder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
builder.addTarget(mPreviewSurface);
session.setRepeatingRequest(builder.build(), null, null);
```

所以我们也要将这个纹理转换成Surface放到CaptureRequest的Target里面传给CameraDevice:

```

mCameraTexture = new SurfaceTexture(mGLRender.getTexture());

...

CaptureRequest.Builder builder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
builder.addTarget(mCameraTexture);
session.setRepeatingRequest(builder.build(), mCaptureCallback, mHandler);
```

这里我们传入了个mCaptureCallback,摄像机画面绘制到纹理上之后会调用回调,我们需要在回调里面将画面上传到GPU,前面说的纹理转换矩阵也是在这个时候才去获取的:

```
@Override
public void onCaptureCompleted() {
  mCameraTexture.updateTexImage();
  mCameraTexture.getTransformMatrix(mTransformMatrix);
  ...
}
```

这里有说明OpenGL ES里面只能用GL\_TEXTURE\_EXTERNAL\_OES这种纹理去接收:

```
/**
 * Update the texture image to the most recent frame from the image stream.  This may only be
 * called while the OpenGL ES context that owns the texture is current on the calling thread.
 * It will implicitly bind its texture to the GL_TEXTURE_EXTERNAL_OES texture target.
 */
public void updateTexImage() {
    nativeUpdateTexImage();
}
```

所以我们拿到片元着色器里的texPreview之后需要将它绑定到GLES11Ext.GL\_SAMPLER\_EXTERNAL\_OES:

```
mTexPreviewId = GLES20.glGetUniformLocation(mProgram, "texPreview");

...

GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
GLES20.glBindTexture(GLES11Ext.GL_SAMPLER_EXTERNAL_OES, mGLTextureId);
GLES20.glUniform1i(mTexPreviewId, 0);
```

# 绘制与双缓冲

最后的最后我们要执行绘制操作,将整个画面绘画出来:

```
GLES20.glClear(GLES20.GL_DEPTH_BUFFER_BIT | GLES20.GL_COLOR_BUFFER_BIT);
GLES20.glDrawElements(GLES20.GL_TRIANGLES, ORDERS.length, GLES20.GL_UNSIGNED_SHORT, mOrder);
EGL14.eglSwapBuffers(mEGLDisplay, mEGLSurface);
```

这个GLES20.glClear用于将上一帧的画面清除,要不然如果有透明通道的话两帧的画面就会重叠。
而GLES20.glDrawElements代表用mOrder这个顶点顺序去绘制图形,GLES20.GL\_TRIANGLES代表要绘制的是三角形。


最后的mGLCore.swapBuffers代表交互缓冲区,这是由于OpenGL使用了双缓冲的技术。


什么是双缓冲呢?就是有两个缓冲区域:前台缓冲和后台缓冲。前台缓冲即我们看到的屏幕,后台缓冲则在内存当中。

我们会先在后台缓冲绘制图像,绘制完成之后调用EGL14.eglSwapBuffers交换两个缓冲区,原先绘制的缓冲就变成了前台缓冲,显示在屏幕上:

{% img /安卓特效相机三/12.png %}

为什么需要双缓冲呢?这是为了解决绘制的时候屏幕闪烁的问题。我们都知道一般手机屏幕的刷新率是60Hz，而且有些高端的手机甚至比这个更高。

也就是说屏幕一秒钟至少从前台缓冲中获取60次画面显示出来,如果只有一个缓冲的话,假设我们的绘制比较复杂耗时比较多,那可能屏幕会拿到画到一半的图片,就会造成闪烁。而两个缓冲的话就画到一半的图像都在后台缓冲并不会显示,只有完全画好才会交换变成前台缓冲去显示,就解决了这个闪烁的问题。

# 完整代码

完整代码见[github](https://github.com/bluesky466/CameraDemo/tree/feature_shader)(注意是feature\_shader分支,master分支是第一篇文章的demo)