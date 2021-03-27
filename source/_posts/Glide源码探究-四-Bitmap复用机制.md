title: Glide源码探究(四) - Bitmap复用机制
date: 2021-03-27 11:39:28
tags:
    - 技术相关
    - Android
---

系列文章:

- [Glide源码探究(一) - 生命周期绑定与Request创建](https://blog.islinjw.cn/2021/02/05/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%80-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E7%BB%91%E5%AE%9A%E4%B8%8ERequest%E5%88%9B%E5%BB%BA/)

- [Glide源码探究(二) - 内存缓存](https://blog.islinjw.cn/2021/02/08/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%BA%8C-%E5%86%85%E5%AD%98%E7%BC%93%E5%AD%98/)

- [Glide源码探究(三) - 网络资源加载](https://blog.islinjw.cn/2021/02/10/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E4%B8%89-%E7%BD%91%E7%BB%9C%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD/)

- [Glide源码探究(四) - Bitmap复用机制](https://blog.islinjw.cn/2021/03/27/Glide%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6-%E5%9B%9B-Bitmap%E5%A4%8D%E7%94%A8%E6%9C%BA%E5%88%B6/)

现在的app界面越做越复杂，图片也越来越多，每次切换或者滑动页面就会有旧图片的释放与新图片的加载。如果我们不做特殊的优化，只是简单的释放和创建bitmap，那么除了内存资源申请的耗时，由于内存的不断申请与释放造成的内存抖动会很容易引发GC耗时。卡上加卡，越来越卡......

其实内存抖动问题已经有非常常规的解决策略了，那就是复用池技术。直接的做法就是我们可以拿旧图片的bitmap给新图片去循环利用。

你说巧不巧，Glide里面就是这么做的......

{% img /Glide源码探究四/1.jpeg %}

Glide的Bitmap复用是通过BitmapPool实现的，它在Glide在初始化的时候创建:

```java
Glide build(@NonNull Context context) {
  ...
  if (bitmapPool == null) {
    int size = memorySizeCalculator.getBitmapPoolSize();
    if (size > 0) {
      bitmapPool = new LruBitmapPool(size);
    } else {
      bitmapPool = new BitmapPoolAdapter();
    }
  }
  ...
}
```

Glide会通过安卓版本、内存大小、屏幕尺寸等参数计算复用池的大小去创建复用池。如果大小是0的话代表不是用复用池，Glide就会用BitmapPoolAdapter去做一个简单的适配。

# BitmapPoolAdapter

简单到基本啥也不做，就只是普通的创建和销毁Bitmap，完全没有复用:

```java
public class BitmapPoolAdapter implements BitmapPool {
  @Override
  public long getMaxSize() {
    return 0;
  }

  @Override
  public void setSizeMultiplier(float sizeMultiplier) {
    // Do nothing.
  }

  @Override
  public void put(Bitmap bitmap) {
    bitmap.recycle();
  }

  @NonNull
  @Override
  public Bitmap get(int width, int height, Bitmap.Config config) {
    return Bitmap.createBitmap(width, height, config);
  }

  @NonNull
  @Override
  public Bitmap getDirty(int width, int height, Bitmap.Config config) {
    return get(width, height, config);
  }

  @Override
  public void clearMemory() {
    // Do nothing.
  }

  @Override
  public void trimMemory(int level) {
    // Do nothing.
  }
}
```

# LruBitmapPool

所以我们的这篇博客的重点就在LruBitmapPool了。我们在[Glide源码探究(二) - 内存缓存](https://blog.islinjw.cn/2021/02/08/Glide源码探究-二-内存缓存/)里面讲过图片资源引用计数被清零的时候就会从弱引用缓存中删除，加入lru cache中。而如果这个时候lru cache满了的话就会对最近最久未使用的图片资源进行回收。简单来讲就是使用BitmapPool.put方法将它丢到复用池:

```java
public class BitmapResource implements Resource<Bitmap>, Initializable {
  ...
  @Override
  public void recycle() {
    bitmapPool.put(bitmap);
  }
  ...
}
```

而我们的LruBitmapPool就会将它放到strategy中:

```java
private static LruPoolStrategy getDefaultStrategy() {
  final LruPoolStrategy strategy;
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    strategy = new SizeConfigStrategy();
  } else {
    strategy = new AttributeStrategy();
  }
  return strategy;
}

@Override
public synchronized void put(Bitmap bitmap) {
  ... // 判断该bitmap是否可被回收
  strategy.put(bitmap);
  ...
}
```

从源码可以看出这个strategy在KITKAT以下的版本使用的是AttributeStrategy，在KITKAT以上的版本使用的是SizeConfigStrategy，它们两者同样都是LRU Cache。

## AttributeStrategy

这样区分的原因是安卓4.4对Bitmap的复用做了优化，在4.4以前只有宽高和Config一致的时候才能复用一个bitmap:

```java
class AttributeStrategy implements LruPoolStrategy {
  private final KeyPool keyPool = new KeyPool();
  private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<>();

  @Override
  public void put(Bitmap bitmap) {
    final Key key = keyPool.get(bitmap.getWidth(), bitmap.getHeight(), bitmap.getConfig());

    groupedMap.put(key, bitmap);
  }

  @Override
  public Bitmap get(int width, int height, Bitmap.Config config) {
    final Key key = keyPool.get(width, height, config);

    return groupedMap.get(key);
  }
  ...
}
```

## KeyPool

这里的Key是通过图片的大小和config去创建的，由于查询的频率比较高，为了防止这个Key的多次创建，这里也用了池化技术:

```java
static class KeyPool extends BaseKeyPool<Key> {

  public Key get(int size, Bitmap.Config config) {
    Key result = get();
    result.init(size, config);
    return result;
  }

  @Override
  protected Key create() {
    return new Key(this);
  }
}

abstract class BaseKeyPool<T extends Poolable> {
  private static final int MAX_SIZE = 20;
  private final Queue<T> keyPool = Util.createQueue(MAX_SIZE);

  T get() {
    T result = keyPool.poll();
    if (result == null) {
      result = create();
    }
    return result;
  }

  public void offer(T key) {
    if (keyPool.size() < MAX_SIZE) {
      keyPool.offer(key);
    }
  }

  abstract T create();
}
```

## SizeConfigStrategy

而由于实际开发的时候两张图片资源尺寸完全一样的情况不多(尤其在不同页面)，会导致复用的命中率比较低。而安卓4.4之后如果config相同只需要旧图片Bitmap的内存大小大于新图片需要的内存大小就能拿来复用了，这样就能提高复用的命中率:

```java
public class SizeConfigStrategy implements LruPoolStrategy {
  ...
  private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<>();
  private final Map<Bitmap.Config, NavigableMap<Integer, Integer>> sortedSizes = new HashMap<>();
  ...
  @Override
  public void put(Bitmap bitmap) {
    int size = Util.getBitmapByteSize(bitmap);
    Key key = keyPool.get(size, bitmap.getConfig());

    groupedMap.put(key, bitmap); // 缓存bitmap

    NavigableMap<Integer, Integer> sizes = getSizesForConfig(bitmap.getConfig());
    Integer current = sizes.get(key.size);
    sizes.put(key.size, current == null ? 1 : current + 1); // size这个大小的bitmap数量加一
  }
  ...
  private NavigableMap<Integer, Integer> getSizesForConfig(Bitmap.Config config) {
    NavigableMap<Integer, Integer> sizes = sortedSizes.get(config);
    if (sizes == null) {
      sizes = new TreeMap<>();
      sortedSizes.put(config, sizes);
    }
    return sizes;
  }
  ...
}
```

可以看到SizeConfigStrategy在回收的时候除了将bitmap放到groupedMap之外，还会用sortedSizes记录每种config的不同尺寸缓存bitmap缓存的数量。

于是在get的时候只需要在缓存的bitmap里面找到能够满足新的图片内存需求的去复用即可:

```java
public Bitmap get(int width, int height, Bitmap.Config config) {
  int size = Util.getBitmapByteSize(width, height, config);
  Key bestKey = findBestKey(size, config); // 从缓存的bitmap中找到内存比新图片需要的内存大的

  Bitmap result = groupedMap.get(bestKey);
  if (result != null) {
    decrementBitmapOfSize(bestKey.size, result); // 减少sortedSizes中可以复用的bitmap
    result.reconfigure(width, height, config); // 修改尺寸实现复用
  }
  return result;
}
```

查找的核心代码在findBestKey:

```java
private Key findBestKey(int size, Bitmap.Config config) {
  Key result = keyPool.get(size, config);
  for (Bitmap.Config possibleConfig : getInConfigs(config)) {
    NavigableMap<Integer, Integer> sizesForPossibleConfig = getSizesForConfig(possibleConfig); // 通过config获取可用的bitmap尺寸
    Integer possibleSize = sizesForPossibleConfig.ceilingKey(size); // 找到大于size的最小可用尺寸
    if (possibleSize != null && possibleSize <= size * MAX_SIZE_MULTIPLE) {
      if (possibleSize != size
          || (possibleConfig == null ? config != null : !possibleConfig.equals(config))) {
        // 如果满足复用条件，就将原本的key回收，通过复用的bitmap尺寸创建复用的key
        keyPool.offer(result);
        result = keyPool.get(possibleSize, possibleConfig);
      }
      break;
    }
  }
  return result;
}
```

这里是先通过config获取到缓存的bitmap尺寸，然后通过NavigableMap.ceilingKey方法查找到大于需要尺寸的最小可用尺寸。如果可以找到就能用这个尺寸去groupedMap里面查找Bitmap复用了。

