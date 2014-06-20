
##jQuery UI util组件 position解析
---
jquery.ui.position组件是一个独立的util组件不依赖任何其他组件，可以独立存在。
它为我们提供了相对另一个元素定位一个元素的能力
举个栗子： 我们需要一个div相对浏览器居中position组件一个方法搞定: <a href="#" target="_blank">示例</a></br>
对position 有了解的可以直接跳过到<a href="source">这里</a>

    $('.mydiv').position({
        'at': 'center',
        'my': 'center',
        'of': window
    });
**at,my** 属性为position组件提供目标元素和相对元素的位置信息，有三个可选值**'left center right'**， **of**
属性提供一个相对元素, 可选值有: **Selector or Element or jQuery or <a href="#">Event</a>**, **event**是个特殊的选项，
后面会专门提到。

进价1：PM突然来了一个需求输入框输入数字同时需要在输入框上边悬浮一个框 so easy。

    $('.mydiv').position({
        'at': 'left top',
        'my': 'left bottom',
        'of': 'input'
    });
    
进价2：PM又来了XXX我们有个需求，想让鼠标在某个区域移动的时候跟随鼠标显示一个层。PM真烦 不过还好position知心朋友
so easy 

    $('.div1').mousemove(function (event) {
        $('.div2').position({
            'my': 'left bottom',
            'of': event
        });
    });
    
是不是so easy。**at,my,of**三个参数已经可以满足工作中得大多数需求了。position还有另外三个参数**collision,using,within**,
**collision**属性提供一个碰撞检测选项，可选值有四个：
>   "flip"：翻转元素到目标的相对一边，再次运行 collision 检测一遍查看元素是否适合。无论哪一边允许更多的元素可见，则使用那一边。<br>
    "fit"：把元素从窗口的边缘移开<br>
    "flipfit"：首先应用 flip 逻辑，把元素放置在允许更多元素可见的那一边。然后应用 fit 逻辑，确保尽可能多的元素可见。
    "none"：不应用任何 collision 检测。
  
##<a href="javascript:void();" name="source">源码分析</a>  
API介绍结束，分析源码前有些基础知识有必要先了解下</br>
height、clientHeight、scrollHeight、offsetHeight区别
>   height :其实Height高度跟其他的高度有点不一样,在javascript中它是属于对象的style对象属性中的一个成员,它的值是一个字符类型的,而另外三个高度的值是int类型的,它们是对象的属性.因此这样document.body.height就会提示undenifine,而必须写成document.body.style.height<br>
    clientHeight:可见区域的宽度,不包括boder的宽度,如果区域内带有滚动条,还应该减去横向滚动条不可用的高度。滚动条的宽度不同浏览器不同的版本宽度值都不一样这个比较坑爹，不过还是有方法可以计算的，在position的源码里你会看到的</br>
    scrollHeight:这个属性就比较麻烦了,因为它们在火狐跟IE下简直差太多了.在火狐下还很好理解,它其实就是滚动条可滚动的部分还要加上boder的高度还要加上横向滚动条不可用的高度,与clientHeight比起来,多个border的高度跟横向滚动条不可用的高度.</br>
    offsetHeight:可见区域的宽度,如果有设置boder的话还应该加上boder的值
    
基础知识了解后，就可以正式分析看他的源码了。 先看下源码里提供的一些工具函数
>   at,my 属性允许我们这样设置at: 'left+10' | at: 'left+10%', 我们需要去计算整数偏移和百分比偏移。
    第一个工具函数getOffsets就是为我们提供了一个这样的功能
    
    function getOffsets( offsets, width, height ) {
        //parseInt('+10%', 10) >> 10， rpercent = /%$/ 检测偏移量是否百分比，注意这里的偏移是相对于at,my自身的。
        return [
    		parseInt( offsets[ 0 ], 10 ) * ( rpercent.test( offsets[ 0 ] ) ? width / 100 : 1 ),
    		parseInt( offsets[ 1 ], 10 ) * ( rpercent.test( offsets[ 1 ] ) ? height / 100 : 1 )
    	];
    }   
