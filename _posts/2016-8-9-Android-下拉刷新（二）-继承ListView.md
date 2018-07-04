---
layout: post
title: Android-下拉刷新（二）-继承ListView
categories: [Android]
description: 通过继承ListView实现下拉刷新效果
tags: 下拉刷新
---
## 前言
记得上一篇用了LinearLayout布局的方式实现了一个下拉刷新库，现在想用另一种方式来实现下拉刷新效果，即继承ListView的方式。

## 实现思路
我们知道ListView自身可以设置HeaderView和FooterView，通常我们可以通过设置HeaderView来实现banner广告栏等，同理，我们也可以让它成为ListView下拉刷新的头部，然后让它在正常状态下隐藏，而在下拉情况下显示。

实现要点主要有两个：

1）如何判断ListView滑动到底部或者滑动到顶部：

```java
    private boolean isReachHeader() {
        return getFirstVisiblePosition() == 0;
    }

```

```java
    private boolean isReachFooter() {
        return getLastVisiblePosition() == getCount() - 1;
    }

```

2）如何让ListView的HeaderView显示和隐藏：

上一篇中，我对于headerView的处理是，设置其layout_maginTop参数，当headerView需要隐藏的时候，设置layout_marginTop的值为负数，当headerView需要显示的时候，设置layout_marginTop的值为正数。

在这里，其实方法也类似，只不过换成对另一个值得处理，通过设置paddingTop的值来实现headerView的显示和隐藏；假设headerView的高度为100，当需要隐藏headerView的时候，我们设置其paddingTop为-100，当我们想要完全显示headerView的时候，设置paddingTop为0，如果想要headerView的位置继续上拉，可以继续增大paddingTop的值。

```java
    private void setHeaderViewTopPadding(int topPadding) {
        if (mHeaderLayout != null) {
            mHeaderLayout.setPadding(mHeaderLayout.getPaddingLeft(), topPadding, 
                    mHeaderLayout.getPaddingRight(), mHeaderLayout.getPaddingBottom());
        }
    }

```

github地址：https://github.com/82367825/RefreshWidget2

再看看效果图：

