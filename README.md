
#### 介绍


`p-map` 是一个迭代处理 `promise` 并且能控制 `promise` 执行并发数的库。作者是 `sindresorhus`，他还创建了许多关于 `promise` 的库 [promise\-fun](https://github.com)，感兴趣的同学可以去看看。


[之前](https://github.com) 提到的 `p-limit` 也是一个控制请求并发数的库，控制并发数方面，两者作用相同，不过 `p-map` 增加了对请求（`promise`）的迭代处理。


之前 `p-limit` 的用法如下，limit 接受一个函数；



```
var limit = pLimit(8); // 设置最大并发数量为 2
var input = [ // Limit函数包装各个请求
    limit(() => fetchSomething('1')),
    limit(() => fetchSomething('2')),
    limit(() => fetchSomething('3')),
    limit(() => fetchSomething('4'))
];
// 执行请求
Promise.all(input).then(res =>{
    console.log(res)
})

```

而 `p-map` 则通过用户传进来的 `mapper` 处理函数处理的一个集合（准确的说是一个**可迭代对象**）；



```
import pMap from 'p-map';
import got from 'got';

const sites = [
	getWebsiteFromUsername('sindresorhus'), //=> Promise
	'https://avajs.dev',
	'https://github.com'
];

const mapper = async site => {
	const {requestUrl} = await got.head(site);
	return requestUrl;
};

// 接收三个参数，一个是可迭代对象，一个是对可迭代对象进行处理的函数，一个是配置选项
const result = await pMap(sites, mapper, {concurrency: 2});

console.log(result);
//=> ['https://sindresorhus.com/', 'https://avajs.dev/', 'https://github.com/']

```

默认的可迭代对象有[`String`](https://github.com)、[`Array`](https://github.com)、[`TypedArray`](https://github.com):[豆荚加速器](https://yirou.org)、[`Map`](https://github.com)、[`Set`](https://github.com) 、[`Intl.Segments`](https://github.com "此页面目前仅提供英文版本")，而要成为**可迭代**对象，该对象必须实现 `[Symbol.iterator]()` 方法；


遍历可迭代对象时，实际上是根据迭代器协议进行遍历；


比如，迭代一个数组是这样的



```
var iterable = [1,2,3,4]
var iterator = iterable[Symbol.iterator]() // iterator 是迭代器
iterator.next() // {value: 1, done: false}
iterator.next() // {value: 2, done: false}
iterator.next() // {value: 3, done: false}
iterator.next() // {value: 4, done: false}
iterator.next() // {value: undefined, done: true}

```

当数组迭代完成后，会返回 `{value: undefined, done: true}`


`p-pap` 控制并发请求的原理是，对传进来的集合进行迭代，当集合第一个元素（元素可能是异步函数）执行完后，会交给 `mapper` 函数处理，`mapper` 处理完后，才开始迭代下一个元素，这样就保持了按照顺序一个个迭代，此时并发数是`1`；


要做到并发是`n`，并且还能执行上面的迭代，作者很巧妙的用了 `for` 循环



```
for(let i=0;inext()
}

```

#### 手写 `p-map`


下面按照作者思路实现一个 `p-map`，现在有一个需要处理的可迭代对象 `arr`



```
var fetchSomething = (str,ms) =>{
    return new Promise((resolve,reject) =>{
        setTimeout(() =>{
            resolve(parseFloat(str))
        },ms)
    })
}

var arr= [
    fetchSomething('1a' ,1000), // promise
    2,3, 
    fetchSomething( '4a' , 5000), // promise
    5,6,7,8,9,10
]

```

集合第一个元素和第四个元素是一个 promise 函数


`p-map` 接收三个参数，分别是要迭代的对象，`mapper` 处理函数，自定义配置；返回值是 `promise`， 如下面所示



```
var pMap = (iterable, mapper, options) => new Promise((resolve,reject) => {});

```

拿到可迭代对象后，对它进行递归迭代，直至迭代完毕；这里定义一个内部递归迭代函数 `next`



```
var pMap = (iterable, mapper, options) => new Promise((resolve,reject) => {
    var iterator = iterable[Symbol.iterator]()

    var next= ()=>{
        var item=iterator.next()
        if(item.done){
            return
        }
        next()
    }
    
});

```

迭代对象中每个元素都是按顺序迭代的；如果元素是异步函数时，需要先等异步函数兑现，并且兑现后的值传给 `mapper` 函数，等到 `mapper` 函数兑现或者拒绝后才继续迭代下一个元素



```
var iterator = iterable[Symbol.iterator]()
var next = () => {
    var item = iterator.next()

    if (item.done) {
        return
    }

    Promise.resolve(item.value)
        .then(res => mapper(res))
        .then(res2 => {
            next()
        })
}


```

并且每次迭代**彻底完成**后保存兑现的结果



```
var iterator = iterable[Symbol.iterator]()
var index = 0 // 序号，根据可迭代对象的顺序保存结果
var ret = []// 保存结果
var next = () => {
    var item = iterator.next()

    if (item.done) {
        return
    }

    var currentIndex = index //保存当前元素序号，用于存入结果
    index++ //下一个元素的序号
    Promise.resolve(item.value)
        .then(res => mapper(res))
        .then(res2 => {
            ret[currentIndex] = res2
            next()
        })
}

```

当整个迭代完后，并且元素全部执行（兑现）完，输出结果集



```
var pMap = (iterable, mapper, options) => new Promise((resolve,reject) => {
    var activeCount = 0 //正在执行的元素个数
    var next = () => {
        var item = iterator.next()
        if (item.done) {  //元素全部迭代完
            if (activeCount == 0) {
                resolve(ret)  //元素全部执行（兑现）完，输出结果集
            }
            return
        }
        var currentIndex = index // 保存当前元素序号，用干存入结果
        index++ //下一个元素的序号
        activeCount++
        Promise.resolve(item.value)
            .then(res => mapper(res))
            .then(res2 => {
                ret[currentIndex] = res2
                activeCount--
                next()
            })
            .catch(err => {
                activeCount--
            })
    }
})

```

##### 配置项 `stopOnError`


传入 `p-map` 配置项中有一个参数是 `stopOnError`，表示当执行遇到错误，是否终止迭代循环，所以这里在 `.catch()` 里面做判断；



```
Promise.resolve(item.value)
    .then(res = mapper(res))
    .then(res2 => {
        // ...
    })
    .catch(err => {
        ret[currentIndex] == err // 将错误的结果也保存起来
        
        if (stopOnError) {
            hasError = true
            reject(err) // 发生错误，终止循环
        }else {
            hasError = false
            activeCount--
            next()
        }
    }

```

##### 忽略错误执行结果 `pMapSkip`


`mapper` 函数是用户自定义的， 如果 `mapper` 执行错误，用户期望忽略错误执行结果，只保留正确结果，这该怎么做呢？，此时 `pMapSkip` 就登场了；


`p-map` 源码中提供了 `pMapSkip`，`pMap5kip` 是一个 `Symbol` 值，`p-map` 内部处理则是：当结果集收到的结果是 `pMapSkip`，则会在迭代完成后清除返回值是 `pMapSkip` 的元素，也就是说 `mapper` 处理时发生错误， 用户不想要这个值，可以 `reject(pMapSkip)` 比如：



```
import pMap, { pMapSkip } from 'p-map'

var arr = [
    fetchSomething('1a', 1000, true),
    2, 3
]

var mapper = (item, index) => {
    return new Promise((resolve, reject) => {
        return item == 2 ? reject(pMapSkip): resolve(parseFloat(item)) // 元素是 2 ，抛出错误
    })
}

(async () => {
    const result = await pMap(arr, mapper, { concurrency: 2 });
    console.log(result); //=>[1,3]
})();

```

所以当 `mapper` 返回 `pMapSkip` 时，需要标记对应的元素



```
var skipIndexArr= []

```

记录需要剔除的元素的位置



```
var skipIndexArr = [];

Promise.resolve(item.value)
  .then((res = mapper(res)))
  .then((res2) => {
    // ...
  })
  .catch((err) => {
    if (err === pMapSkip) {
      skipIndexArr.push(currentIndex); //记录需要剔除的元素的位置
    } else {
      ret[currentIndex] == err; 

      if (stopOnError) {
        hasError = true;
        reject(err);
      } else {
        hasError = false;
        activeCount--;
        next();
      }
    }
  });

```

并且在迭代结束时剔除结果集中的有 `pMapSkip` 的元素



```
if (item.done) {
  if (activeCount == 0) {
    for (var k of skipIndexArr) {
      ret.splice(k, 1);
    }
    resolve(ret);
    return;
  }
}

```

在数据里大的情况下，频繁使用 `splice` 性能可能没那么好，因为执行 `splice` 后，其后的元素的索引都会改变；那么就要改造下，将 `skipIndexArr` 改为 `Map` 形式。



```
// var skipIndexArr= []
var skipIndexArr= new Map()

```

记录需要删除的元素的位置



```
if (err === pMapSkip) {
  skipIndexArr.set(currentIndex, err);
}

```

然后迭代结束时，不再在原数组里面 `splice` ，改为用新数组接收；`push` 比 `splice` 性能好；



```
if (item.done) {
  if (activeCount == 0) {
    if (skipIndexArr.size === 0) {
      resolve(ret);
      return;
    }

    const pureRet = [];

    for (const [index, value] of ret.entries()) {
      if (skipIndexArr.get(index) === pMapSkip) {
        continue;
      }

      pureRet.push(value);
    }

    resolve(pureRet);
  }
  return;
}


```

##### 在外部取消 `p-map` 的请求或者取消迭代： `AbortController`


存在某些情况，当我们不再需要 `p-map` 返回的结果，或者不再想要使用 `p-map` 时，我们就需要在外部取消 `p-map` 的请求或者取消迭代，这时就可以使用 `AbortController`；


简单介绍下 `AbortController` 的用法，有一个请求 `fetchSomething`



```
var fetchSomething = (str) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(str)
        }, 1000)
    })
}

```

想要取消 `fetchSomething` 请求，就需要传一个 `signal` 到里面；`signal` 是 `AbortController` 的实例属性；`AbortController` 和请求之间就是由 `signal` 建立关联；



```
var controller = new AbortController()
var signal = controller.signal

var fetchSomething = (str,signal) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(str)
        }, 1000)
    })
}

fetchSomething('fetch',signal).then(res => {
    console.log('res:', res)
}).catch(err => {
    console.log('err:', err)
})

```

建立关联后，外部取消请求使用的是 `AbortController` 实例方法 `controller.abort()`；



```
controller.abort()

```

然后在请求里面监听外部是否调用了 `controller.abort()`；有两种方式



```
signal.addEventListener('abort', () => {

}, false)

或者

if (signal.aborted) {

}

```

完整示例：



```
var controller = new AbortController()
var signal = controller.signal

var fetchSomething = (str,signal) => {
    return new Promise((resolve, reject) => {
        signal.addEventListener('abort', () => {
            console.log(' addEventListener')
            reject('addEventListener取消')
        }, false)
        setTimeout(() => {
            if (signal.aborted) {
                console.log('aborted')
                reject('aborted取消')
                return
            }
            console.log('进入setTimeout')
            resolve(str)
        }, 1000)
    })
}

setTimeout(() => {
    controller.abort()
}, 500)

fetchSomething('fetch',signal).then(res => {
    console.log('res:', res)
}).catch(err => {
    console.log('err:', err)
})

```

`500ms` 后输出：



```
addEventListener
err: addEventListener取消
aborted

```

结合 `p-map` 使用如下：



```
import pMap from 'p-map';

const abortController = new AbortController();

setTimeout(() => {
	abortController.abort();
}, 500);

const mapper = async value => value;

await pMap([fetchSomething(1000), fetchSomething(1000)], mapper, {signal: abortController.signal});
// 500 ms 结束 pMap 方法，抛出错误信息.

```

那么 `p-map` 内部实现就好写了：



```
var pMap = (iterable, mapper, options) => new Promise((resolve, reject) => {
    var { signal } = options

    if (signal) {
        if (signal.aborted) {
            reject(signal.reason);
        }

        signal.addEventListener('abort', () => {
            reject(signal.reason);
        });
    }

    var next = () =>{
        //...
    }
})

```

##### 手写的完整源码:


下面的源码也可以把 `promise.then() 和 .catch()` 写法改进为 `async await + try catch` 写法；



```
let pMapSkip = Symbol('skip')

var pMap = (iterable, mapper, options) => new Promise((resolve, reject) => {
    var iterator = iterable[Symbol.iterator]()
    var index = 0 // 序号，根据可迭代对象的顺序保存结果
    var ret = []// 保存结果
    var activeCount = 0 //正在执行的元素个数
    var isIterableDone = false
    var hasError = false
    var skipIndexArr = new Map()
    var { signal, stopOnError, concurrency } = options

    if (signal) {
        if (signal.aborted) {
            reject(signal.reason);
        }

        signal.addEventListener('abort', () => {
            reject(signal.reason);
        });
    }

    var next = () => {
        var item = iterator.next()
        if (item.done) {
            isIterableDone = true
            if (activeCount == 0) {
                if (skipIndexArr.size === 0) {
                    resolve(ret);
                    return;
                }

                const pureRet = [];

                for (const [index, value] of ret.entries()) {
                    if (skipIndexArr.get(index) === pMapSkip) {
                        continue;
                    }

                    pureRet.push(value);
                }

                resolve(pureRet);
            }
            return;
        }
        var currentIndex = index // 保存当前元素序号，用干存入结果
        index++ //下一个元素的序号
        activeCount++
        Promise.resolve(item.value)
            .then(res => mapper(res))
            .then(res2 => {
                ret[currentIndex] = res2
                activeCount--
                next()
            })
            .catch(err => {
                ret[currentIndex] == err;
                if (stopOnError) {
                    hasError = true;
                    reject(err);
                } else {
                    ret[currentIndex] == err;
                    if (err === pMapSkip) {
                        skipIndexArr.set(currentIndex, err);
                    }
                    hasError = false;
                    activeCount--;
                    next();
                }

            })
    }

    for (let k = 0; k < concurrency; k++) {
        if (isIterableDone) {
            break
        }
        next()
    }
})

```

##### 测试一下


1、测试 `pMapSkip`



```
var arr= [
    fetchSomething('1a' ,1000), // promise
    2,3, 
    fetchSomething( '4a' , 5000), // promise
    5,6,7,8,9,10
]
var mapper = (item, index) => {
    return new Promise((resolve, reject) => {
        return item == 3 ? reject(pMapSkip): resolve(parseFloat(item))
    })
}

pMap(arr, mapper, { concurrency: 2 }).then(res => {
    console.log(res)  // [1, 2, 4, 5, 6, 7, 8, 9, 10] ，剔除了 3
})

```

2、测试中止请求



```
var controller = new AbortController()
var signal = controller.signal

pMap(arr, mapper, { concurrency: 2,signal:signal }).then(res => {
    console.log(res)
}).catch(err =>{
    console.log(err) // 500ms 后打印 AbortError: signal is aborted without reason
})

setTimeout(() =>{
    controller.abort()
},500)

```

3、测试 `stopOnError`



```
pMap(arr, mapper, { concurrency: 2,signal:signal,stopOnError:true }).then(res => {
    console.log(res)
}).catch(err =>{
    console.log(err) // Symbol(skip)
})

```

至此，`p-map` 核心功能实现完了；感兴趣的同学可以点点赞；


