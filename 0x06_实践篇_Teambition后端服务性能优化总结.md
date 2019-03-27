# Teambition 后端服务性能优化总结

作者: [richardwei](https://github.com/richardwei195)

> 很少总结 Node.Js 相关的调试以及优化技巧，正好最近在做性能调优并取得了不错的效果，借此机会总结和分享一下

[Teambition](http://teambition.com) 是一款实时协同的跨平台应用，相关的业务接口也是处于读多写少的场景，因此主要的性能瓶颈也是出现在接口平均响应速度慢、不稳定等方面，此次我将结合 `Teambition` 实际业务从以下几个维度来回顾总结优化的整个过程:

- 我们的接口是如何变慢的? 慢在哪里？
- 接口性能和并发操作的连带关系?
- 通过 `Chrome:inspect` 进行代码 `Debug` 及性能分析?
- Node.Js 事件循环模型及单线程为基础带来的优势与不足，如何避免及解决?

## 一、不得不优化的的地步

何谓不得不优化的地步？就是老板(CEO)说 "我觉得我们应用现在很慢，我希望达到秒开的效果"  以及后端工程师我们自己的抱负:  接口这么慢，于情于理都说不过去，多没面子啊，说好的分布式、高性能、高并发呢？

说了这么多我们的接口慢，是如何慢呢？举一个特殊的场景，也是一个比较重要的场景: Teambition 敏捷研发项目下通过 TQL (Teambition 数据查询语言 ) 获取任务列表的场景，贴一下几十个接口并发打过来的效果 (部分):

![](https://pic4.zhimg.com/80/v2-1b054ec97c899cd7dc076e4992459583_hd.jpg)

![](https://pic4.zhimg.com/80/v2-1d212775b627447b1022e68c2970e39f_hd.jpg)

可以明显的看见有部分接口耗时已经不能用毫秒级来衡量了，这个环境是 Teambition 内部的一个测试环境，由于所有应用现在均部署并运行在 k8s 中，当前这个环境为了测试仅启动了一个 pod 并运行单个 Teambition Server 实例保证多个请求均请求至同一个 pod 同一进程中。

作为现代应用，让用户多等待一秒钟可能也会降低用户体验甚至抓狂，因此性能优化也迫在眉睫...

## 二、接口慢，慢在哪里为什么这么慢？

我曾经无数的问过自己，接口为什么这么慢，是数据太多了吗？是计算太多了吗？难不成，是数据库？语言的问题？ 

带着这么多的疑问很久，每次我都努力安慰自己，算了，还是先把业务写完吧…..好不开玩笑，其实这是我一种很久以来的一个心结，我尝试查过，但是一直集中不了零碎的时间导致迟迟没有结果，这次终于狠下心来刮骨疗毒，Teambition 一定要更快更优秀！

接口慢，慢在哪里为什么就可以这么慢，对接口本身可能出现的情况进行分析:

a. 数据量不多就几百条，耗时也不是在传输过程中 。❌

b. 虽然就几百条数据，但是每条数据的逻辑计算和补全可能会很花时间。✅

c. 早就觉得框架中间件的使用方法有问题了。可能问题在这里。✅

d. 也有可能是数据库慢啊。❌

以上是我的四个猜测，应该大多接口性能应该也离不开这几个点，这里我已经表明了最后的结果，接下来我们就一一结合文章开头提出的几个点进行分析、处理。

## 三、解决问题的过程总是那么令人兴奋

1、 **对于框架中间件使用的问题**

   Teambition 服务端是多种语言来进行支撑和拓展的，包括 Node.Js、Go、Java，其中又以 Node 应用最多、应用最广，它支撑起了 Teambition 各产品线的搜索服务 (teambition-soa-search)、通知服务(tb-notification)、数据转换应用 (TIS)、应用中心以及支付应用等等，而 GO 则作为 Teambition 后端 SOA 架构的最底层 (中台) 应用的首选语言。

   Teambition 主站服务是一个大型的(十几万行代码) 的 Node.js 单体应用，框架使用的是 express，框架模型不用我多介绍，中间件是我们代码分层、结构模块化的核心组件，由此也衍生了大大小小大弊端，其中最为重要的则是中间件的串行使用:

   ```js
   // 举一个简单的 Demo, 我们为每一个 API、中间件做了一层封装，使得每一个 API、路由、中间件能够独立在各自的文件维护、并且不重复
   // 其中最为核心的问题是我们将许多后置业务放到中间件中来进行执行，比如
   arr = [
     mdl.toJson(),
     mdl.fillCreator(),
     mdl.fillExecutor(),
     mdl.fill....,
     mdl.emit()...,
     mdl.socket()....
     .....
   ]
   // 是的，我们通过中间件的形式来进行外键的补全、消息同步、事件触发、以及后续的一些业务同步处理
   (req, res, next) => {
     //do...
     next()
   }
   // 中间件传过来的数据始终是同一份，但是我们对这一份数据进行了多次重复计算、串行阻塞处理
   ```

   此为问题一。

2、 **数据本身的读取、处理暴露的问题**

除了中间件使用的问题，剩下的就是数据处理的问题，首先说说数据的查询。

Teambition 数据库主要使用的是 MongoDB，所以从早期到现在我们一直使用的是 `mongoose` ODM，在一开始问题的查找中，我发现停留在 DB 查询的耗时特别久，大概也占到了总耗时的一半，于是我会猜测是否存在 DB 的慢查询，但是在运维同事的协查中并没有在 oplog 中找到相关的一丁点慢查询，那么问题来了，有可能与 ODM 有关，带着猜疑而又激动的心情，果断 Debug 起来，这里顺便分享一下使用 `Chrome inspect` 的调试方法，当然本质还是 Node 本身支持的 [inspector Debug 方案](https://nodejs.org/en/docs/guides/debugging-getting-started/)的封装，同时也安利一下调试工具 [ndb](https://github.com/GoogleChromeLabs/ndb):

a. 启动应用时携带 `--inspect` 参数，terminal 中应该会有如下提示:

```console
// 应用已经通过 websocket 的方式连接至如下地址
Debugger listening on ws://127.0.0.1:9229/3d9f739a-127f-44df-97fb-c7b179350fda
For help, see: https://nodejs.org/en/docs/inspector
```

b. 打开 Chrome 浏览器访问: [chrome://inspect/#devices](chrome://inspect/#devices)

![](https://pic1.zhimg.com/80/v2-f0ff0e8ec3d7a2413296685ef1bf6900_hd.jpg)

![](https://pic2.zhimg.com/80/v2-b10e44538dd20cb870f99fc03a35218d_hd.jpg)

在这里，我们主要查看我们的 CPU 耗时时间，点击开始，访问对应的接口，生成一段时间的 CPU 利用率的时间线快照，大概如下:

![](https://pic2.zhimg.com/80/v2-dea6b8480c85408ea0a95713cce77d7d_hd.jpg)

c. 这里我就不做过多的演示和介绍，相关的调试方法网上及官方文档有对应的介绍，在这里我们主要关注耗时 CPU 耗时最久的函数，也就是 `completeMany` 这个 func 大概花了 2.+s，简直不可思议，但是在代码中并没有发现封装有此方法，再联系之前执行的函数:

![](https://pic3.zhimg.com/80/v2-dab0cee74e48429602b4c4fbff3a6622_hd.jpg)

我们可以清晰地发现，在这之前做了 DB 操作，并且将 mongoDB 的 `BSON` 对象反序列化解析出来，因此猜测后面应该是 mongoose 做了一些额外的操作，这里大概已经能猜到大概了，应该是做了 数据的补全，因为在 mongoose Model Query 的操作中，如果不加 `lean()` 方法的话，mongoose 拿到结果会将数据包装成为我们熟知的 `mongoose Document` ，这也是我们用 mongoose 原因之一，可以轻松为每一个 field 定义默认值、method、以及 validator等等，但前提是这是一个 `Document`，而源码也是做了如此操作:

```js
/*!
 * Given a model and an array of docs, hydrates all the docs to be instances
 * of the model. Used to initialize docs returned from the db from `find()`
 *
 * @param {Model} model
 * @param {Array} docs
 * @param {Object} fields the projection used, including `select` from schemas
 * @param {Object} userProvidedFields the user-specified projection
 * @param {Object} opts
 * @param {Array} [opts.populated]
 * @param {ClientSession} [opts.session]
 * @param {Function} callback
 */

function completeMany(model, docs, fields, userProvidedFields, opts, callback) {
  const arr = [];
  let count = docs.length;
  const len = count;
  let error = null;

  function init(_error) {
    if (_error != null) {
      error = error || _error;
    }
    if (error != null) {
      --count || process.nextTick(() => callback(error));
      return;
    }
    --count || process.nextTick(() => callback(error, arr));
  }

  for (let i = 0; i < len; ++i) {
    arr[i] = helpers.createModel(model, docs[i], fields, userProvidedFields);
    try {
      arr[i].init(docs[i], opts, init);
    } catch (error) {
      init(error);
    }
    arr[i].$session(opts.session);
  }
}
```



而在 Teambition 的大部分接口中，对于数据的获取，通常是数据量不大、接口做了分页等等，所以这个问题没有引起大家的重视，也仅仅有部分查询是用了 `lean()` func，而 mongoose 在做 `createModel()` 操作的时候会为拿到的每一条数据包装上对应的 default value、method等等，其中包含其自定义的默认方法和我们使用时候加上的各类数据，形成一个很大的 `model` 对象，对 CPU 的压力可想而知

当查询使用了 `lean()` 以后，则不会调用 `completeMany()` 

```js
    _this.model.populate(docs, pop, function(err, docs) {
      if (err) return callback(err);
      return mongooseOptions.lean ?
        callback(null, docs) :
        completeMany(_this.model, docs, fields, userProvidedFields, completeManyOptions, callback);
    });
```

优化后，DB 查询速度提高了至少 1.5 倍

3、 **DB、中间件大优化过后，接口响应不稳定**

在前两步的优化中，response time 已经降为了毫秒级 600-700 ms，已经取得了很大的进步了，但是回到我们的业务场景，刷新整个页面在接口并发请求过来的时候，接口响应速度同样为提升至 2-3 s，这让我百思不得其解，单接口的 QPS 性能已经很明显了并且稳定，为何实际并发的场景下响应速度明显变慢了。

有了之前的经验，很快定位到了问题所在，在第一张我们的API请求列表中，还有一个特别耗时的接口—「计数」，仔细排查之后，同样发现该接口计数的计数方式存在不合理性：查询到所有的数据，然后进行分类，在此过程中，`completeMany()` 同样占用了CPU大量的时间。

到这里一切都能够解释的通了，因为 Node.Js 单线程的原因，即便同时能处理多个 I/O，也可能因为 CPU 没有空闲时间导致多个请求 hang up，因为前一个「计数」接口占用的 CPU 过高，导致主线程没有空闲的时间去处理别的计算，导致后续依赖大量计算操作的接口 hang up，可以看到一个很明显的现象就是: 「计数」接口响应完毕之后，waiting 的接口也就跟着返回了—— Node 说这锅我可不背，所以针对计数接口我们做了一定的优化: 主要是 DB 操作以及算法上的优化，这里就不细说了，压测之后，当前场景下整体页面加载效率大概提升了 8-10 倍，算是一个相当惊人的优化了，当然也和我们自己使用不当有关，也算是一个坑吧，目前为止，填坑完毕，看看效果:

![](https://pic3.zhimg.com/80/v2-cf5159faeb16e369f4c40a46f15348a2_hd.jpg)

- 计数接口 TTFB 从近 `5 S` 下降到 `197 ms`
- 任务获取接口 TTFB 从 近 `6S` 下降至 `800+ ms`


## 四、性能是一个后端服务必须要关注的问题

之前的种种优化，伴随着问题的解决，在用户端有了明显的表现，当然，这仅仅是肉眼可以感知的提升，实际在 CPU 占用、内存消耗上面也有了很大的优化和进步，Teambition 现在使用 Node 出现的问题，主要还是在计算上，在面对大量的计算的时候，还是显得比较吃力，因为一个 CPU 同时只能处理一个线程的任务，而一个线程中的任务也只能在同一个 CPU 上进行，不管能够支撑多大的 I/O，也禁不住线程的长时间阻塞，无法利用多核的情况下，我们应当避免可能出现的长时间造成的线程阻塞、耗时计算等阻塞我们的事件循环，试着把耗时操作交给别的服务或者编写一个 C++ 库 去进行计算，这也是众多方案中比较简单粗暴的选择。

![](https://pic2.zhimg.com/80/v2-dac12e2bbeb0ba7c4447a3394c28617d_hd.jpg)

当然，实际的生产环境中，单个线程的阻塞可能并不会有太大的影响，在 Teambition 的 k8s 集群中，布满了成百上千的 Pods，通过 k8s 自带的 [轮询调度算法](https://baike.baidu.com/item/Round%20Robin)，请求基本会被均衡到不同的 pod、不同进程的实例中。但假如这是一个高频的接口或是大部分接口都存在这样的问题呢？势必会拖慢整体应用的平均响应时间，所以，在业务代码的编写过程中，性能优化也是一个需要我们持续关注、不断改进的过程，我相信每一个应用、服务的成长，随着日积月累、业务的增长，会在某些方面表现出一定的颓势，我们应该将这种现象提早治理，刮骨疗毒，才能为以后的业务保驾护航、承载更多的压力和优异的表现。

性能优化的解决途径有很多种，当前使用的方法可能也不是最好的办法，欢迎与我讨论其中的不足与你的宝贵建议。
