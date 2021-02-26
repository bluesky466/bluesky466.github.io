title: Glide源码探究(二) - 内存缓存
date: 2021-02-08 00:06:28
tags:
    - 技术相关
    - Android
---

系列文章:

- [Glide源码探究(一) - 生命周期绑定与Request创建](https://blog.islinjw.cn/2021/02/05/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%80-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%BB%91%E5%AE%9A%E4%B8%8ERequest%E5%88%9B%E5%BB%BA/)
- [Glide源码探究(二) - 内存缓存](https://blog.islinjw.cn/2021/02/08/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%BA%8C-%E5%86%85%E5%AD%98%E7%BC%93%E5%AD%98/)
- [Glide源码探究(三) - 网络资源加载](https://blog.islinjw.cn/2021/02/10/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%89-%E7%BD%91%E7%BB%9C%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD/)


让我们接着[上一篇笔记](https://blog.islinjw.cn/2021/02/05/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%80-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%BB%91%E5%AE%9A%E4%B8%8ERequest%E5%88%9B%E5%BB%BA/)继续讲Engine的load方法，这里面就是Glide的资源加载流程。

```java
public <R> LoadStatus load(...) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations, resourceClass, transcodeClass, options);

    EngineResource<?> memoryResource;
    synchronized (this) {
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(...);
      }
    }
    
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE, false);
    return null;
}
```

这个方法的流程其实也挺清晰的:

1. 创建缓存的key，这个key由一系列的参数组成，其中最重要的参数model在我们的例子中就是传进去的url。
2. 使用这个key从内存缓存中查询资源
3. 如果内存缓存中查不到资源就开启线程去加载资源
4. 如果内存缓存中可以查到资源就调用cb.onResourceReady回调

流程图如下:

{% img /Glide源码探究二/1.png %}

# 内存缓存

内存缓存的流程也比较清晰从代码上看，如果开启了内存缓存的话会先从ActiveResources中查询，查不到的话再从Cache里面查询:

```java
private EngineResource<?> loadFromMemory( EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      ...
      return active;
    }

    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      ...
      return cached;
    }

    return null;
}
```

这两个东西同样是内存缓存，那有啥区别呢？我们先看ActiveResources:

```java
// Engine
private EngineResource<?> loadFromActiveResources(Key key) {
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      active.acquire();
    }

    return active;
}

// ActiveResources
final class ActiveResources {
  ...
  final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
  ...
  synchronized EngineResource<?> get(Key key) {
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
      return null;
    }

    EngineResource<?> active = activeRef.get();
    if (active == null) {
      cleanupActiveReference(activeRef);
    }
    return active;
  }
  ...
  static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
    ...
  }
  ...
}
```

loadFromActiveResources实际上是从弱引用缓存里面查询资源。既然是缓存当然就要讲讲它的添加和删除。

## 弱引用缓存的添加

首先是添加，弱引用缓存的添加基本有两个时机。

1. 从Cache里面查询到的时候如果能查到，会将查到的资源放入弱引用缓存:

```java
private EngineResource<?> loadFromCache(Key key) {
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
}
```

2. 子线程加载完资源后会将资源放入弱引用缓存:

```java
public synchronized void onEngineJobComplete(EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
    if (resource != null && resource.isMemoryCacheable()) {
      activeResources.activate(key, resource);
    }
    ...
}
```

## 弱引用缓存的删除

细心的同学可能会看到cached.acquire()这个操作，我们来看看它的代码:

```java
synchronized void acquire() {
    ...
    ++acquired;
}
```

有没有想到些啥?没错，引用计数!

EngineResource是通过引用计数来管理的。有引用计数增加那就有引用计数减少。减少的操作在release方法里面:

```java
void release() {
  boolean release = false;
  synchronized (this) {
    if (acquired <= 0) {
      throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
    }
    if (--acquired == 0) {
      release = true;
    }
  }
  if (release) {
    listener.onResourceReleased(key, this);
  }
}
```

如果引用计数降到了0就会调用listener的onResourceReleased回调回去，在onResourceReleased里面Engine会将资源从弱引用缓存删除然后移到cache里:

```java
// Engine
public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    activeResources.deactivate(cacheKey);
    if (resource.isMemoryCacheable()) {
      cache.put(cacheKey, resource);
    } else {
      resourceRecycler.recycle(resource, /*forceNextFrame=*/ false);
    }
}

// ActiveResources
synchronized void deactivate(Key key) {
    ResourceWeakReference removed = activeEngineResources.remove(key);
    if (removed != null) {
      removed.reset();
    }
}
```

EngineResource.release又是什么时候被调用的呢?其实调用的地方有好几处，但是最重要的两处是

1. 我们手动调Glide的clear清理资源的时候:

```kotlin
// 手动清理资源
Glide.with(context)
    .clear(img)
```

2. 绑定的生命LifecycleListener.onDestroy的时候:

```java
// RequestManager
public synchronized void onDestroy() {
  ...
  for (Target<?> target : targetTracker.getAll()) {
    clear(target);
  }
  ...
}

public void clear(@Nullable final Target<?> target) {
  ...
  untrackOrDelegate(target);
}

private void untrackOrDelegate(@NonNull Target<?> target) {
  ...
  Request request = target.getRequest();
  if (!isOwnedByUs && !glide.removeFromManagers(target) && request != null) {
    target.setRequest(null);
    request.clear();
  }
}

// SingleRequest
public void clear() {
  ...
  engine.release(toRelease);
  ...
}

// Engine
public void release(Resource<?> resource) {
  if (resource instanceof EngineResource) {
    ((EngineResource<?>) resource).release();
  }
  ...
}
```

简单来讲就是加载资源的时候会把资源放入弱引用缓存，但资源不需要的时候会从弱引用缓存里面拿出移到另一个内存缓存里面。所以这些资源都是正在使用的，这个弱引用缓存Glide把它叫做ActiveResources也是比较准确的。

**这个缓存使用弱引用的意义在于: 资源是保存在request里面的，而根据我们[上篇笔记](https://www.jianshu.com/p/85da220d8442)的知识，request是以setTag的方式保存在view里面的。所以当view被回收之后，resource也就没有别的强引用可以连接到gc root，可以被java垃圾回收机制回收**

# LRUCache

Engine.load会先从ActiveResources中查询，查不到的话再从Cache里面查询，这个Cache其实是一个LruResourceCache:

```java
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {
    ...
}
```

从这个lru cache里面加载资源意味着把资源从lru cache里面移出，放到弱引用缓存中:

```java
private EngineResource<?> loadFromCache(Key key) {
  EngineResource<?> cached = getEngineResourceFromCache(key);
  if (cached != null) {
    cached.acquire();
    activeResources.activate(key, cached);
  }
  return cached;
}

private EngineResource<?> getEngineResourceFromCache(Key key) {
  Resource<?> cached = cache.remove(key);

  final EngineResource<?> result;
  if (cached == null) {
    result = null;
  } else if (cached instanceof EngineResource) {
    result = (EngineResource<?>) cached;
  } else {
    result = new EngineResource<>(cached, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ true, key, /*listener=*/ this);
  }
  return result;
}
```

而正如上节我们讲的资源的引用计数被清零的时候就会从弱引用缓存中删除，加入lru cache中:

```java
// Engine
public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    activeResources.deactivate(cacheKey);
    if (resource.isMemoryCacheable()) {
      cache.put(cacheKey, resource);
    } else {
      resourceRecycler.recycle(resource, /*forceNextFrame=*/ false);
    }
}

// ActiveResources
synchronized void deactivate(Key key) {
    ResourceWeakReference removed = activeEngineResources.remove(key);
    if (removed != null) {
      removed.reset();
    }
}
```

# 内存缓存整体流程

至此整个内存缓存的架构就大体完整了，当资源被使用的时候会被放到弱引用缓存，当资源不再被使用的时候就会被放入LRU Cache(注意这里放的是强引用，因为是从view里面getTage拿到Resource强引用进行release的):

{% img /Glide源码探究二/2.png %}

# 补充: 内存缓存的查询顺序

先查弱引用缓存再查lru cache这个顺序并不是一开始就这样的，我刚看glide源码的时候看的是比较旧的版本，那个时候是先查lru cahe，查不到再查弱引用缓存。

这个顺序在2017年11月[这个commit](https://github.com/bumptech/glide/commit/02096625b38f5c5fd6c820752a2fc4f0ae2b07ea)之后修改的:

{% img /Glide源码探究二/3.png %}

这个修改是为了修复资源被重复加载的bug，但实际上我看这部分修改的时候，感觉交换查询顺序应该和解这个bug没有直接关系，它更像是作者在重构之后觉得先查lru cache再查弱引用缓存的顺序怪怪的(我那个时候也有这种感觉)，顺手改了下:

{% img /Glide源码探究二/4.png %}

这里它将原本写在Engine的弱引用Map封装成了ActiveResources。

那为啥顺序不是一开始就是先查弱引用缓存呢?原因可能是[一开始的代码](https://github.com/bumptech/glide/blob/fe7154fc88d47c779aec395af7020a69d61f6392/library/src/com/bumptech/glide/load/engine/Engine.java)内存缓存就没有弱引用缓存:

```
public <T, Z> LoadStatus load(String id, int width, int height, ResourceDecoder<InputStream, Z> cacheDecoder,
        ResourceFetcher<T> fetcher, ResourceDecoder<T, Z> decoder,  Transformation<Z> transformation,
        ResourceEncoder<Z> encoder, Priority priority, ResourceCallback cb) {

    Key key = keyFactory.buildKey(id, width, height, cacheDecoder, decoder, transformation, encoder);

    Resource cached = cache.get(key);
    if (cached != null) {
        cached.acquire(1);
        cb.onResourceReady(cached);
        return null;
    }

    ResourceRunner current = runners.get(key);
    if (current != null) {
        EngineJob job = current.getJob();
        job.addCallback(cb);
        return new LoadStatus(cb, job);
    }

    ResourceRunner<Z> runner = factory.build(key, width, height, cacheDecoder, fetcher, decoder, transformation,
            encoder, priority, this);
    runner.getJob().addCallback(cb);
    runners.put(key, runner);
    runner.queue();
    return new LoadStatus(cb, runner.getJob());
}
```
可能是作者在后面优化添加这个弱引用缓存的时候就顺手放到了原有逻辑的后面。

其实仔细想想内存缓存的架构，我觉得这个顺序其实并不重要，谁先谁后都不会有什么问题，无非是说流程是从lru cache拿出来放到弱引用缓存的，查询的时候先查弱引用缓存会比较符合一般人的思路。

我们回想下两个缓存存放的资源，简化到Activity的场景。弱引用缓存放的都是当前activity正在使用的图片，lru cache是activity退出之后回收的图片。如果先查弱引用缓存，意味着当上下不停滑动recyclerview的时候效率高一丢丢。如果先查lru cahe，意味着反复进出同一个activity的时候效率高一丢丢。很难说哪个顺序性能比较高。而且这一丢丢性能在现代设备中的确真的是毫无影响，所以让人好理解是最重要的，先查弱引用缓存没毛病。