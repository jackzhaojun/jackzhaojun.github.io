---
category: jQuery
---
###jquery@version1.11.1
<!--more-->
###总体设计

jQuery的Ajax实现依赖于$.Callbacks和$.Deferred（了解Ajax建议先了解<a href="http://t.cn/Rh6oTUs">Callbacks,Deferred</a>）,从总体设计来看可以分三个部分。

    1. prefilters 前置过滤器
    2. transports 请求分发器
    3. 响应处理

**前置过滤器**是啥意思呢？举个栗子 我们想让响应数据类型（dataType）为json的请求在发送前做一些设置，我们可以这样搞。

    jQuery.ajaxPrefilter( "json", function( s ) {
        //各种前置设置
    	s.data = {x:xxx};
    });

这样做的目的可以方便指定数据类型的响应，在发送前做一些统一处理。详细的使用可以参考官方的API jQuery.ajaxPrefilter()。

**请求分发器**又是啥意思呢？ 举个栗子 现在的jQuery.Ajax请求满足不了我的需求该怎么办呢？自己重新实现一个Ajax？但是jQuery封装的Ajax已经帮我们屏蔽了很多
兼容性的问题以及一些隐性bug。这个时候我们就可以用到**请求分发器**jQuery.ajaxTransport()，这个函数需要传两个参数：响应的数据类型，分发函数。
分发函数必须返回一个带有send,abort方法的对象。

    ///为dataType：json添加一个自定义的请求处理，这样的话只要响应数据类型为json的请求都会进入到这个请求分发器，
    jQuery.ajaxTransport( "json", function(options) {
        var callback;
        return {
            //headers客户的设置的请求头和默认的请求头，complete jQuery默认响应处理函数
            send: function (headers, complete) {
                //此处可参照xhr.js
                //xhr已经通过ajaxSettings设置到所有请求的options中
                var xhr = options.xhr();
                xhr.open(options.type, options.url, options.async, options.username, options.password);
                xhr.send();
                //...
            },
            abort: function () {
                //...
            }
        }
    } )；

**响应处理**jQuery.Ajax的回调是通过$.Deferred，和$.Callbacks来管理的。阅读源码会发现options.success,options.error
都会被添加到deferred.done与deferred.fail函数里。那Callbacks是用来干嘛的呢？ 先看这面这句代码

    deferred = jQuery.Deferred(),
	completeDeferred = jQuery.Callbacks("once memory"),
    deferred.promise( jqXHR ).complete = completeDeferred.add;
    for ( i in { success: 1, error: 1, complete: 1 } ) {
        jqXHR[ i ]( s[ i ] );
    }

completeDeferred就是用来管理一个通用的回调。不管响应是否成功或者失败completeDeferred.fireWith都会去执行。表现在实际的应用中就是这样的

    $.ajax(options)
        .done(function () {})//只有成功的时候才会执行
        .fail(function () {})//只有失败的时候才会执行
        .complete(function () {}) //不管成功失败都会执行

jQuery.ajax的整体设计思路就是这样子的。当然还有很多细节的问题没有说到，后续继续分析细节问题