---
layout: post  
title: BottomNavigationBar代码分析
categories: [blog ]  
tags: [Tech, ]  
description: 「」   
---
注：注。
git地址：[BottomNavigationBar](https://github.com/Ashok-Varma/BottomNavigation)


这是一个导航栏的实现，亮点在于动画的处理，本文将分析BottomNavigationBar的实现方式。


	public BottomNavigationBar(Context context, AttributeSet attrs, int defStyleAttr) {
	    super(context, attrs, defStyleAttr);
	    parseAttrs(context, attrs);
	    init();
	}
	
构造方法包含两步，parseAttrs获取属性和init创建父布局，有两个属性需要关注mMode有三种取值MODE_FIXED，MODE_SHIFTING和MODE_DEFAULT，mBackgroundStyle有三种取值，BACKGROUND_STYLE_STATIC，BACKGROUND_STYLE_RIPPLE和BACKGROUND_STYLE_DEFAULT这两个属性的作用后面会用到很多，还有就是动画时间，mRippleAnimationDuration是mAnimationDuration的2.5倍。

	private void init() {
	    setLayoutParams(new ViewGroup.LayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT)));
	    LayoutInflater inflater = LayoutInflater.from(getContext());
	    View parentView = inflater.inflate(R.layout.bottom_navigation_bar_container, this, true);
	    mBackgroundOverlay = (FrameLayout) parentView.findViewById(R.id.bottom_navigation_bar_overLay);
	    mContainer = (FrameLayout) parentView.findViewById(R.id.bottom_navigation_bar_container);
	    mTabContainer = (LinearLayout) parentView.findViewById(R.id.bottom_navigation_bar_item_container);
	    ......
	}
	
mBackgroundOverlay是在mTabContainer下的一层背景层，设置了整个View的高度阴影。这是初始化的部分，在使用过程中会创建BottomNavigationItem，然后调用bottomNavigationBar的addItem

	ArrayList<BottomNavigationItem> mBottomNavigationItems = new ArrayList<>();
	public BottomNavigationBar addItem(BottomNavigationItem item) {
	    mBottomNavigationItems.add(item);
	    return this;
	}
	
就是将所有的item听见到List里。在添加完成后会调用initialise()，这里才是一切的开始，

	public void initialise() {
	    if (!mBottomNavigationItems.isEmpty()) {
	        mTabContainer.removeAllViews();
	        if (mMode == MODE_DEFAULT) {
	            if (mBottomNavigationItems.size() <= MIN_SIZE) {
	                mMode = MODE_FIXED;
	            } else {
	                mMode = MODE_SHIFTING;
	            }
	        }
	        if (mBackgroundStyle == BACKGROUND_STYLE_DEFAULT) {
	            if (mMode == MODE_FIXED) {
	                mBackgroundStyle = BACKGROUND_STYLE_STATIC;
	            } else {
	                mBackgroundStyle = BACKGROUND_STYLE_RIPPLE;
	            }
	        }
	        if (mBackgroundStyle == BACKGROUND_STYLE_STATIC) {
	            mBackgroundOverlay.setVisibility(View.GONE);
	            mContainer.setBackgroundColor(mBackgroundColor);
	        }
	        int screenWidth = Utils.getScreenWidth(getContext());
	        if (mMode == MODE_FIXED) {
	            int[] widths = BottomNavigationHelper.getMeasurementsForFixedMode(getContext(), screenWidth, mBottomNavigationItems.size(), mScrollable);
	            int itemWidth = widths[0];
	            for (BottomNavigationItem currentItem : mBottomNavigationItems) {
	                FixedBottomNavigationTab bottomNavigationTab = new FixedBottomNavigationTab(getContext());
	                setUpTab(bottomNavigationTab, currentItem, itemWidth, itemWidth);
	            }
	        } else if (mMode == MODE_SHIFTING) {
	            int[] widths = BottomNavigationHelper.getMeasurementsForShiftingMode(getContext(), screenWidth, mBottomNavigationItems.size(), mScrollable);
	            int itemWidth = widths[0];
	            int itemActiveWidth = widths[1];
	            for (BottomNavigationItem currentItem : mBottomNavigationItems) {
	                ShiftingBottomNavigationTab bottomNavigationTab = new ShiftingBottomNavigationTab(getContext());
	                setUpTab(bottomNavigationTab, currentItem, itemWidth, itemActiveWidth);
	            }
	        }
	        if (mBottomNavigationTabs.size() > mFirstSelectedPosition) {
	            selectTabInternal(mFirstSelectedPosition, true, false);
	        } else if (!mBottomNavigationTabs.isEmpty()) {
	            selectTabInternal(0, true, false);
	        }
	    }
	}
	
 首先可以看到MODE_DEFAULT会根据添加的item数量确定当前的模式，3个及以下时是MODE_FIXED，以上是MODE_SHIFTING，mBackgroundStyle如果是DEFAULT的话，会根据mMode决定，mBackgroundStyle是static，背景颜色不需要改变直接设置。fixed模式每一个item无论是否活跃状态宽度都一样，计算出每一个item的宽度itemWidth，然后生成tabView，FixedBottomNavigationTab，setUpTab会会根据传入的tab设置绑定数据。
 
	 private void setUpTab(BottomNavigationTab bottomNavigationTab, BottomNavigationItem currentItem, int itemWidth, int itemActiveWidth) {
	    bottomNavigationTab.setInactiveWidth(itemWidth);
	    bottomNavigationTab.setActiveWidth(itemActiveWidth);
	    bottomNavigationTab.setPosition(mBottomNavigationItems.indexOf(currentItem));
	    bottomNavigationTab.setOnClickListener(new OnClickListener() {
	        @Override
	        public void onClick(View v) {
	            BottomNavigationTab bottomNavigationTabView = (BottomNavigationTab) v;
	            selectTabInternal(bottomNavigationTabView.getPosition(), false, true);
	        }
	    });
	    mBottomNavigationTabs.add(bottomNavigationTab);
	    BottomNavigationHelper.bindTabWithData(currentItem, bottomNavigationTab, this);
	    bottomNavigationTab.initialise(mBackgroundStyle == BACKGROUND_STYLE_STATIC);
	    mTabContainer.addView(bottomNavigationTab);
	}
	
