title: 浅谈SurfaceView与GLSurfaceView
date: 2018-09-21 00:10:51
tags:
	- 技术相关
  - Android
---

#　什么是Surface

让我们看看Surface的官方介绍:

> Handle onto a raw buffer that is being managed by the screen compositor.

Surface是一个raw buffer的句柄,我们可以通过它在raw buffer上进行绘制．

对应到代码其实就是可以通过Surface获得一个Canvas:

```
Canvas canvas = mSurface.lockCanvas(null);
//使用Canvas进行绘制
mSurface.unlockCanvasAndPost(canvas);
```

# SurfaceView

Surface可能大家比较陌生,但是SurfaceView和GLSurfaceView相信大家或多或少都会听说过．

SurfaceView其实就是对Surface进行了一次封装,它内部帮我们管理了一个Surface．我们使用SurfaceView其实最终都是获取到这个Surface去绘制．

这里开门见山,直接抛出一个简单的SurfaceView的用法,下面的Demo用SurfaceView画了一个１００*１００的红色矩形

```
public class MySurfaceView extends SurfaceView implements SurfaceHolder.Callback {
    private DrawThread mDrawThread;

    public MySurfaceView(Context context) {
        this(context, null);
    }

    public MySurfaceView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public MySurfaceView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        getHolder().addCallback(this);
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        mDrawThread = new DrawThread(holder.getSurface());
        mDrawThread.start();
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {

    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        mDrawThread.stopDraw();
        try {
            mDrawThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static class DrawThread extends Thread {
        private Surface mSurface;
        private boolean mRunning = true;
        private Paint mPaint = new Paint();

        DrawThread(Surface surface) {
            mSurface = surface;
            mPaint.setColor(Color.RED);
        }

        void stopDraw() {
            mRunning = false;
        }

        @Override
        public void run() {
            super.run();

            while (mRunning) {
                Canvas canvas = mSurface.lockCanvas(null);
                canvas.drawColor(Color.WHITE);
                canvas.drawRect(0, 0, 100, 100, mPaint);
                mSurface.unlockCanvasAndPost(canvas);

                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

运行效果如下:

{% img /浅谈SurfaceView与GLSurfaceView/1.jpg %}

这个Demo有几个关键代码,第一个是在构造函数里面使用getHolder()获取到SurfaceHolder,然后使用addCallback注册了个监听．这样就能监听SurfaceView内部Surface的生命周期．

接着我们在surfaceCreated回调里面开启了一个DrawThread线程．它的主要工作就是在一个while循环里面不停的绘制．

通过代码我们可以看到这个绘制的过程:

1. 通过SurfaceHolder．getSurface可以获取到Surface
2. 通过Surface.lockCanvas可以获取到Surface的Canvas
3. 使用Canvas去绘制图像
4. 使用Surface.unlockCanvasAndPost可以释放Canvas

相信这个Demo代码不用再多说,大家都可以很快理解．从中我们可以看到,SurfaceView最大的特点就是可以在子线程中绘制图像．

在子线程中绘制图像有什么好处呢？

我们都知道一般情况下View都是在主线程中绘制的,而且需要通过measure、layout、draw三个步骤．当布局越复杂,绘制的效率就越低,而且主线程中的一些耗时操作也会进一步降低效率．

如果使用SurfaceView的话我们就能越过measure、layout操作,而且不会被主线程的运算减低绘制性能．这样的特性十分适合于一些频繁更新且对刷新率有一定要求的程序,如相机的预览、画笔书写等．

# GLSurfaceView

而GLSurfaceView继承自SurfaceView,其实是对SurfaceView再做了一次封装,方便我们在安卓中使用OpenGL.

我们都知道OpenGL是一个跨平台的图形库.它提供了一些全平台统一的图形接口.但是各个平台其实都有一些很难统一的差异,所以为了跨平台的兼容性,OpenGL不负责窗口管理及上下文管理.这部分由各个平台自己实现．EGL就是安卓平台上的实现它是 OpenGL ES 和底层 Native 平台视窗系统之间的接口．

所以在安卓上使用OpenGL,都需要先用EGL进行一些初始化操作,结束的时候再用EGL做一些清理工作．

GLSurfaceView已经帮我们用SurfaceHolder做了EGL的初始化和清理操作,所以我们不需要再去关心EGL．

和我们上面写的SurfaceView的Demo一样,GLSurface的绘制也是在子线程中进行的,它为我们开启了一个GLThread，对一些处理事件进行了处理.我们只需要实现Renderer接口进行绘制即可,GLSurfaceView就会在GLThread中调用我们的Renderer进行绘制:

```
public class GLSurfaceView extends SurfaceView implements SurfaceHolder.Callback2 {
  ...
  private Renderer mRenderer
  ...
  static class GLThread extends Thread {
    ...
    @Override
    public void run() {
      ...
      guardedRun();
      ...
    }
    ...
    private void guardedRun() throws InterruptedException {
      ...
      while(true){
        ...
        view.mRenderer.onSurfaceCreated(gl, mEglHelper.mEglConfig);
        ...
        view.mRenderer.onSurfaceChanged(gl, w, h);
        ...
        view.mRenderer.onDrawFrame(gl);
        ...
      }
      ...
    }
    ...
  }
  ...
}
```

一个简单的Demo如下:

```
public class MyGLSurfaceView extends GLSurfaceView {
    public MyGLSurfaceView(Context context) {
        this(context, null);
    }

    public MyGLSurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);

        setRenderer(new MyRender());
    }

    private static class MyRender implements Renderer {
        private FloatBuffer mVB;

        MyRender() {
            float coords[] = {
                    -0.5f, 0.5f, 0.0f,
                    -0.5f, -0.5f, 0.0f,
                    0.5f, -0.5f, 0.0f,
                    0.5f, -0.5f, 0.0f,
                    0.5f, 0.5f, 0.0f,
                    -0.5f, 0.5f, 0.0f
            };

            ByteBuffer vbb = ByteBuffer.allocateDirect(coords.length * 4);
            vbb.order(ByteOrder.nativeOrder());
            mVB = vbb.asFloatBuffer();
            mVB.put(coords);
            mVB.position(0);
        }

        @Override
        public void onSurfaceCreated(GL10 gl, EGLConfig config) {
            gl.glEnableClientState(GL10.GL_VERTEX_ARRAY);
        }

        @Override
        public void onSurfaceChanged(GL10 gl, int width, int height) {
            gl.glViewport(0, 0, width, height);
        }

        @Override
        public void onDrawFrame(GL10 gl) {
            gl.glClear(GL10.GL_COLOR_BUFFER_BIT | GL10.GL_DEPTH_BUFFER_BIT);
            gl.glClearColor(1.0f, 1.0f, 1.0f, 1.0f);
            gl.glColor4f(1.0f, 0.0f, 0.0f, 1.0f);
            gl.glVertexPointer(3, GL10.GL_FLOAT, 0, mVB);
            gl.glDrawArrays(GL10.GL_TRIANGLES, 0, 6);
        }
    }
}
```

可以看到,我们实现了Renderer去画一个红色矩形,然后使用setRenderer设置给GLSurfaceView就可以了,运行效果如下:

{% img /浅谈SurfaceView与GLSurfaceView/2.jpg %}

