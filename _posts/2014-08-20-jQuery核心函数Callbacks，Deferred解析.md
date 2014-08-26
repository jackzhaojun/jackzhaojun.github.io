---
        layout: page
---
---
**jquery@version1.11.1**

jQuery.Callbacks函數是jQuery的核心函数之一，另一个核心函数-延迟对象$.Deferred的实现就是依赖其Callbacks函数。而其他重要的函数如$.ajax,
$.animate的实现都是依赖$.Deferred来实现的。 深入了解$.Callbacks和$.Deferred函数的原理有助于我们可以更高效的使用其实现我们需要的一些功能。

###jQuery.Callbacks
Callbacks是一个多用途的回调列表对象，提供了强大的方式来管理函数列表。先来看一个简单的示例来了解一下基本的API

    function fn1 (value) {
        console.log("fn1：" + value);
    }
    function fn2 (value) {
        console.log("fn2：" + value);
    }
    function fn3 (value) {
        console.log("fn3：" + value);
    }
    var callback = $.Callbacks(); //创建一个回调对象
    callback.add(fn1); //回调列表里添加一个函数
    callback.add(fn2); //回调列表里添加一个函数
    callback.add(fn3); //回调列表里添加一个函数
    callback.remove(fn3); //回调列表里删除一个函数
    callback.fire('触发了'); //触发回调列表函数
    //输出  fn1：触发了  fn2：触发了

通过这个简单的示例，我们大致了解到了Callbacks所能提供的一些功能了。我们可以先大概的想下这个功能该如何实现，Callbacks对象提供了add方法那么对应的
函数内部肯定会有一个存放回调函数的列表list, 调用fire方法触发回调列表其实就是把list列表里的函数拿出来一一执行，当然实际情况要稍微判断一点规则。
我们使用$.Callbacks()获取Callbacks对象的时候 可以传递一个options参数，options参数就是指明这个回调列表的一些规则。我们看下options指定的一些规则。

    //options有如下可选参数   options参数控制回调列表的执行规则
    once: 确保这个回调列表只执行一次
    memory: 保持以前的值，将添加到这个列表的最新函数立即执行
    unique：确保一次只能添加一个回调
    stopOnFalse: 当一个回调返回false时中断后续的所有回调
    //详细示例见 http://api.jquery.com/jQuery.Callbacks/
    jQuery.Callbacks = function( options ) {
        //参数加工 options可以为一个对象也可以为一个string字符串。如果是string的话在使用前也会加工为一个对象
        options = typeof options === "string" ?
        		( optionsCache[ options ] || createOptions( options ) ) :
        		jQuery.extend( {}, options );

        //接下来定义的一些变量基本都是各种状态位标记，callbacks的实现很大程度依赖于这些状态位的标记
        var firing, //标记回调列表是否执行中
            memory, //标记最后一次调用fire（内部函数fire）传递的值
            fired, //标记是否触发过
            firingLength, //当前触发的回调列表实际长度
            firingIndex, //执行当前回调列表的开始下标
            firingStart, //标记列表执行start
            list = [], //实际回调列表
            stack = !options.once && [] //当列表处于执行中状态，新添加的回调存防于stack
            //整个callbacks的实现依赖依赖于两个重要函数fire，和self.add。fire函数主要用于触发回调的执行
            fire = function( data ) {
                memory = options.memory && data; //当前传递参数记录，用于memory规则
                fired = true; //标记状态回调列表是否执行过
                firingIndex = firingStart || 0; //标记回调列表从哪个下标开始执行（主要用于memory规则）
                firingStart = 0;
                firingLength = list.length; //回调列表可执行的长度
                firing = true; //标记执行状态中
                //取出列表中得函数执行，如果是单独memory(没有once规则)规则firingIndex总是等于上一次执行时list.length
                for ( ; list && firingIndex < firingLength; firingIndex++ ) {
                    //stopOnFalse规则过滤
                    if ( list[ firingIndex ].apply( data[ 0 ], data[ 1 ] ) === false && options.stopOnFalse ) {
                        memory = false; // To prevent further calls using add//防止进一步调用使用添加
                        break;
                    }
                }
                firing = false;
                if ( list ) {
                    if ( stack ) {
                        // memory规则
                        if ( stack.length ) {
                            fire( stack.shift() );
                        }
                    } else if ( memory ) {
                        //如果是memory+once规则 每次触发回调后清空list
                        list = [];
                    } else {
                        //once 规则
                        self.disable();
                    }
                }
            },
            接下来看下另一个重要的函数self.add，此函数只要用于想回调列表添加函数
            self = {
                add: function() {
                    if ( list ) {
                        //标记当前执行list长度，用于memory规则
                        var start = list.length;
                        (function add( args ) {
                            jQuery.each( args, function( _, arg ) {
                                var type = jQuery.type( arg );
                                if ( type === "function" ) {
                                    //unique规则过滤
                                    if ( !options.unique || !self.has( arg ) ) {
                                        list.push( arg );
                                    }
                                } else if ( arg && arg.length && type !== "string" ) {
                                    // Inspect recursively
                                    //递归检测 （如何参数是数组递归调用）
                                    add( arg );
                                }
                            });
                        })( arguments );
                        //我们需要添加回调到当前触发的一批（多线程的情况下，如果添加一个回调的时候状态还是执行中就改变当前回调list的长度）
                        if ( firing ) {
                            firingLength = list.length;
                        } else if ( memory ) {
                            //memory规则下，标记下次执行firingStart为当前执行list长度。
                            firingStart = start;
                            fire( memory );
                        }
                    }
                    return this;
                }
                ...
            }
    }

