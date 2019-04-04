Fragment学习笔记

#### 1.Fragment关系图



![](/home/caocong/caocong/源码分析/Fragment框架.jpeg)



**1、**对Fragment的add,remove,replace等操作，由FragmentManagerImpl和BaceStackRecord一起完成，BaceStackRecord可看成是一个事务，把各种操作抽象成一个类Op，存储到一个ArrayList中，commit后集中处理，从ArrayList中依次取出Op交给FragmentManagerImpl去处理，op处理完成后，FragmentManagerImpl将相应的Fragment调整到对应的状态，核心方法moveToState();



***BaceStackRecord.java***

```
static final class Op {
    int cmd;
    Fragment fragment;
    int enterAnim;
    int exitAnim;
    int popEnterAnim;
    int popExitAnim;

}
```

```
void executeOps() {
    final int numOps = mOps.size();
    for (int opNum = 0; opNum < numOps; opNum++) {
        final Op op = mOps.get(opNum);
        final Fragment f = op.fragment;
        if (f != null) {
            f.setNextTransition(mTransition, mTransitionStyle);
        }
        switch (op.cmd) {
            case OP_ADD:
                f.setNextAnim(op.enterAnim);
                mManager.addFragment(f, false);
                break;
            case OP_REMOVE:
                f.setNextAnim(op.exitAnim);
                mManager.removeFragment(f);
                break;
            case OP_HIDE:
                f.setNextAnim(op.exitAnim);
                mManager.hideFragment(f);
                break;
            case OP_SHOW:
                f.setNextAnim(op.enterAnim);
                mManager.showFragment(f);
                break;
            case OP_DETACH:
                f.setNextAnim(op.exitAnim);
                mManager.detachFragment(f);
                break;
            case OP_ATTACH:
                f.setNextAnim(op.enterAnim);
                mManager.attachFragment(f);
                break;
            case OP_SET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(f);
                break;
            case OP_UNSET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(null);
                break;
            default:
                throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
        }
        if (!mReorderingAllowed && op.cmd != OP_ADD && f != null) {
            mManager.moveFragmentToExpectedState(f);
        }
    }
    if (!mReorderingAllowed) {
        // Added fragments are added at the end to comply with prior behavior.
        mManager.moveToState(mManager.mCurState, true);
    }
}


```

***FragmentManagerImpl.java***

```
public void showFragment(Fragment fragment) { 
	...
	
}
public void removeFragment(Fragment fragment) {
	...
}
public void hideFragment(Fragment fragment) {
	...
}
public void attachFragmentFragment(Fragment fragment) {
	...
}
```

......

```
void moveToState(Fragment f, int newState, int transit, int transitionStyle,boolean keepActive){
	...
}
```





**2、**怎么样实现Activity和Fragment的生命周期保持一致，FragmentAcitvity的各种dispatchxxx()和FragmentManager的moveToState()的互动,实时改变Fragment的状态

***FragmentActivity.java***

```
protected void onCreate(@Nullable Bundle savedInstanceState) {
    ......
    mFragments.dispatchCreate();
}
```

```
protected void onDestroy() {
    
    mFragments.dispatchDestroy();

}
```

```
protected void onPause() {

  mFragments.dispatchPause();
}
```

```

protected void onStart() {

  mFragments.dispatchStart();
}
.......

```

**3、**使用Fragment有两种方式：

a.动态调用

b.layout中添加



layout布局添加的方式，为什么Fragment能直接读取并显示出来， 这和  LayoutInflater相关，LayoutInfalter会把layout的xml文件解析出来，然后先交给Factory来创建，没有Factory则反射来创建View（包括自定义控件和sdk已有控件），最后add到decorView并显示出来，FragmentActivity继承了LayoutInfalter的Factory2,并交给Fragment来创建Fragment

***LayoutInflater.java***



```
public interface Factory2 extends Factory {
     public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}

```



```
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
  		
		//先交给Factory来创建
       try {
        View view;
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        //没有Factory，则通过反射创建

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    
}


```

***FragmentActivity.java***

```

public View onCreateView(String name, Context context, AttributeSet attrs) {
        final View v = dispatchFragmentsOnCreateView(null, name, context, attrs);
        if (v == null) {
            return super.onCreateView(name, context, attrs);
        }
        return v;
 }
 
  
 
```

***FragmentManagerImpl.java***



```
public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    if (!"fragment".equals(name)) {
        return null;
    } else {
        String fname = attrs.getAttributeValue((String)null, "class");
        TypedArray a = context.obtainStyledAttributes(attrs, FragmentManagerImpl.FragmentTag.Fragment);
        if (fname == null) {
            fname = a.getString(0);
        }

        int id = a.getResourceId(1, -1);
        String tag = a.getString(2);
        a.recycle();
        if (!Fragment.isSupportFragmentClass(this.mHost.getContext(), fname)) {
            return null;
        } else {
            int containerId = parent != null ? parent.getId() : 0;
            if (containerId == -1 && id == -1 && tag == null) {
                throw new IllegalArgumentException(attrs.getPositionDescription() + ": Must specify unique android:id, android:tag, or have a parent with an id for " + fname);
            } else {
                Fragment fragment = id != -1 ? this.findFragmentById(id) : null;
                if (fragment == null && tag != null) {
                    fragment = this.findFragmentByTag(tag);
                }

                if (fragment == null && containerId != -1) {
                    fragment = this.findFragmentById(containerId);
                }

                if (DEBUG) {
                    Log.v("FragmentManager", "onCreateView: id=0x" + Integer.toHexString(id) + " fname=" + fname + " existing=" + fragment);
                }

                if (fragment == null) {
                    fragment = this.mContainer.instantiate(context, fname, (Bundle)null);
                    fragment.mFromLayout = true;
                    fragment.mFragmentId = id != 0 ? id : containerId;
                    fragment.mContainerId = containerId;
                    fragment.mTag = tag;
                    fragment.mInLayout = true;
                    fragment.mFragmentManager = this;
                    fragment.mHost = this.mHost;
                    fragment.onInflate(this.mHost.getContext(), attrs, fragment.mSavedFragmentState);
                    this.addFragment(fragment, true);
                } else {
                    if (fragment.mInLayout) {
                        throw new IllegalArgumentException(attrs.getPositionDescription() + ": Duplicate id 0x" + Integer.toHexString(id) + ", tag " + tag + ", or parent id 0x" + Integer.toHexString(containerId) + " with another fragment for " + fname);
                    }

                    fragment.mInLayout = true;
                    fragment.mHost = this.mHost;
                    if (!fragment.mRetaining) {
                        fragment.onInflate(this.mHost.getContext(), attrs, fragment.mSavedFragmentState);
                    }
                }

                if (this.mCurState < 1 && fragment.mFromLayout) {
                    this.moveToState(fragment, 1, 0, 0, false);
                } else {
                    this.moveToState(fragment);
                }

                if (fragment.mView == null) {
                    throw new IllegalStateException("Fragment " + fname + " did not create a view.");
                } else {
                    if (id != 0) {
                        fragment.mView.setId(id);
                    }

                    if (fragment.mView.getTag() == null) {
                        fragment.mView.setTag(tag);
                    }

                    return fragment.mView;
                }
            }
        }
    }
}
```



**4、**动态换肤的应用

参见 https://blog.csdn.net/u013085697/article/details/53898879







