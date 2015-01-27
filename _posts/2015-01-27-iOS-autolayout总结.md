---
layout: post
title:  "iOS autoLayout总结"
date:   2015-01-27 20:18:00
categories: iOS autoLayout UIScrollView
---

以前学习iOS的时候没怎么接触autoLayout，自从iPhone6个6+出来之后一直在为以前的app做适配，所以接触到了很多关于autoLayout，一开始很不习惯，但是越用越觉得好用，接触到现在遇到很多问题，在这里总结一下，包括三部分：限制的优先级、autoLayout下得UIScrollView和UITableView。

##优先级
在一开始autolayout的使用过程中，优先级常常是被我所忽略掉的，所以有的时候在一些稍微复杂的布局中往往会出现一些很奇怪的问题和警告，尤其是布局一些大小随内容改变的控件时（UIButton、UILabel、UIImageView），而这些问题和警告都可以通过优先级来解决，下面以UILabel为例子来总结一下：

![图1](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto1.png) 

以上是UILabel的限制，都是采用默认的优先级，当点击长文字按钮的时候label上附有长文本，点击短文字按钮则是短的文本。

首先看一下Content Hugging Priority以下部分，一开始使用autolayout的时候我是没有关注到这一部分的。Content Hugging Priority的的意思是限制内容变大优先级，下面对应横向和纵向，Content Compression Resistance Priority是限制内容缩小优先级，最下面的Intrinsic Size则是设置内容固定大小。

为更好理解上述术语的意思，demo中就关注了横向。运行demo无论点击长文字还是短文字按钮label的大小都是不会改变的。下面通过改变一些优先级来使label大小随文字大小改变，首先将label的Trailing的限制优先级改为700，其他不变，然后运行，发现label可随文字变大但不能变小，这是因为label的右边距离父视图的优先级700小于750，所以Trailing Constrain失效，限制内容变小的限制生效，所以当label内容变多时就限制住label变小，但是Content Hugging Priority的优先级为251小于700，当文字少时无法阻止label变大，现在改变Content Hugging Priority为800。

![图2](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto2.png)

出现警告，期望label宽度0，是因为在storyboard设计阶段自动计算label文字宽度为0，所以label大小也为0；可以通过设置Intrinsic Size 为PlaceHolder去掉警告，这里告诉storyboard设置一个临时占位尺寸，这个占位尺寸仅在storyboard设计阶段有效，不会影响到运行时的尺寸，运行，现在正常了。

![图3](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto3.png)


##UIScrollView

在autoLayout下，UIScrollView的contentSize是由其中的内容大小来决定的，依赖关系和正常的子视图依赖父视图是相反的，所以UIScrollView的子视图的布局约束是不可以通过UIScrollView来确定的，所以一般情况下的约束到了UIScrollView中就会出现很多错误和警告。

要处理这种情况就去要确定UIScrollView中子视图的宽和高，但是这又和autolayout下宽、高的可变性冲突，目前的方法是引进一个锚点视图，子视图的宽和高根据锚点视图确定。

![图4](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto4.png)

anchorViewForWidth是一个宽和父视图相等，高为0的视图，contentView的高固定、宽度和anchorViewForWidth相等，我们也必须设置contentView的top、trailing、leading、button，这不影响contentView的大小，这相当于是UIScrollView可滚动区域的旁白，所以像这样在一般视图中看上去重复设定限制会发生警告，但在这里就不会出现。

在纵向滑动的UIScrollView项目中contentView的宽度依赖可以这么设置，上面的contentView的高度我们是固定的，但如果高度是随运行时确定的我们就不可以设置固定了，在ios8中像UILabel、UIButton、UIImageView这类大小随内容改变的控件我们是不需要设定高度，系统会自动根据内容计算，如果控件中的内容是动态获得的，我们可以设定placeHolder占位尺寸来进行预设定；

![图5](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto5.png)

设置placeHolder之后

![图6](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto6.png)

而对于系统无法计算的控件虽然设置了placeHolder没有了警告，但是运行时控件却不可见，所以只有先设定高度固定再将限制映射为变量，运行时计算修改constant。

![图7](https://raw.githubusercontent.com/RuiKQ/RuiKQ.github.io/master/assets/images/0127/auto7.png)

####UIScrollView中的注意事项
1、有些情况下UIScrollView不能滚动
原因是使用autolayout之后，在ViewDidLoad之后，系统会重新计算控件的一些值会导致UIScrollView的ContentSize变为（0，0），所以需要在viewDidLayoutSubViews方法中重新设置UIScrollView的contentSize，但有时在ios7上不行，ios7需要在viewDidAppear：animated方法设置contentSize。

##UITableView

在UITableView的cell中使用autoLayout，可以根据内容本身来计算cell的高度，在iOS8中只要将tableView.rowHeight设置为UITableViewAutomaticDimension，系统就会根据cell设定好的约束自动计算出高度，在iOS7中需要使用systemLayoutSizeFittingSize:方法来根据约束计算cell的Size。