回过头来看下整个callbacks的设计主要是由三部分组成的：

1. 规则 options
2. 回调函数添加
3. 回调函数触发

当我们添加一个函数到回调列表的时候，首先会标记当前回调的list长度， 接下来要把回调函数添加到回调列表的时候我们要拿到**unique**规则来确定是否添加
到当前回调列表, 添加完之后我们要判断是否是**memory**规则，如果是memory规则的话标记firingStart为当前回调列表长度，然后立即调用fire函数。

当我们调用fire函数执行回调的时候，会通过当前标记的firingStart和firingLength来循环回调列表，循环过程中会拿到**stopOnFalse**规则来进行标记，
循环接受之后则是进行无规则、**memory**规则、**memory once**组合规则的判断来进行一些标记，方便下次触发时进行判断。

###jQuery.Deferred

jQuery.Deferred的实现依赖于jQuery.Callbacks，deferred提供了三种动作的管理分别是resolve, reject, notify这三组动作分别用三个callbacks来
管理，向这三组动作添加回调函数则使用done, fail, progress。如何理解resolve，reject，notify这三个动作呢，如果你有一个延迟的操作(比如Ajax操作)
需要deferred来管理的话， 当Ajax响应成功的时候你需要调用resolve来说明延迟操作成功了，resolve执行后deferred会把对应的由done添加的函数拿出来全部执行一次；
相应的reject则是在Ajax响应失败的情况下来调用，同样会把对应的由fail添加的函数拿出来执行一遍； notify则是一个通知动作当Ajax没有发出前可以调用notify。
接下来看下 大概的源码

    Deferred: function( func ) {
    		var tuples = [
    				// 这里标识了三种动作，分别用三个callbacks来管理
    				[ "resolve", "done", jQuery.Callbacks("once memory"), "resolved" ],
    				[ "reject", "fail", jQuery.Callbacks("once memory"), "rejected" ],
    				[ "notify", "progress", jQuery.Callbacks("memory") ]
    			],
    			state = "pending",
    			promise = {
    				...
    			},
    			deferred = {};
    		//遍历tuples, 把resolve，reject，notify以及对对应的done,fail,progress都添加到上一行定义的deferred对象里。
    		jQuery.each( tuples, function( i, tuple ) {
    			var list = tuple[ 2 ], //每个操作对应的callbacks
    				stateString = tuple[ 3 ]; //每个操作对应的状态标识 resolved | rejected

                //done，fail， progress方法添加到promise上
    			promise[ tuple[1] ] = list.add;

    			// Handle state
    			if ( stateString ) {
    			    //添加resolve，reject列表前需要添加三个函数
    			    //1. 状态标识函数
    			    //2. 如何添加resolve列表则把reject列表置为disable，反之亦然
    			    //3. 锁定notify列表
    				list.add(function() {
    					// state = [ resolved | rejected ]
    					state = stateString;

    				// [ reject_list | resolve_list ].disable; progress_list.lock
    				}, tuples[ i ^ 1 ][ 2 ].disable, tuples[ 2 ][ 2 ].lock );
    			}

    			// deferred[ resolve | reject | notify ]
    			deferred[ tuple[0] ] = function() {
    				deferred[ tuple[0] + "With" ]( this === deferred ? promise : this, arguments );
    				return this;
    			};
    			deferred[ tuple[0] + "With" ] = list.fireWith;
    		});

    		// Make the deferred a promise
    		promise.promise( deferred );

    		// Call given func if any
    		if ( func ) {
    			func.call( deferred, deferred );
    		}

    		// All done!
    		return deferred;
    	}

callbacks, deferred的源码整体比较少，但是思路比较精妙。 想深入了解源码的话，首先要把官方API提供的示例都走一遍有个整体了解， 然后再阅读下源码效果会
好很多。