![这里写图片描述](http://img.blog.csdn.net/20160809001927782)



## 关于ListView需要注意的点

### 1）关于position<br/>
既然谈到利用ListView自带的headerView来实现，就不得不谈到position的注意.

当ListView中含有一个headerView的时候，我们调用setSelection(0)，显示的第一行则是headerView，而调用setSelection(1)，显示的第一行才是ListView列表数据的第一项。

同理，当ListView添加了headerView，在onItemClick（）方法中返回的position参数，实际上=数据项position+1
```java
        mRefreshListViewWidget.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Toast.makeText(MainActivity.this, String.valueOf(position), Toast.LENGTH_SHORT)
                        .show();
            }
        });
```

我们看到结果：

![这里写图片描述](http://img.blog.csdn.net/20160808231828771)

而如果我们想要获取比较准确的position值，

```java
        mRefreshListViewWidget.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Toast.makeText(MainActivity.this, String.valueOf(parent.getAdapter().getItem(position)), 
                        Toast.LENGTH_SHORT)
                        .show();
            }
        });
```

这时候，获取到的position就是我们想要的position不包含HeaderView和FooterView的准确位置啦。可是为什么？为什么通过这种方式，我们又能够获取到正确的position值呢？

我们查看ListView的源码：
```java
   /**
     * Sets the data behind this ListView.
     *
     * The adapter passed to this method may be wrapped by a {@link WrapperListAdapter},
     * depending on the ListView features currently in use. For instance, adding
     * headers and/or footers will cause the adapter to be wrapped.
     *
     * @param adapter The ListAdapter which is responsible for maintaining the
     *        data backing this list and for producing a view to represent an
     *        item in that data set.
     *
     * @see #getAdapter() 
     */
    @Override
    public void setAdapter(ListAdapter adapter) {
        if (mAdapter != null && mDataSetObserver != null) {
            mAdapter.unregisterDataSetObserver(mDataSetObserver);
        }

        resetList();
        mRecycler.clear();

        if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
            mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
        } else {
            mAdapter = adapter;
        }

        mOldSelectedPosition = INVALID_POSITION;
        mOldSelectedRowId = INVALID_ROW_ID;

        // AbsListView#setAdapter will update choice mode states.
        super.setAdapter(adapter);

        if (mAdapter != null) {
            mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
            mOldItemCount = mItemCount;
            mItemCount = mAdapter.getCount();
            checkFocus();

            mDataSetObserver = new AdapterDataSetObserver();
            mAdapter.registerDataSetObserver(mDataSetObserver);

            mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

            int position;
            if (mStackFromBottom) {
                position = lookForSelectablePosition(mItemCount - 1, false);
            } else {
                position = lookForSelectablePosition(0, true);
            }
            setSelectedPositionInt(position);
            setNextSelectedPositionInt(position);

            if (mItemCount == 0) {
                // Nothing selected
                checkSelectionChanged();
            }
        } else {
            mAreAllItemsSelectable = true;
            checkFocus();
            // Nothing selected
            checkSelectionChanged();
        }

        requestLayout();
    }
```
可以看到，当我们调用setAdapter()方法添加适配器之后，系统检测到ListView已经添加了HeaderView，又帮我们修改了Adpater，可以看到mAdpater被重新赋值为HeaderViewListAdapter的实例。

我们明明设置的适配器是ListAdapter，为什么自作主张帮我们修改了，这是什么意思，HeaderViewListAdapter又是什么，带着疑问，我继续查看源码。

```java
/**
 * ListAdapter used when a ListView has header views. This ListAdapter
 * wraps another one and also keeps track of the header views and their
 * associated data objects.
 *<p>This is intended as a base class; you will probably not need to
 * use this class directly in your own code.
 */
public class HeaderViewListAdapter implements WrapperListAdapter, Filterable {

    private final ListAdapter mAdapter;

    // These two ArrayList are assumed to NOT be null.
    // They are indeed created when declared in ListView and then shared.
    ArrayList<ListView.FixedViewInfo> mHeaderViewInfos;
    ArrayList<ListView.FixedViewInfo> mFooterViewInfos;
.......    
```
可以看到注释，当ListView添加了HeaderView或者FooterView的时候，会使用这个适配器来调节。它保留了原来的ListAdapter传递的数据列表，引用了headerView列表和footerView列表，同时重写了几个重要的方法，避免因为HeaderView的存在影响了数据。


我们先看上面可以获取正确数据的getItem()方法：
```java
    public Object getItem(int position) {
        // Header (negative positions will throw an IndexOutOfBoundsException)
        int numHeaders = getHeadersCount();
        if (position < numHeaders) {
            return mHeaderViewInfos.get(position).data;
        }

        // Adapter
        final int adjPosition = position - numHeaders;
        int adapterCount = 0;
        if (mAdapter != null) {
            adapterCount = mAdapter.getCount();
            if (adjPosition < adapterCount) {
                return mAdapter.getItem(adjPosition);
            }
        }

        // Footer (off-limits positions will throw an IndexOutOfBoundsException)
        return mFooterViewInfos.get(adjPosition - adapterCount).data;
    }
```
可以看到，这里关键的处理adjPosition = position - numHeaders，减去了headerView的数量，获取了当前数据列表的position位置，这里的adjPosition就是我们想要的正确的position位置。

同理再看，getItemId()

```java
    public long getItemId(int position) {
        int numHeaders = getHeadersCount();
        if (mAdapter != null && position >= numHeaders) {
            int adjPosition = position - numHeaders;
            int adapterCount = mAdapter.getCount();
            if (adjPosition < adapterCount) {
                return mAdapter.getItemId(adjPosition);
            }
        }
        return -1;
    }
```

但是，当我们调用getCount()方法的时候，就没有那么理所应当了，因为这里返回的数量是包含HeaderView和FooterView数量的。
```java
    public int getCount() {
        if (mAdapter != null) {
            return getFootersCount() + getHeadersCount() + mAdapter.getCount();
        } else {
            return getFootersCount() + getHeadersCount();
        }
    }
```

同理，getView()方法也做了减去HeaderView的处理，因此，当我们添加了HeaderView,而在我们自己写的Adapter类中，却不需要担心getView()方法中传递来的position值得准确性问题，因为源码已经帮我们搞定了，它确保了传递进来的position肯定是准确的。

```java
    public View getView(int position, View convertView, ViewGroup parent) {
        // Header (negative positions will throw an IndexOutOfBoundsException)
        int numHeaders = getHeadersCount();
        if (position < numHeaders) {
            return mHeaderViewInfos.get(position).view;
        }

        // Adapter
        final int adjPosition = position - numHeaders;
        int adapterCount = 0;
        if (mAdapter != null) {
            adapterCount = mAdapter.getCount();
            if (adjPosition < adapterCount) {
                return mAdapter.getView(adjPosition, convertView, parent);
            }
        }

        // Footer (off-limits positions will throw an IndexOutOfBoundsException)
        return mFooterViewInfos.get(adjPosition - adapterCount).view;
    }
```

看完这段源码，真的感叹源码的强大......


### 关于setAdapter()和addHeaderView()的顺序问题

一般不会去想，这两个方法调用顺序的问题，但是有时候问题就会这样出现。

当api<=17的时候，我们如果先调用setAdapter()再调用addHeaderView()方法，我们会发现弹出异常：java.lang.IllegalStateException: Cannot add header view to list -- setAdapter has already been called.

正确的顺序应该是先调用addHeaderView()，再设置Adapter，发生异常的原因，我们只要去查看API17的addHeaderView()方法，我们就能够很快发现.....

<br/>



码了很多字，手好酸，慢慢地自己在看别人博客的同时，也会自己去看Android源码，觉得其实也能看懂一些些，一切慢慢来！
如有错漏，欢迎指正～
