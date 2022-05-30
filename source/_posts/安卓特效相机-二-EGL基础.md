title: 安卓特效相机(二) EGL基础
date: 2019-09-13 11:51:20
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

上一篇[文章](http://blog.islinjw.cn/2019/08/27/%E5%AE%89%E5%8D%93%E7%89%B9%E6%95%88%E7%9B%B8%E6%9C%BA-%E4%B8%80-Camera2%E7%9A%84%E4%BD%BF%E7%94%A8/)已经和大家讲解了下Camera2的使用,能够将相机捕捉到的画面显示到TextureView上,接下来就是要对这个画面进行特效处理了。这里我们选用OpenGL ES去实现特效。

但是在讲如何使用OpenGL ES去实现特效之前我们要先讲下EGL

# EGL

OpenGL作为一个被广泛使用的跨平台图形库,相信大家或多或少都有听说。安卓上使用的是它针对手机、PDA和游戏主机等嵌入式设备而设计的子集OpenGL ES

为了实现跨平台,OpenGL本体只包含了图形运算方面的接口,也可以大致理解为只包含操作GPU的API。而作为和平台强相关的本地窗口系统交互部分由于每个平台都不一样,而且没有办法抽离出统一的接口,所以不能包含进OpenGL本体里面。例如Android上我们可以指定OpenGL绘制在哪个Surface上,但是IOS上并没有这个东西,所以不能在OpenGL里面开一个接口接收Surface这个参数。

那OpenGL运行到实际的系统上的时候又是怎么和具体的系统做交互的呢？计算机领域有一句话:"计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决"。EGL就是安卓上的这个中间层,OpenGL通过它来和安卓系统的交互。我们可以用这个EGL来设置安卓上特有的一些配置项,比如之前说的Surface。顺带提一嘴在IOS里面这个中间层叫做EAGL。


## EGL的使用

EGL的使用其实套路比较死板,没有比较死记整个流程和api,只要大概知道每个接口的作用就好。可以写一个自己的工具类,配置上默认值,在需要的时候再去修改。

在代码里面安卓经过这么多年的迭代已经给我们提供了几个版本的EGL,比如EGL10、EGL11、EGL14。它们用法基本一样,我这里就用EGL14举例了。

EGL使用的整个流程如下:

{% img /安卓特效相机二/1.png %}

### 初始化EGL

首先我们要创建EGLDisplay并且初始化EGL环境:

```
mEGLDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);

if (mEGLDisplay == EGL14.EGL_NO_DISPLAY){
    throw new RuntimeException("can't get eglGetDisplay");
}

if (!EGL14.eglInitialize(mEGLDisplay, null, 0, null, 0)) {
    throw new RuntimeException("eglInitialize failed");
}
```

EGLDisplay指的是物理的显示设备比如我们的手机屏幕,我们可以通过传入屏幕设备的id去获取到设备句柄,绝大多数情况下我们传入EGL14.EGL\_DEFAULT\_DISPLAY获取默认的屏幕就好,而且一般情况下我们的手机也只有一个屏幕。

如果拿不到设备就会返回EGL\_NO\_DISPLAY。

拿到EGLDisplay之后就可以调用EGL14.eglInitialize去初始化EGL环境。第一个参数是EGLDisplay,然后可以通过后面的参数获取系统中EGL的实现版本号,做一些兼容处理,这里我们没有使用什么高级特性,不需要获取,传null和0就好。如果需要获取的话可以传入一个数组,并且指定major版本和minor想要存放到数组的哪个位置去获取。

### 选择EGLConfig

在使用OpenGL的时候EGLDisplay支持的配置有很多种,例如颜色可能支持ARGB888、RGB888、RGB444、RGB565等,我们可以通过eglGetConfigs拿到EGLDisplay支持的所有配置,然后选择我们需要的。

```
public static native boolean eglGetConfigs(
    EGLDisplay dpy,
    EGLConfig[] configs,
    int configsOffset,
    int config_size,
    int[] num_config,
    int num_configOffset
);
```

不过如果直接去遍历所有的配置找我们需要的那个,代码写起来比较麻烦。所以EGL提供了一个eglChooseConfig方法,我们输入关心的属性,其他的属性让EGL自己匹配就好。可能会匹配出多个EGLConfig,这个时候随便选一个都可以:

```
private EGLConfig chooseEglConfig(EGLDisplay display) {
    int[] attribList = {
            EGL14.EGL_BUFFER_SIZE, 32,
            EGL14.EGL_ALPHA_SIZE, 8,
            EGL14.EGL_RED_SIZE, 8,
            EGL14.EGL_GREEN_SIZE, 8,
            EGL14.EGL_BLUE_SIZE, 8,
            EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,
            EGL14.EGL_SURFACE_TYPE, EGL14.EGL_WINDOW_BIT,
            EGL14.EGL_NONE
    };
    EGLConfig[] configs = new EGLConfig[1];
    int[] numConfigs = new int[1];
    if (!EGL14.eglChooseConfig(
            display,
            attribList,
            0,
            configs,
            0,
            configs.length,
            numConfigs,
            0)) {
        throw new RuntimeException("eglChooseConfig failed");
    }
    return configs[0];
}

...

mEGLConfig = chooseEglConfig(mEGLDisplay);
```

### 创建EGLContext

EGLContext是OpenGL的线程相关上下文环境,我们在OpenGL中创建的数据如图片、顶点、着色器等最后获取到的只是一个id,它的具体内容其实依赖这个EGLContext。所以接下来我们需要将它创建出来。

```
mEGLContext = createEglContext(mEGLDisplay, mEGLConfig);
if (mEGLContext == EGL14.EGL_NO_CONTEXT) {
    throw new RuntimeException("eglCreateContext failed");
}

private EGLContext createEglContext(EGLDisplay display, EGLConfig config) {
    int[] contextList = {
            EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
            EGL14.EGL_NONE
    };
    return EGL14.eglCreateContext(
            display,
            config,
            EGL14.EGL_NO_CONTEXT,
            contextList,
            0);
}
```

这个上下文环境是线程相关的,一般来讲OpenGL的操作都在同一个线程中进行,但是有些复杂的业务场景可能需要多线程,于是可以在eglCreateContext的第三个参数里面传入share_context做到多线程共享。如果不需要多线程共享的话传入EGL14.EGL\_NO\_CONTEXT就好

### 创建并指定EGLSurface

我们在EGLDisplay指定了屏幕,在EGLContext里面提供了上下文环境,最后一步就是要指定一个EGLSurface告诉OpenGL应该往哪里画东西。


很多安卓程序员可能就算写过一些简单的OpenGL程序,也没有直接使用过EGL。因为我们熟悉的GLSurfaceView已经帮我们封装好了,我们只需要在Render.onDrawFrame里面直接使用OpenGL的接口绘图就好了,绘制的图形就好显示在GLSurfaceViews上。

但是由于我们这里使用TextureView替代SurfaceView,并且也没有啥GLTextureView帮我们封装好EGL。需要自己去配置EGL并用TextureView的SurfaceTexture去创建并指定EGLSurface让预览画面绘制到TextureView上:

```
public EGLSurface createEGLSurface(Surface surface) {
        int[] attribList = {
                EGL14.EGL_NONE
        };
        return EGL14.eglCreateWindowSurface(
                mEGLDisplay,
                mEGLConfig,
                surface,
                attribList,
                0);
    }

...

public void makeCurrent(EGLSurface eglSurface) {
    EGL14.eglMakeCurrent(mEGLDisplay, eglSurface, eglSurface, mEGLContext);
}

...

public void initEGL(SurfaceTexture surface) {
    ...
    EGLSurface eglSurface = createEGLSurface(new Surface(surface));
    makeCurrent(eglSurface);
    ...
}
```

我们可以用eglCreateWindowSurface创建EGLSurface然后用eglMakeCurrent指定OpenGL绘制的结果最后输出到这个EGLSurface上。

其实TextureView和SurfaceView都可以用来显示预览画面,它们各有优缺点。SurfaceView在WMS中有对应的WindowState实际上是多开了个窗口浮在应用的上面,因为这个Surface不在View hierachy中，它的显示也不受View的属性控制，所以不能进行平移，缩放等变换。而TextureView不会创建多个窗口,所以可以用view的属性去控制它,但是渲染的性能的话会比SurfaceView稍微低一点。

# 总结

总结一下EGL的三大模块和相关方法如下:

{% img /安卓特效相机二/2.png %}

完整的EGL初始化代码如下,大多数情况不需要修改就可以直接拷贝去用了:

```
private void initEGL(SurfaceTexture surface) {
        mEGLDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);

        if (mEGLDisplay == EGL14.EGL_NO_DISPLAY) {
            throw new RuntimeException("can't get eglGetDisplay");
        }

        if (!EGL14.eglInitialize(mEGLDisplay, null, 0, null, 0)) {
            throw new RuntimeException("eglInitialize failed");
        }

        mEGLConfig = chooseEglConfig(mEGLDisplay);
        mEGLContext = createEglContext(mEGLDisplay, mEGLConfig);
        if (mEGLContext == EGL14.EGL_NO_CONTEXT) {
            throw new RuntimeException("eglCreateContext failed");
        }

        EGLSurface eglSurface = createEGLSurface(new Surface(surface));
        makeCurrent(eglSurface);
    }


    private EGLConfig chooseEglConfig(EGLDisplay display) {
        int[] attribList = {
            EGL14.EGL_BUFFER_SIZE, 32,
            EGL14.EGL_ALPHA_SIZE, 8,
            EGL14.EGL_RED_SIZE, 8,
            EGL14.EGL_GREEN_SIZE, 8,
            EGL14.EGL_BLUE_SIZE, 8,
            EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,
            EGL14.EGL_SURFACE_TYPE, EGL14.EGL_WINDOW_BIT,
            EGL14.EGL_NONE
        };
        EGLConfig[] configs = new EGLConfig[1];
        int[] numConfigs = new int[1];
        if (!EGL14.eglChooseConfig(
            display,
            attribList,
            0,
            configs,
            0,
            configs.length,
            numConfigs,
            0)) {
            throw new RuntimeException("eglChooseConfig failed");
        }
        return configs[0];
    }

    private EGLContext createEglContext(EGLDisplay display, EGLConfig config) {
        int[] contextList = {
            EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
            EGL14.EGL_NONE
        };
        return EGL14.eglCreateContext(
            display,
            config,
            EGL14.EGL_NO_CONTEXT,
            contextList,
            0);
    }

    public EGLSurface createEGLSurface(Surface surface) {
        int[] attribList = {
            EGL14.EGL_NONE
        };
        return EGL14.eglCreateWindowSurface(
            mEGLDisplay,
            mEGLConfig,
            surface,
            attribList,
            0);
    }

    public void makeCurrent(EGLSurface eglSurface) {
        EGL14.eglMakeCurrent(mEGLDisplay, eglSurface, eglSurface, mEGLContext);
    }
}
```