setUpTab首先设置宽度，fix时传入的itemWidth和itemActiveWidth相同，设置当前tab的所在位置，添加点击事件，selectTabInternal就是切换时的操作，这里是关键稍后需要仔细看。mBottomNavigationTabs是存放所有生成tabView的列表，给每一个刚生成的tabView绑定item数据，bindTabWithData设置了各种属性。

bottomNavigationTab.initialise，这里可以看到StateListDrawable，该类定义了不同状态值下与之对应的图片资源，即我们可以利用该类保存多种状态值，多种图片资源。View的五种状态值，enabled，focused，pressed，selected，window_focused.状态前面的“-”表示相反状态。

	iconView.setSelected(false);
	if (isInActiveIconSet) {
	    StateListDrawable states = new StateListDrawable();
	    states.addState(new int[]{android.R.attr.state_selected},
	            mCompactIcon);
	    states.addState(new int[]{-android.R.attr.state_selected},
	            mCompactInActiveIcon);
	    states.addState(new int[]{},
	            mCompactInActiveIcon);
	    iconView.setImageDrawable(states);
	}
	
这样就很好理解item的icon设置图标了。如果没有设置InActive图标的时候，则判断是否是BACKGROUND_STYLE_STATIC，分别处理。再回到setUpTab，将生成好的tab添加到mTabContainer容器中，setUpTab的流程看完后回到initialise初始化第一个选中的tab。

	private void selectTabInternal(int newPosition, boolean firstTab, boolean callListener) {
	       int oldPosition = mSelectedPosition;
	       if (mSelectedPosition != newPosition) {
	           if (mBackgroundStyle == BACKGROUND_STYLE_STATIC) {
	               if (mSelectedPosition != -1)
	                   mBottomNavigationTabs.get(mSelectedPosition).unSelect(true, mAnimationDuration);
	               mBottomNavigationTabs.get(newPosition).select(true, mAnimationDuration);
	           } else if (mBackgroundStyle == BACKGROUND_STYLE_RIPPLE) {
	               if (mSelectedPosition != -1)
	                   mBottomNavigationTabs.get(mSelectedPosition).unSelect(false, mAnimationDuration);
	               mBottomNavigationTabs.get(newPosition).select(false, mAnimationDuration);
	               final BottomNavigationTab clickedView = mBottomNavigationTabs.get(newPosition);
	               if (firstTab) {
	                   // Running a ripple on the opening app won't be good so on firstTab this won't run.
	                   mContainer.setBackgroundColor(clickedView.getActiveColor());
	                   mBackgroundOverlay.setVisibility(View.GONE);
	               } else {
	                   mBackgroundOverlay.post(new Runnable() {
	                       @Override
	                       public void run() {
	                           BottomNavigationHelper.setBackgroundWithRipple(clickedView, mContainer, mBackgroundOverlay, clickedView.getActiveColor(), mRippleAnimationDuration);
	                       }
	                   });
	               }
	           }
	           mSelectedPosition = newPosition;
	       }
	        ...
	   }
	   
mFirstSelectedPosition是第一个选中的tab位置，也可以调用setFirstSelectedPosition手动设置第一个选中的tab，不设置的情况下，
初次调用传入的newPosition=0，firstTab=true，callListener=false，oldPosition标记了上一个点击的item，设置当前选中的item，
对于FixedBottomNavigationTab它的select和unSelect方法如下。


	@Override
	public void select(boolean setActiveColor, int animationDuration) {
	    labelView.animate().scaleX(1).scaleY(1).setDuration(animationDuration).start();
	    super.select(setActiveColor, animationDuration);
	}
	@Override
	public void unSelect(boolean setActiveColor, int animationDuration) {
	    labelView.animate().scaleX(labelScale).scaleY(labelScale).setDuration(animationDuration).start();
	    super.unSelect(setActiveColor, animationDuration);
	}
	
