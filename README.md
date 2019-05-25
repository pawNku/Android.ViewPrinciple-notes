# Android.ViewPrinciple-notes
Android之View的进而深入原理的一些理解和整理。结合了入门的‘’第一行代码‘’和进阶的‘’Android开发艺术探索‘’两本书和一个实例Demo~   

## 开发艺术探索之View的工作原理   
> 主要内容：
> * View的工作原理
> * 自定义View的实现方式 
> * 自定义View的底层工作原理，比如View的测量流程、布局流程、绘制流程  
> * View常见的回调方法，比如构造方法、onAttach.onVisibilityChanged/onDetach等

### 1. 初识ViewRoot和DecorView
> 在ActivityThread中Activity被创建时 会将DecorView顶级View-->加入Window 同时创建ViewRootImpl 而ViewRoot就是具体实现了     
* 简单的说就是： ViewRoot就是连接Window和DecorView的纽带，View的三大流程（ mearsure、layout、draw） 均是通过ViewRoot来完成 
```
viewRoot = new ViewRootImpl(view.getContext(),display);   //viewRoot是ViewRootImpl具体实现
viewRoot.setView(view,wparams, panelParentView);
```

* View的绘制流程是从ViewRoot也就是ViewRootImpl对象的performTraversals(翻译过来就是执行遍历)开始的   
![图片](https://jakkypan.gitbooks.io/android-develop-art-discovery/content/QQ20151224-1@2x.png)   
* measure用来测量View的宽高   
* layout来确定View在父容器中的位置    
* draw负责将View绘制在屏幕上    
> performTraversals会依次调用 performMeasure 、 performLayout 和performDraw 三个方法，这三个方法分别完成顶级顶级顶级View的measure、layout和draw这三大流程。其中 performMeasure 中会调用 measure 方法，在 measure 方法中又会调用 onMeasure 方法，在 onMeasure 方法中则会对所有子元素子元素子元素进行measure过程，这样就完成了一次measure过程；子元素会重复父容器的measure过程，如此反复完成了整个View数的遍历。另外两个过程同理-->所以有点像树结构的~   

* Measure完成后, 可以通过getMeasuredWidth 、getMeasureHeight 方法来获取View测量后的宽/高。特殊情况下，测量的宽高不等于不等于不等于最终的宽高，详见后面   

* Layout过程决定了View的四个顶点的坐标和实际实际实际View的宽高，完成后可通过 getTop 、 getBotton 、 getLeft 和 getRight 拿到View的四个定点坐标

* View层的事件都是先经过DecorView，然后才传递到子View  

### 2. 理解MeasureSpec    
> MeasureSpec决定了一个View的尺寸规格。但是父容器会影响View的MeasureSpec的创建过程。系统将View的 LayoutParams 根据父容器所施加的规则转换成对应的MeasureSpec，然后根据这个MeasureSpec来测量出View的宽高   

### 3. View的工作流程   

**3.1 measure过程** 
* 直接继承View的自定义控件需要重写 onMeasure 方法并设置 wrap_content （ 即specMode是 AT_MOST 模式） 时的自身大小，否则在布局中使用 wrap_content 相当于使用 match_parent 。对于非 wrap_content 的情形，我们沿用系统的测量值即可   
```
 @Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
      // 在 MeasureSpec.AT_MOST 模式下，给定一个默认值mWidth,mHeight。默认宽高灵活指定
      //参考TextView、ImageView的处理方式
      //其他情况下沿用系统测量规则即可
    if (widthSpecMode == MeasureSpec.AT_MOST
            && heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWith, mHeight);
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWith, heightSpecSize);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSpecSize, mHeight);
    }
}
```
**3.2 layout过程** 
> layout的作用是ViewGroup用来确定子View的位置，当ViewGroup的位置被确定后，它会在onLayout中遍历所有的子View并调用其layout方法，在 layout 方法中， onLayout 方法又会被调用

* View的 layout 方法确定本身的位置，源码流程如下：
> setFrame 确定View的四个顶点位置，即确定了View在父容器中的位置    
  调用 onLayout 方法，确定所有子View的位置，和onMeasure一样，onLayout的具体实现和布局有关，因此View和ViewGroup均没有真正实现 onLayout 方法  
 
* 以LinearLayout的 onLayout 方法为例：父onLayout-->setChildFrame来确定孩子-->子layout   
> 遍历所有子View并调用 setChildFrame 方法来为子元素指定对应的位置   
  setChildFrame 方法实际上调用了子View的 layout 方法，形成了递归   
  
**3.3 draw过程** 

* View的绘制过程遵循如下几步：
> 绘制背景 drawBackground(canvas)    
  绘制自己 onDraw   
  绘制children dispatchDraw 遍历所有子View的 draw 方法   
  绘制装饰 onDrawScrollBars     
  
### 4. 自定义View   

**4.1 自定义View的分类** 

* 继承View 重写onDraw方法--不规则
> 通过 onDraw 方法来实现一些不规则的效果，这种效果不方便通过布局的组合方式来达到。这种方式需要自己支持 wrap_content ，并且padding也要去进行处理

* 继承ViewGroup派生特殊的layout--不规则
> 实现自定义的布局方式，需要合适地处理ViewGroup的测量、布局这两个过程，并同时处理子View的测量和布局过程

* 继承特定的View子类（ 如TextView、Button）
> 扩展某种已有的控件的功能，比较简单，不需要自己去管理 wrap_content 和padding

* 继承特定的ViewGroup子类（ 如LinearLayout）
> 比较常见，实现几种view组合一起的效果。与方法二的差别是方法二更接近底层实现

**4.3 自定义View实例** 
