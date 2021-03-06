---
category: jQuery-UI
---
###jquery-ui@version1.9.1
<!--more-->

jquery.ui.position组件是一个独立的util组件不依赖任何其他组件，可以独立存在。
它为我们提供了相对另一个元素定位一个元素的能力
举个栗子： 我们需要一个div相对浏览器居中position组件一个方法搞定: <a href="#" target="_blank">示例</a><br>
对position 有了解的可以直接跳过到<a href="#source">这里</a>

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
>"flip"：翻转元素到目标的相对一边，再次运行 collision 检测一遍查看元素是否适合。无论哪一边允许更多的元素可见，则使用那一边。<br>
>"fit"：把元素从窗口的边缘移开<br>
>"flipfit"：首先应用 flip 逻辑，把元素放置在允许更多元素可见的那一边。然后应用 fit 逻辑，确保尽可能多的元素可见。<br>
>"none"：不应用任何 collision 检测。

##<a href="javascript:void();" name="source">源码分析</a>

API介绍结束，分析源码前有些基础知识有必要先了解下<br>
height、clientHeight、scrollHeight、offsetHeight区别
>height :其实Height高度跟其他的高度有点不一样,在javascript中它是属于对象的style对象属性中的一个成员,它的值是一个字符类型的,而另外三个高度的值是int类型的,它们是对象的属性.因此这样document.body.height就会提示undenifine,而必须写成document.body.style.height<br>
>clientHeight:可见区域的宽度,不包括boder的宽度,如果区域内带有滚动条,还应该减去横向滚动条不可用的高度。滚动条的宽度不同浏览器不同的版本宽度值都不一样这个比较坑爹，不过还是有方法可以计算的，在position的源码里你会看到的<br>
>scrollHeight:这个属性就比较麻烦了,因为它们在火狐跟IE下简直差太多了.在火狐下还很好理解,它其实就是滚动条可滚动的部分还要加上boder的高度还要加上横向滚动条不可用的高度,与clientHeight比起来,多个border的高度跟横向滚动条不可用的高度.<br>
>在IE中 scrollHeight确是指这个对象它所包含的对象的高度加上boder的高度和marging,如果它里面没有包含对象或者这个对象的高度值未设置,那么它的值将为15<br>
>offsetHeight:可见区域的宽度,如果有设置boder的话还应该加上boder的值
    
基础知识了解后，就可以正式分析看他的源码了。 先看下源码里提供的一些工具函数
>at,my 属性允许我们这样设置at: 'left+10' | at: 'left+10%', 我们需要去计算整数偏移和百分比偏移。
>第一个工具函数getOffsets就是为我们提供了一个这样的功能
    
    function getOffsets( offsets, width, height ) {
        //parseInt('+10%', 10) >> 10， rpercent = /%$/ 检测偏移量是否百分比，注意这里的偏移是相对于at,my自身的。
        return [
    		parseInt( offsets[ 0 ], 10 ) * ( rpercent.test( offsets[ 0 ] ) ? width / 100 : 1 ),
    		parseInt( offsets[ 1 ], 10 ) * ( rpercent.test( offsets[ 1 ] ) ? height / 100 : 1 )
    	];
    }   
$.position 对象提供了三个工具方法
>scrollbarWidth: 用来计算滚动条的宽度，不同浏览器不同的版本滚动条的宽度也是不一样，所以需要一个可以计算滚动宽度的函数。
>它的实现思路很简单， 先把两个嵌套div放到body里，设置外层的overflow：hidden 然后获取此时外层div的offsetWidth，
>然后把外层的div设置为overflow:scroll再获取外层div的offsetWidth，最后两个值想减就是滚动条的宽度了。如果两个值相等的话 则第二次取clientWidth的值
    
    var w1, w2,
		div = $( "<div style='display:block;width:50px;height:50px;overflow:hidden;'><div style='height:100px;width:auto;'></div></div>" ),
		innerDiv = div.children()[0];
    	$( "body" ).append( div );
    	w1 = innerDiv.offsetWidth;
    	div.css( "overflow", "scroll" );
    
    	w2 = innerDiv.offsetWidth;
        //两次值想等的话取clientWidth
    	if ( w1 === w2 ) {
    		w2 = div[0].clientWidth;
    	}
    
    	div.remove();
    
    	return (cachedScrollbarWidth = w1 - w2);
        
>getScrollInfo:  用来判断容器是否有滚动条，并且返回x,y滚动条的宽度值，没有返回0。
>函数首先会获取容器overflow-x,y的值然后判断是否是，scroll或者auto如果是auto的话还需要判断当前容器的可视区长度与容器的scrollWidth做比较来判断容器是否滚动。
    
>getWithinInfo: 获取容器的信息，默认容器为window, 包括信息如下

    var withinElement = $( element || window ),
        isWindow = $.isWindow( withinElement[0] );
	return {
		element: withinElement, //jquery对戏那个
		isWindow: isWindow, //是否为window对象
		offset: withinElement.offset() || { left: 0, top: 0 },
		scrollLeft: withinElement.scrollLeft(),
		scrollTop: withinElement.scrollTop(),
		width: isWindow ? withinElement.width() : withinElement.outerWidth(),
		height: isWindow ? withinElement.height() : withinElement.outerHeight()
	};

前期的工具函数已经准备妥当，接下来就看下它的实现思路。
>首先会判断对of元素进行判断，分为四种情况1.document 2.window 3.event 4.普通文本节点 并获取到他们的内容高度，宽度和offset信息。<br>
>接下来会对my,at 参数进行加工 例如： at: 'left'会加工为at: 'left center', at,my确少的值都会以center补全。 at: 'left+10 bottom+10' 会加工为at: 'left bottom' 然后把偏移量保存到一个变量里<br>
>在接下来会通过上面的值来计算被定位元素的top,left值， 当top,left计算完毕后需要对他进行碰撞检测。最后才会进行offset的设置
