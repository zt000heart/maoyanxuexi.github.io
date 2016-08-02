---
layout: post  
title: 关于RecyclerView的滚动问题
categories: [blog ]  
tags: [Tech, ]  
description: 「」   
---
注：注。

RecyclerView在我们的开发过程中已经使用的不能再多了，开发过程中十分实用，并且使用简单，但我们使用RecyclerView的定位滚动，就会发现问题。

RecyclerView本身提供的滚动方法有两个

	scrollToPosition(int)
	
原本以为这个方法的作用是设置选择位置显示在屏幕顶端，但用过的才知道这是什么东西，当设置position为1，2等，你会发现RecyclerView根本就不动，当设置大一点时，才会发生滚动，但也不会达到我们预想的位置，当再向上设置滚动时，RecyclerView变的正常了，如果不仔细观察你就看不懂了，Recycler怎么滚动完全是看心情啊。。。。。

你仔细观察或者查阅一下相关说明，你才会明白为什么会产生这样的结果，这个方法的作用是显示指定项，就是把你想置顶的项显示出来，但是在屏幕的什么位置是不管的，只要那一项现在看得到了，那它就罢工了！ 而且这个方法是瞬间跳到位置，不带有动画。

另一个方法是

	scrollBy(int x,int y)
	
这个方法使用时是传入滚动的距离，单位是像素，我们使用这个方法要自己控制滚动的距离，在动态的布局中且各项样式高度可能都不一样的情况下，自己计算高度是很有难度的。

还有两个与上面两方法对应的带有动画的移动方法分别是

	smoothScrollToPosition(int)
	smoothScrollBy(int x,int y)

这两个方法的最终结果与上两个相同，只是过程多了滚动动画。

上面说了这么多废话，结论就是这些方法都不能很好解决问题，但是，当ScrollTo和ScrollBy结合使用的时候，我们的问题就变的好解决很多了！
我们解决问题的方法就是先用scrollToPosition，将要置顶的项先移动显示出来，然后计算这一项离顶部的距离，用scrollBy完成剩余的移动，将置顶项移动到顶部。

下面我们来看具体的代码实现：
我们先来分析可能遇到的几种情况哈


第一需要置顶项在屏幕显示的上方，这个最简单直接调用scrollToPosition就会将这一项置顶。

第二需要置顶项在屏幕中显示，但需要置顶，这时直接调用scrollBy，将其置顶。

第三种置顶项在屏幕的下方，这时需要先调用scrollToPosition将置顶项移动到屏幕内，然后就和第二种情况一样了。


    private void moveToPosition(int n) {

        int firstItem = mLinearLayoutManager.findFirstVisibleItemPosition();
        int lastItem = mLinearLayoutManager.findLastVisibleItemPosition();
        if (n <= firstItem ){
            mRecyclerView.scrollToPosition(n);
        }else if ( n <= lastItem ){
            int top = mRecyclerView.getChildAt(n - firstItem).getTop();
            mRecyclerView.scrollBy(0, top);
        }else{
            mRecyclerView.scrollToPosition(n);
            move = true;
        }

    }
    
对于第三种情况我们需要监听滚动，但我们又不想在其他的情况触发，所以我们增加一个标志位move，第三种情况下才有二次移动。
    
    
    class RecyclerViewListener extends RecyclerView.OnScrollListener{

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            if (move){
                move = false;
                int n = mIndex - mLinearLayoutManager.findFirstVisibleItemPosition();
                if ( 0 <= n && n < mRecyclerView.getChildCount()){
                    int top = mRecyclerView.getChildAt(n).getTop();
                    mRecyclerView.scrollBy(0, top);
                }
            }
        }
    }
    
对于smoothScroll实现类似但需要监听onScrollStateChanged

    private void smoothMoveToPosition(int n) {

        int firstItem = mLinearLayoutManager.findFirstVisibleItemPosition();
        int lastItem = mLinearLayoutManager.findLastVisibleItemPosition();
        if (n <= firstItem ){
            mRecyclerView.smoothScrollToPosition(n);
        }else if ( n <= lastItem ){
            int top = mRecyclerView.getChildAt(n - firstItem).getTop();
            mRecyclerView.smoothScrollBy(0, top);
        }else{
            mRecyclerView.smoothScrollToPosition(n);
            move = true;
        }

    }
    
    
    @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);
            if (move && newState == RecyclerView.SCROLL_STATE_IDLE ){
                move = false;
                int n = mIndex - mLinearLayoutManager.findFirstVisibleItemPosition();
                if ( 0 <= n && n < mRecyclerView.getChildCount()){
                    int top = mRecyclerView.getChildAt(n).getTop();
                    mRecyclerView.smoothScrollBy(0, top);
                }

            }
        }
        
这样我们需要的效果就实现了。

 
—End—

## **迭代**

---

### **【乐趣在于挑战】**

不上高山，体会不到一览众山小的乐趣；不下大海，享受不到战胜风浪的自豪。

----