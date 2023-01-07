---
title: FragmentPagerAdapter 使用指南
date: 2017-09-16
tags:
- Android
---
本文记录一些关于使用 FragmentPagerAdapter 时的方法总结，帮助我们优化 ViewPager 的性能。

# 数据刷新
在默认情况下，调用 PagerAdapter#notifyDataSetChanged() 并不会刷新已经存在的页面排序。需要我们重写 PagerAdapter#getItemPosition() 方法。该方法返回刷新后已经存在 fragment 的新位置。并且已经预定义好了两个常量  POSITION_NONE 和 POSITION_UNCHANGED，分别表示原先存在的 fragment 现在已经不在列表中了和原先存在的 fragment 刷新后位置没有发生改变。示例：
``` java
  @Override
  public int getItemPosition(Object object) {
    Fragment f = (Fragment) object;
    long id = f.getArguments().getLong("id");
    int index = indexOfId(id);
    return index == -1 ? POSITION_NONE : index;
  }
```
这里传入的 object 参数是 PagerAdaper#instantiateItem() 方法的返回值，在 FragmentPagerAdapter 中实际上是我们在 FragmentPagerAdapter#getItem() 中返回的
``` java
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }
```
因为一般情况下，我们会把数据通过 Fragment#setArguments() 传递给 fragment 实例，所以就先从实例对象中获取到 fragment 的相关信息，然后确定 fragment 在新列表中的位置返回即可。这样，在我们调用 PagerAdapter#notifyDataSetChange() 之后，就会根据这个值调整已有的 fragment 的顺序并且创建新的还不存在的 fragment。假如该方法直接返回 POSITION_NONE，那么将会导致所有的 fragment 重建，在 fragment 创建开销比较大时，就会造成性能上的瓶颈。
如上重写了 getItemPosition() 了之后会发现当页面多了之后就出现了 bug。这是因为在 PagerAdapter 中默认使用 position 来唯一标识 fragment，在页面多之后就需要重建和销毁 fragment，而怎么找到需要销毁和重建的 fragment 呢，就是通过 position。
``` java
        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
```
我们还需要重写 FragmentPagerAdapter#getItemId() 方法。示例：
``` java
    @Override
    public long getItemId(int position) {
      return dataList.get(position).getId();
    }
```
建议在 fragment 之间传值直接传递 id，而不要传递整个数据的序列化对象。还有一个需要注意的地方，千万不要持有 fragment 的引用 List。例如这样实现 FragmentPagerAdapter#getItem() 函数。
``` java
    @Override
    public Fragment getItem(int position) {
      return fragmentList.get(position);
    }
```
因为 FragmentManager 会帮助我们管理 fragment，在 activity 被系统销毁重建后，在 PagerAdapter 中的 fragment 将和我们持有的 fragment 不是同一个引用了。正确的获取 fragment 实例的方法是通过 tag。
``` java
    private static String makeFragmentName(int viewId, long id) {
        return "android:switcher:" + viewId + ":" + id;
    }
```
上面的方法是 FragmentPagerAdapter 为每个 fragment 创建对应 tag 的方法，访问权限是 private 的，所以我们只能假设在未来该方法的实现也不会改变，然后我们就拷贝该方法实现到我们需要的地方，然后通过     FragmentManager#findFragmentByTag() 方法获取到实例。

# 懒加载
当 fragment 创建的 view 十分复杂，需要较长的渲染时间时且有多个这样的页面存在于 ViewPager 中时，就需要让 fragment 不在同一时间加载出来，只有需要显示到用户面前时，才开始渲染页面。在使用 FragmentPagerAdapter 时如何实现这样的功能呢？首先，我们需要知道，在什么时候页面会展示到用户的视线中。
``` java
    @Override
    public void setPrimaryItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment)object;
        if (fragment != mCurrentPrimaryItem) {
            if (mCurrentPrimaryItem != null) {
                mCurrentPrimaryItem.setMenuVisibility(false);
                mCurrentPrimaryItem.setUserVisibleHint(false);
            }
            if (fragment != null) {
                fragment.setMenuVisibility(true);
                fragment.setUserVisibleHint(true);
            }
            mCurrentPrimaryItem = fragment;
        }
    } 
```
在 setPrimaryItem 方法中我们会发现，当 fragment 显示在用户面前时，会调用 fragment#setUserVisibleHint 方法，表明这个 fragment 对用户时可见的。开始页面渲染的条件：数据已经准备好 && 第一次进入用户视野。如果对 RxJava 很熟悉的话，要表达这样的逻辑简直就不要太简单。
``` java
public class SampleFragment extends Fragment {

  private ConnectableFlowable<Date> dataSource;

  public SampleFragment() {
    dataSource = getFlowableDate.publish();
  }

  @Override
  public void onViewCreated(View view, @Nullable Bundle savedInstanceState) {
    super.onViewCreated(view, savedInstanceState);
    dataSource.subscribe(/*render*/);
  }

  @Override
  public void setUserVisibleHint(boolean isVisibleToUser) {
    super.setUserVisibleHint(isVisibleToUser);
    if(isVisibleToUser){
      dataSource.connect();
    }
  }
}
```
只需要调用 publish() 把一个 Flowable 变成 ConnectableFlowable，然后在 setUserVisibleHint() 中判断 isVisibleToUser 值为真时就调用 ConnectableFlowable#connect 方法就好了。需要注意的是，setUserVisibleHint 可能会比 fragment 的生命周期都先被调用，所以建议在 fragment 的构造方法中就完成 ConnectableFlowable 的创建工作。