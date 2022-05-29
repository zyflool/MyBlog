---
title: 杂七杂八的知识点和tips
date: 2020-05-03 15:26:24
tags: Android
categories: 学习
---

##### ImageButton波纹点击效果的background

一般ImageButton都会把图片设置为src，然后background就默认成很丑的button默认的灰色背景，但是换成纯色背景就会取代原来的点击的波纹效果，所以要设置background为android自带的点击效果的背景：

```xml
//圆形波纹
android:background=”?android:attr/selectableItemBackgroundBorderless”
//有边界波纹
android:background=”?android:attr/selectableItemBackground”
```

##### NavigationBottomView 图标文字颜色设置

属性设置：`app:itemIconTint="@color/bottom_nav_select_color"`

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:color="#CBDEFA" android:state_checked="true" />
    <item android:color="#001C35" android:state_checked="false"/>
</selector>
```

##### NavigationBottomView+ViewPager+Fragment

```java
BottomNav = view.findViewById(R.id.bnv_home);
        DataVp = view.findViewById(R.id.vp_data);

        DataVp.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {

            }

            @Override
            public void onPageSelected(int position) {
                BottomNav.getMenu().getItem(position).setChecked(true);
            }

            @Override
            public void onPageScrollStateChanged(int state) {

            }
        });
        
        BottomNav.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem item) {
                DataVp.setCurrentItem(item.getOrder());
                return true;
            }
        });

        DataVp.setAdapter(new FragmentPagerAdapter(getActivity().getSupportFragmentManager()) {
            @Override
            public Fragment getItem(int position) {
                switch (position) {
                    case 0:
                        return dataFragment;
                    case 1:
                        return infoFragment;
                    case 2:
                        return aboutFragment;
                }
                return null;
            }

            @Override
            public int getCount() {
                return 3;
            }
        });

        return view;
    }
```

##### TabLayout标签消失

TabLayout.setupWithViewPager()执行路径上有一个方法：

```java
void populateFromPagerAdapter() {
        //注意此处：
        removeAllTabs();

        if (mPagerAdapter != null) {
            final int adapterCount = mPagerAdapter.getCount();
            for (int i = 0; i < adapterCount; i++) {
                addTab(newTab().setText(mPagerAdapter.getPageTitle(i)), false);
            }

            // Make sure we reflect the currently set ViewPager item
            if (mViewPager != null && adapterCount > 0) {
                final int curItem = mViewPager.getCurrentItem();
                if (curItem != getSelectedTabPosition() && curItem < getTabCount()) {
                    selectTab(getTabAt(curItem));
                }
            }
        }
    }
```

把先前加进去的tab都删除了，所以要在绑定ViewPager之后再设置tab的标签。

##### FloatingActionButton 有透明度的背景

使用`app:backgroundTint`属性设置背景颜色，但当颜色具有一定的透明度时，显示的按钮中间会有一块多边形的区域，区域外的透明度是正常的，区域内就是不正常的，点击时这块区域还会缩小。为了减小这样的效果带来的影响，要设置下面的属性：

```xml
app:borderWidth="0dp"
app:elevation="1dp"
```

##### AlertDialog 设置背景

```java
AlertDialog dialog = new AlertDialog.Builder(mContext).create();
final Window window = dialog.getWindow();
window.setBackgroundDrawable(new ColorDrawable(0));
```

##### Fragment 单例

实际经验告诉我们，不要轻易给自定义Fragment设置单例模式，不然会`androidx.fragment.app.Fragment$InstantiationException: Unable to instantiate fragment XXX.XXX.XXXFragment : could not find Fragment constructor`

##### RecyclerView 的数据更新

有时候会因为传入adapter的数据与显示数据不符，出现下面这样的报错：

> java.lang.IndexOutOfBoundsException: Inconsistency detected. Invalid view holder adapter positionViewHolder{42319ed8 position=1 id=-1, oldPos=0, pLpos:0 scrap tmpDetached no parent}

可能是在更新adapter数据List的时候进行notify相关操作导致的两者的不符。

应该尽量避免使用`notifySetChanged()`方法，如果有数据更新，最好先`notifyItemRangeRemoved()`再`notifyItemRangeInserted()`，保证同步。

```java
if ( mRumorsList.size() != 0 ) {
  int preSize = mRumorsList.size();
  mRumorsList.clear();
  notifyItemRangeRemoved(0, preSize);
} 
mRumorsList.addAll(rumorsList);
notifyItemRangeInserted(0, mRumorsList.size());
```

##### 用注解代替枚举

@IntDef、@StringDef、@LongDef

```java
//1.指定注解的保留策略，RetentionPolicy.SOURCE表示只保留源码中，编译时删除。还有CLASS和RUNTIME
    @Retention(RetentionPolicy.SOURCE)

    //2.定义int 值 ，
    //写法1： flag true表示可以用 MODE_STANDARD|MODE_LIST 或者 MODE_LIST & MODE_TABS 这种方式使用，然并卵 ,默认false
    @IntDef(flag = true,value = {MODE_STANDARD, MODE_LIST, MODE_TABS})
    //写法2：
    //@IntDef({MODE_STANDARD, MODE_LIST, MODE_TABS})

    //3.定义注解类型
    public @interface MODE {
        int MODE_STANDARD = 1;      //默认是 public static final
        int MODE_LIST = 2;
        int MODE_TABS = 4;
    }

    //4.使用注解 示例
    @MODE int getMode()         { return MODE_STANDARD; }
    void setMode(@MODE int m)   { mode = m; }
    int mode;
    void testMode(){
        int m1 = MODE_LIST;
        int m2 = getMode();
        setMode(m1);
        m2 = getMode();

        m2 = m1 | MODE_LIST & MODE_TABS ;

    }
```

```kotlin
//1.指定注解的保留策略，AnnotationRetention.SOURCE表示只保留源码中，编译时删除。还有CLASS和RUNTIME
    @Retention(AnnotationRetention.SOURCE)
    //2.定义int 值 ，
    @IntDef(flag = true, value = [MODE_STANDARD, MODE_LIST, MODE_TABS])
    //3.定义注解类型
    annotation class MODE {
        companion object {
            const val MODE_STANDARD   = 1
            const val MODE_LIST       = 2
            const val MODE_TABS       = 4
        }
    }
    //4.使用注解 示例
    fun resetMode(@MODE m: Int) {
        mode = m
    }
    var mode: Int = 0

    fun testMode() {
        val m1 = MODE_LIST

        resetMode(m1)

        var m2 = m1 or (MODE_LIST and MODE_TABS)

    }
```

