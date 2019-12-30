## Scrap : 

**mAttachScarp** :仅在pre-layout阶段不为空。**mChangeScrap**:post-layout后为空。

1.当adapter.notifyItemXXX的时候，改变的item进入mChangedScrap,没有改变的item进入mAttachScrap，然后，从mAttachScrap取出来的item是没有改变过的，不用重新bind，被改变的item对应的position在mAttachScrap中找不到就需要create 和 bind，然后mChangeScrap的viewholder移除进入到RecycledViewPool。

2.当adapter.notifyDataSetChanged的时候，如果有setHasStableId(true)和重写getItemId()的话，和notifyItemXXX的表一样，但是从mAttachScrap取出来的item要从新bind。

3.当adapter.notifyDataSetChanged的时候，如果没有setHasStableId(true)和重写getItemId()的话，所有的viewholder全部进入RecycledViewPool,超过5个的被丢弃。然后重新bind.

## Recycl

**mCachedView** :第二级缓存，缓存的是移除屏幕的item.默认大小容量为2，支持设置改变大小。比如在一些需要频繁上下来回滑动切换的地方就可以把这个设置得大一些。当缓存2个以后，又有新item要缓存的时候，会把第一个item挤出去，第一个item进入到RecycledViewPool.

** ViewCacheExtension ** ：第三级缓存，提供给开发者自定义的缓存策略，默认为null.

**RecycledViewPool** :第四级缓存:按照itemtype缓存，每个itemtype默认最多缓存5个，超过的丢弃。

缓存过程举例：手机屏幕上只显示4个item，手指向上滑，列表把item0移出屏幕，item0进入到mCachedView,
再向上滑，要创建item4的时候到mCachedView中取，但是取出来的item的position是0，和4不一致所以不能使用，继续到第三级第四级找都没有，就走create，然后item1也进入mCacheedView,再向上滑，创建item5通样取出来的item的position不对也走create,然后item2要进入mCachedView了，发现size已经为2了，这时候就把item0挤出去，item0进入到RecycledViewPool，在向上滑，创建item6的时候，mCachedView取出来的item的position通样不对，然后从RecycledViewPool有找到一个item，就拿来重新bind完成复用。

参考：

- ["看完这篇，面试RecyclerView的时候再也不怕了"](https://mp.weixin.qq.com/s/auphzaQF6_wJx6dGFY6niA)
- ["真正带你搞懂 RecyclerView 的缓存机制，再也不怕面试被虐了"](https://mp.weixin.qq.com/s/S31bHWLtUeR4-sjI-ULoUQ)
- ["面试加分项：RecyclerView全面的源码解析"](https://mp.weixin.qq.com/s/FiQEa0M93eSi1i4PzW_6Nw)