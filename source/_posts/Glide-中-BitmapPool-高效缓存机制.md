---
title: Glide 中 BitmapPool 的极致性能优化
date: 2018-05-22
tags:
- Android
- Glide
---
# 对象池
在某些时候，我们需要频繁使用一些临时对象，如果每次使用的时候都申请新的资源，很有可能会引发频繁的 gc 而影响应用的流畅性。这个时候如果对象有明确的生命周期，那么就可以通过定义一个对象池来高效的完成复用对象。  
``` java
private final Map<String, Bitmap> cache = new HashMap<>()

private void setImage(ImageView iv, String name){
    Bitmap b = cache.get(name);
    if(b == null){
        b = loadBitmap(name);
        cache.put(name, b);
    }
    iv.setImageBitmap(b);
}
```

# 多条件 Key
例如上述代码就简单的通过 HashMap 缓存了 Bitmap 资源，只有在缓存不存在时才会执行加载这个耗时操作。但是上面的缓存条件十分简单，是通过图片的名字决定的，这很大程度上满足不了实际的需求。所以我们就需要定义一个 Key 对象来包含各种缓存的条件，例如我们除了图片名字作为条件，还有图片的宽度，高度也决定了是否是同一个资源，那么代码将变成如下：  
``` java
private final Map<Key, Bitmap> cache = new HashMap<>()

private void setImage(ImageView iv, String name, int width, int height){
    Key key = new Key(name, width, height);
    Bitmap b = cache.get(key);
    if(b == null){
        b = loadBitmap(name, width, height);
        cache.put(key, b);
    }
    iv.setImageBitmap(b);
}

public class Key {
  
  private final String name;
  private final int width;
  private final int heifht;

  public Key(String name, int width, int heifht) {
    this.name = name;
    this.width = width;
    this.heifht = heifht;
  }

  public String getName() {
    return name;
  }

  public int getWidth() {
    return width;
  }

  public int getHeifht() {
    return heifht;
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || getClass() != o.getClass()) {
      return false;
    }

    Key key = (Key) o;

    if (width != key.width) {
      return false;
    }
    if (heifht != key.heifht) {
      return false;
    }
    return name != null ? name.equals(key.name) : key.name == null;
  }

  @Override
  public int hashCode() {
    int result = name != null ? name.hashCode() : 0;
    result = 31 * result + width;
    result = 31 * result + heifht;
    return result;
  }
}
```

# Key 复用
这样虽然可以支持多条件的缓存键值了，但是每次查找缓存前都需要创建一个新的 Key 对象，虽然这个 Key 对象很轻量，但是终归觉得不优雅。而刚好想起来在写 Glide 的自定义 BitmapTransformation 时，会提供一个 BitmapPool 来获取 Bitmap 以避免 Bitmap 的频繁申请。而 BitmapPool 中 get 方法的签名是这样的：
``` java
  @NonNull
  Bitmap get(int width, int height, Bitmap.Config config);
```
Bitmap 需要同时满足三个条件（高度、宽度、颜色编码）都相同时才能算是同一个 Bitmap，那么内部是如何进行查找的呢？需要知道的是，BitmapPool 只是一个接口，内部的默认实现是 LruBitmapPool，get 实现如下：
``` java
  @Override
  @NonNull
  public Bitmap get(int width, int height, Bitmap.Config config) {
    Bitmap result = getDirtyOrNull(width, height, config);
    if (result != null) {
      // Bitmaps in the pool contain random data that in some cases must be cleared for an image
      // to be rendered correctly. we shouldn't force all consumers to independently erase the
      // contents individually, so we do so here. See issue #131.
      result.eraseColor(Color.TRANSPARENT);
    } else {
      result = createBitmap(width, height, config);
    }

    return result;
  }

    @Nullable
  private synchronized Bitmap getDirtyOrNull(
      int width, int height, @Nullable Bitmap.Config config) {
    assertNotHardwareConfig(config);
    // Config will be null for non public config types, which can lead to transformations naively
    // passing in null as the requested config here. See issue #194.
    final Bitmap result = strategy.get(width, height, config != null ? config : DEFAULT_CONFIG);
    if (result == null) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Missing bitmap=" + strategy.logBitmap(width, height, config));
      }
      misses++;
    } else {
      hits++;
      currentSize -= strategy.getSize(result);
      tracker.remove(result);
      normalize(result);
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Get bitmap=" + strategy.logBitmap(width, height, config));
    }
    dump();

    return result;
  }
```
其中的重点是  
``` java
final Bitmap result = strategy.get(width, height, config != null ? config : DEFAULT_CONFIG);
```
strategy 是 LruPoolStrategy 接口类型，查看其中一个实现 AttributeStrategy 的 get 方法的实现：  
``` java
  @Override
  public Bitmap get(int width, int height, Bitmap.Config config) {
    final Key key = keyPool.get(width, height, config);

    return groupedMap.get(key);
  }
```
这里同样也需要一个专门的类型用来描述键，但是键居然是也是从一个对象池中获取的，KeyPool#get 方法实现如下：
``` java
    Key get(int width, int height, Bitmap.Config config) {
      Key result = get();
      result.init(width, height, config);
      return result;
    }
```
可以看到 Key 是一个可变对象，每次先获取一个 Key 对象（可能是池中的，也可能是新创建的），然后把变量初始化。但是大家知道，HashMap 中的 Key 不应该是可变对象，因为如果 Key的 hashCode 发生变化将会导致查找失效，那么这里是如何做到 Key 是可变对象的同时保证能正确的作为 HashMap 中的键使用呢，奥妙就在 GroupedLinkedMap#get 函数中：
``` java
  @Nullable
  public V get(K key) {
    LinkedEntry<K, V> entry = keyToEntry.get(key);
    if (entry == null) {
      entry = new LinkedEntry<>(key);
      keyToEntry.put(key, entry);
    } else {
      key.offer();
    }

    makeHead(entry);

    return entry.removeLast();
  }
```
在查找时，如果没有发现命中的值，那么就会创建新的值，并将其连同 Key 保存在 HashMap 中，不会对 Key 进行复用。而如果发现了命中的值，也就是说 HashMap 中已经有一个和当前 Key 相同的 Key 对象了，那么 Key 就可以通过 offer 方法回收到了 KeyPool 中，以待下一次查找时复用。

# 结语
透过这个小案例，可以看到 Glide 在性能优化方面可谓是达到了极致，不光对开销较大的 Bitmap 进行了复用，就连为了复用 Bitmap 时重复申请的 Key 对象都进行了复用，尽可能的减少了对象的创建开销，保证了应用的流畅性。