相比较基类方法多的是底部文本的缩放动画，基类实现了图片上下移动。看下基类的select方法

	public void unSelect(boolean setActiveColor, int animationDuration) {
	    isActive = false;
	    ValueAnimator animator = ValueAnimator.ofInt(containerView.getPaddingTop(), paddingTopInActive);
	    animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	        @Override
	        public void onAnimationUpdate(ValueAnimator valueAnimator) {
	            containerView.setPadding(containerView.getPaddingLeft(),
	                    (Integer) valueAnimator.getAnimatedValue(),
	                    containerView.getPaddingRight(),
	                    containerView.getPaddingBottom());
	        }
	    });
	    animator.setDuration(animationDuration);
	    animator.start();
	    labelView.setTextColor(mInActiveColor);
	    iconView.setSelected(false);
	    if (badgeItem != null) {
	        badgeItem.unSelect();
	    }
	}
	
看完这段代码我们再回过头来看fix–static类型的动画效果，其实就只是点击的时候，上面的图片上移（图片的移动是通过属性动画改编Padding），下面的文字变大。这样点击起来就产生了立体点击的效果。

再来看STYLE_RIPPLE这个模式的效果实现。

ripple相比较static增加的是背景颜色的变换，选中的背景颜色会以水波纹的形式扩展，到整个导航栏。再回到selectTabInternal方法，查看ripple部分，其他部分都比较好理解，除了这行就是实现圆形展开的秘密

	BottomNavigationHelper.setBackgroundWithRipple(clickedView, mContainer, mBackgroundOverlay, clickedView.getActiveColor(), mRippleAnimationDuration);

	public static void setBackgroundWithRipple(View clickedView, final View backgroundView,
	                                           final View bgOverlay, final int newColor, int animationDuration) {
	    int centerX = (int) (clickedView.getX() + (clickedView.getMeasuredWidth() / 2));
	    int centerY = clickedView.getMeasuredHeight() / 2;
	    int finalRadius = backgroundView.getWidth();
	    backgroundView.clearAnimation();
	    bgOverlay.clearAnimation();
	    Animator circularReveal;
	    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
	        circularReveal = ViewAnimationUtils
	                .createCircularReveal(bgOverlay, centerX, centerY, 0, finalRadius);
	    } else {
	        bgOverlay.setAlpha(0);
	        circularReveal = ObjectAnimator.ofFloat(bgOverlay, "alpha", 0, 1);
	    }
	    circularReveal.setDuration(animationDuration);
	    ...
	    circularReveal.start();
	}
	
水波动画的中心点是点击tab的中心，展开的长度最多不超过整个导航栏的宽度。LOLLIPOP以上是水波动画，以下则直接使用改编透明度，水波动画使用安卓系统自带的ViewAnimationUtils.createCircularReveal，创建后开启。

然后看的是SHIFTING类型，不同的是该类型创建的tab是ShiftingBottomNavigationTab，同样需要看select和unselect方法。

	@Override
	public void select(boolean setActiveColor, int animationDuration) {
	    super.select(setActiveColor, animationDuration);
	    ResizeWidthAnimation anim = new ResizeWidthAnimation(this, mActiveWidth);
	    anim.setDuration(animationDuration);
	    this.startAnimation(anim);
	    labelView.animate().scaleY(1).scaleX(1).setDuration(animationDuration).start();
	}
	
select使用了一个自定义的ResizeWidthAnimation是一个自定义的动画，然后让底部的文本逐渐变小到消失，来看一下ResizeWidthAnimation的实现

	public class ResizeWidthAnimation extends Animation {
	    private int mWidth;
	    private int mStartWidth;
	    private View mView;
	    public ResizeWidthAnimation(View view, int width) {
	        mView = view;
	        mWidth = width;
	        mStartWidth = view.getWidth();
	    }
	    @Override
	    protected void applyTransformation(float interpolatedTime, Transformation t) {
	        mView.getLayoutParams().width = mStartWidth + (int) ((mWidth - mStartWidth) * interpolatedTime);
	        mView.requestLayout();
	    }
	    @Override
	    public boolean willChangeBounds() {
	        return true;
	    }
	}
	
ResizeWidthAnimation在applyTransformation中更新布局实现动画。 

 再点击每个tab的时候同样会调用selectTabInternal，整个流程也就分析的差不多了。
 
—End—

## **迭代**

---

### **【乐趣在于挑战】**

不上高山，体会不到一览众山小的乐趣；不下大海，享受不到战胜风浪的自豪。

----