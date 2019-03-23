# Node.js 性能平台使用指南


<a name="7ac42f43"></a>
## 楔子
前一节中我们借助于 Chrome devtools 实现了对线上 Node.js 应用的 CPU/Memory 问题的排查定位，但是在实际生产实践中，大家会发现 Chrome devtools 更加偏向本地开发模式，因为显然 Chrome devtools 不会负责去生成分析问题所需要的 Dump 文件，这意味着开发者还得额外在线上项目中设置好 [v8-profiler](https://www.npmjs.com/package/v8-profiler) 和 [heapdump](https://www.npmjs.com/package/heapdump) 这样的工具，并且通过额外实现的服务来能够去对线上运行的项目进行实时的状态导出。

加上实际上预备章中除了 CPU/Memory 的问题，我们还会遇到一些需要分析错误日志、磁盘和核心转储文件等才能定位问题的状况，因此在这些场景下，仅仅靠 Chrome devtools 显然会有一些力不从心。正是为了解决广大 Node.js 开发者的这些痛点，我们在这里推荐大家在使用 [Node.js 性能平台](https://www.aliyun.com/product/nodejs)，即原来的 AliNode，它已经在阿里巴巴集团内部承载了几乎所有的 Node.js 应用线上运行监控和问题排查，因此大家可以放心在生产环境部署使用。<br /><br /><br />本节将从 [Node.js 性能平台](https://www.aliyun.com/product/nodejs) 的设计架构、核心能力以及最佳实践等角度，帮助开发者更好地使用这一工具来解决前面提到的异常指标分析和线上 Node.js 应用故障定位。<br /><br />
<a name="0eaa6af9"></a>
## 架构
Node.js 性能平台其实简单的说由三部分组成：**云控制台** +** AliNode runtime** + **Agenthub**，如下图所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552530575156-249d5bf6-aa4b-4bde-8978-6efa7c37c2d1.png#align=left&display=inline&height=444&name=image.png&originHeight=932&originWidth=1228&size=289984&status=done&width=585)<br />具体的部署步骤可以查看官方文档：[自助式部署 Runtime](https://help.aliyun.com/document_detail/60902.html)。借助于 Node.js 性能平台的整套解决方案，我们可以很方便地实现预备章节中提到的绝大部分异常指标的告警分析的能力。在生产实践过程中，实际上在笔者看来，Node.js 性能平台解决方案其实仅仅是提供了三个最核心却也是最有效的能力：
* **异常指标告警**，即预备节中一些异常指标出现异常时能通过短信/钉钉通知到开发者
* **导出线上 Node.js 应用状态**，包括但不限于即前面 Chrome devtools 一节中提到的 CPU/Memory 状态导出
* **在线分析结果和更好的 UI 展示**，定制化解析应用导出状态和展示，更符合国内开发者习惯

换言之，Node.js 性能平台作为一个产品本身功能也在不断迭代新增修改中，但是以上的三个核心能力一定是第一优先级保障的，其它边边角角的功能则相对来说响应优先级没有那么高。

实际上我们也理解作为使用平台的开发者希望能在一个地方看到 Node.js 线上应用从底层到业务层的所有细节，然而我个人感觉不同的工具都有应该有其核心的能力输出，很多时候不断做加法容易让产品本身定位模糊化以及泛而不精，Node.js 性能平台实际上始终在致力于让原本线上黑盒的运行时状态，能更加直观地反馈给开发者，让 Node.js 应用的开发者面对一些偏向底层的线上疑难杂症能够不再无所适从。

<a name="34062b25"></a>
## 最佳实践
<a name="05901149"></a>
### I. 配置合适的告警
线上应用的告警实际上是一种自我发现问题的保护机制，如果没有告警能力，那每次都会等到问题暴露到用户侧导致其反馈才能发现问题，这显然对用户体验非常的不友好。

因此部署完成一个项目后，开发者首先需要去配置合适的告警，而在我们的生产实践中，线上问题通过错误日志、Node.js 进程 CPU/Memory 的分析、核心转储（Core dump）的分析以及磁盘分析能够得出结论，因此我们需要的基本的告警策略也是源自以上五个部分。幸运的是平台已经给我们预设好了这些告警，大家只需要选择一下即可完整这里的告警配置，如下图所示：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552533917519-1687a2cf-3a6d-414f-b874-f9a7b4b8afbd.png#align=left&display=inline&height=517&name=image.png&originHeight=1034&originWidth=2362&size=356241&status=done&width=1181)

在 [Node.js 性能平台](https://www.aliyun.com/product/nodejs) 的告警页面上有 **快速添加规则**，点开选中后会自动生成告警规则的阈值表达式模板和报警说明模板，我们可以按照项目实际监控需求进行修改，比如想要对 Node.js 进程的堆内存进行监控，可以选中 **Memory 预警** 选项，如下图所示：

![](https://cdn.nlark.com/yuque/0/2019/png/155185/1552534297094-3830b72d-b51e-4774-b0db-5ac3beabe5a0.png#align=left&display=inline&height=320&originHeight=948&originWidth=2208&size=0&status=done&width=746)

此时点击 **添加报警项** 即完整了对进程堆内存的告警，并且将出现告警时需要点击 **通知设置->添加到联系人列表** 来添加的联系人加入此条规则，如下图所示：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552546534824-dc090160-cf7d-4aca-9552-66bcc40ed9ff.png#align=left&display=inline&height=131&name=image.png&originHeight=262&originWidth=2714&size=178611&status=done&width=1357)<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552546466183-b2ff5838-4c13-4f3d-9a44-90fcfab5b6a2.png#align=left&display=inline&height=396&name=image.png&originHeight=792&originWidth=2016&size=207599&status=done&width=1008)

那么在例子中的这条默认的规则里，当我们的 Node.js 进程分配的堆内存超过堆上线的 80%（默认 64 位机器上堆上限是 1.4G）时会触发短信通知到配置绑定到此条规则的联系人。

实际上快速添加规则列表中给大家提供的是最常见的一些预配置好的告警策略，如果这些尚不能满足你的需求，更多定制化的自定义的服务告警策略配置方法可以看官方文档 [报警设置](https://help.aliyun.com/document_detail/60494.html)。并且除了短信告警，也支持钉钉机器人推送告警消息到群，方便群发感知线上 Node.js 应用态势。

<a name="be962bc6"></a>
### II. 按照告警类型进行分析
按照 I 节中配置完成合适的告警规则后，那么当收到告警短信时就可以按照策略类型进行对应的分析了。本节将按照预备节中比较常见的五大类问题来逐一讲解。
<a name="d41d8cd9"></a>
#### 
<a name="b4279098"></a>
#### a. 磁盘监控
这个是比较好处理的问题，在快速添加的规则里实际上我们会在服务器的磁盘使用超过 85% 时进行告警，那么收到磁盘告警后，可以连接到服务器，使用如下命令查看那个目录占用比较高：

```bash
sudo du -h --max-depth=1 /
```

找到占比比较高的目录和文件后，看看是否需要备份后删除来释放出磁盘空间。
<a name="d41d8cd9-1"></a>
#### 
<a name="b96da0ed"></a>
#### b. 错误日志
收到特定的错误日志告警后，只需要去对应的项目的 [Node.js 性能平台](https://www.aliyun.com/product/nodejs) 控制台找到问题 **实例** 去查看其 **异常日志** 即可，如下图所示：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552550040426-0a7fee02-0445-4764-92b8-c3fb16f58e9d.png#align=left&display=inline&height=87&name=image.png&originHeight=174&originWidth=1626&size=98734&status=done&width=813)

这里会按照错误类型进行规整，大家可以结合展示的错误栈信息来进行对应的问题定位。注意这里的错误日志文件需要你在部署 Agenthub 的时候写入配置文件，详细可以参见文档 [配置和启动 Agenthub](https://help.aliyun.com/document_detail/60902.html) 一节中的 **详细配置** 内容。
<a name="d41d8cd9-2"></a>
#### 
<a name="63230030"></a>
#### c. 进程 CPU 高
终于到了前一节中借助 [v8-profiler](https://www.npmjs.com/package/v8-profiler) 导出 CPU Profile 文件再使用 Chrome devtools 进行分析的异常类型了。那么在 [Node.js 性能平台](https://www.aliyun.com/product/nodejs) 的整套解决方案下，我们并不需要额外的去依赖类似 v8-profiler 这样的第三方库来实现进程状态的导出，与此相对的，当我们收到 Node.js 应用进程的 CPU 超过我们设置的阈值告警时，我们只需要在控制台对应的 **实例** 点击 **CPU Profile** 按钮即可：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552550958706-9d5f5da4-be38-4388-9810-49ae695fde62.png#align=left&display=inline&height=175&name=image.png&originHeight=350&originWidth=578&size=83233&status=done&width=289)

默认会给抓取的进程生成 3 分钟的 CPU Profile 文件，等到结束后生成的文件会显示在 **文件** 页面：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552551250695-743e2e25-2dbd-4346-a6b4-25fff4cb2dbc.png#align=left&display=inline&height=154&name=image.png&originHeight=308&originWidth=2692&size=232999&status=done&width=1346)

此时点击 **转储** 即可上传到云端以供在线分析展示了，如下图所示：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552551276064-82f5c8b5-cbee-451e-9111-3b998c0d6508.png#align=left&display=inline&height=157&name=image.png&originHeight=314&originWidth=2692&size=275904&status=done&width=1346)

这里可以看到有两个 **分析** 按钮，其实第二个下标带有 **(devtools)** 的分析按钮实际上就是前一节中提到的 Chrome devtools 分析，这里不再重复讲解了，如果有遗忘的同学可以再去回顾下本大章前一节的内容。我们重点看下第一个 AliNode 定制的分析，点击第一个分析按钮后，可以在新页面看到如下所示内容：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552551330816-828f166d-31db-4464-b38c-9a9786ae7099.png#align=left&display=inline&height=403&name=image.png&originHeight=806&originWidth=2398&size=1259526&status=done&width=1199)

这里其实也是火焰图，但与 Chrome devtools 提供的火焰图不一样的地方在于，这里是将抓取的 3 分钟内的 JS 函数执行进行了聚合展示出来的火焰图，在一些存在多次执行同一个函数（可能每次执行非常短）的情况下，聚合后的火焰图可以很方便的帮助我们找到代码的执行瓶颈来进行对应的优化。

值得一提的是，如果你使用的 AliNode runtime 版本在 **v3.11.4** 或者 **v4.2.1** 以上（包含这两个版本）的话，当你的应用出现类死循环问题，比如由于异常的用户参数导致的正则回溯（即执行完这个正则要十几年，类似于 Node.js 进程发生了死循环）这类问题时，可以通过抓取 CPU Profile 文件来很方便地定位到问题代码，详细信息有兴趣的同学可以看下 [Node.js 性能平台支持死循环和正则攻击定位](https://yq.aliyun.com/articles/610395)。
<a name="d41d8cd9-3"></a>
#### 
<a name="0a9ad162"></a>
#### d. 内存泄漏
与 CPU 高的问题一样，当我们收到 Node.js 应用进程的堆内存占据堆上限的比率超过我们设置的阈值时，我们也不需要类似 [heapdump](https://www.npmjs.com/package/heapdump) 这样的第三方模块来导出堆快照进行分析，我们还是在控制台对应的 **实例** 点击 **堆快照** 按钮即可生成对应 Node.js 进程的堆快照：<br /><br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552552073249-50f12176-5bee-496c-bb9a-2ec6393e87b3.png#align=left&display=inline&height=161&name=image.png&originHeight=322&originWidth=526&size=80354&status=done&width=263)

生成的堆快照文件同样会显示在 **文件** 列表页面，点击 **转储** 将堆快照上传至云端以供接下来的分析：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552552952658-66dc94ed-5d19-47cf-9b2a-7553160ab831.png#align=left&display=inline&height=153&name=image.png&originHeight=306&originWidth=2692&size=275136&status=done&width=1346)

与上面一样，下标带有 **(devtools)** 的分析按钮还是前一节中提到的 Chrome devtools 分析，这里还是着重解析下 AliNode 定制的第一个分析按钮，点击后新页面如下图所示：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552553672894-f536d47c-0b2f-40a0-b65f-5f1272249d4e.png#align=left&display=inline&height=340&name=image.png&originHeight=680&originWidth=2726&size=377823&status=done&width=1363)

首先解释下上面的总览栏目的内容信息：
* **文件大小：** 堆快照文件本身的大小
* **Shallow Szie 总大小：** 回顾下上一节中的内容，GC 根的 Retained Size 大小其实就是堆大小，也等于堆上分配的所有对象的 Shallow Size 总大小，因此这里其实就是已使用的堆空间
* **对象个数：** 当前堆上分配的 Heap Object 总个数
* **对象边个数：** 这个稍微抽象一些，假如 Object A.b 指向另一个 Object B，我们则认为表示指向关系的 b 属性就是一条边
* **GC Roots 个数：** V8 引擎实现的堆分配，其实并不是我们之前为了帮助大家理解简化的只有一个 GC 根的情况，在实际的运行模型下，堆空间内存在许多的 GC 根，这里是真实的 GC 根的个数

这部分的信息旨在给大家一个概览，部分信息需要深入解读堆快照才能彻底理解，如果你实在无法理解其中的几个概览指标信息，其实也无伤大雅，因为这并不影响我们定位问题代码。

简单了解了概览信息的含义后，接着我们来看对于定位 Node.js 应用代码段非常重要的信息，第一个是默认展示的 **可疑点** 信息，上图中的内容表示 @15249 这个对象占据了堆空间 97.41% 的内存，那么它很可能就是一个泄漏对象，这里又存在两种可能：
* 此对象本身应该被释放但是却没有释放，造成堆空间占用如此大
* 此对象的某些属性应该被释放但是却没有释放，造成表象是此对象占据大量的堆空间

要判断是哪一种情况，以及追踪到对应的代码段，我们需要点击图中的 **簇视图** 链接进行进一步观察：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552554573705-c2ab7c8d-68ae-4533-a11f-cac143c01d9b.png#align=left&display=inline&height=558&name=image.png&originHeight=1116&originWidth=2716&size=626089&status=done&width=1358)

这里继续解释下什么是簇视图，簇视图实际上是支配树的一个别名，也就是这个视图下我们看到的正是前面一节中提到的从可疑泄漏对象出发的支配树视图，它的好处是，在这个视图下，父节点的 Retained Size 可以直接由其子节点的 Retained Size 累加后再加上父节点自身的 Shallow Size 得到，换言之，在这个视图下我们层层展开即可以看到可疑泄漏对象的内存究竟被哪些子节点占用了。

并且结合前一节的支配树描述，我们可以知道支配树下的父子节点关系，并不一定是真正的堆上空间内的对象父子关系，但是对于那些支配树下父子关系在真正的堆空间内也存在父子节点关系的簇节点，我们将真正的 **边** 也用浅紫色标识出来，这部分的 **边** 信息对于我们映射到真正的代码段非常有帮助。在这个简单的例子中，我们可以很清晰的看到可疑泄漏对象 @15249 实际上是下属的 test-alinode.js 中存在一个 array 变量，其中存储了四个 45.78 兆的数组导致的问题，这样就找到了问题代码可以进行后续优化。

而在实际生产环境的堆快照分析下，很多情况下簇视图下的父子关系在真实的堆空间中并不存在，那么就不会有这些紫色的边信息展示，这时候我们想要知道可疑泄漏对象如何通过 JavaScript 生成的对象间引用关系引用到后面真正占据掉堆空间的对象（比如上图中的 40 多兆的 Array 对象），我们可以点击 **可疑节点自身的地址链接** ：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552555184344-caf95c4a-3d7e-4da9-bca2-b9523135ab4f.png#align=left&display=inline&height=234&name=image.png&originHeight=468&originWidth=2742&size=257019&status=done&width=1371)

这样就进入到以此对象为起点的堆空间内真正的对象引用关系视图 **Search 视图**：

![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552555278928-61e5fcf7-2c52-4e32-8387-f9128f9073c5.png#align=left&display=inline&height=442&name=image.png&originHeight=884&originWidth=2710&size=497787&status=done&width=1355)

这个视图因为反映的是堆空间内各个 Heap Object 之间真正的引用连接关系，因此父对象的 Retained Size 并不能直接由子节点的 Retained Size 累加获取，如上图红框内的内容，显然这里的三个子节点 Retained Size 累加已经超过 100%，这也是 Search 视图和簇视图很大的不同点。借助于 Search 视图，我们可以根据其内反映出来的对象和边之间的关系来定位可疑泄漏对象具体是在我们的 JavaScript 代码中的哪一部分生成。

其实看到这边，一些读者应该意识到了这里的 Search 视图实际上对应的就是前一节中提到的 Chrome devtools 的 Containment 视图，只不过这里的起始点是我们选中的对象本身罢了。

最后就是需要提一下 **Retainers 视图**，它和前一节中提到的 Chrome devtools 中解析堆快照展示结果里面的 Retainers 含义是一致的，它表示对象的父引用关系链，我们可以来看下：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552555684039-d787467f-22d7-4116-834a-3f426621cc48.png#align=left&display=inline&height=358&name=image.png&originHeight=716&originWidth=2734&size=382783&status=done&width=1367)

这里 globa@1279 对象的 clearImmediate 属性指向 timers.js()@15325，而 timers.js()@15325 的 context 属性指向了可疑的泄漏对象 system / Context@15249。

在绝大部分的情况下，通过结合 **Search 视图** 和 **Retainers 视图** 我们可以定位到指定对象在 JavaScript 代码中的生成位置，而 **簇视图** 下我们又可以比较方便的知道堆空间被哪些对象占据掉了，那么综合这两部分的信息，我们就可以实现对线上内存泄漏的问题进行分析和代码定位了。
<a name="d41d8cd9-4"></a>
#### 
<a name="e68c5c71"></a>
#### e. 出现核心转储
最后就是收到服务器生成核心转储文件（Core dump 文件）的告警了，这表示我们的进程已经出现了预期之外的 Crash，如果你的 Agenthub 配置正常的话，在 **文件** -> **Coredump 文件** 页面会自动将生成的核心转储文件信息展示出来：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552557614675-853f8851-1411-4cf2-8464-5f18b3778835.png#align=left&display=inline&height=230&name=image.png&originHeight=460&originWidth=2840&size=438868&status=done&width=1420)

和之前的步骤类似，我们想要看到服务端分析和结果展示，首先需要将服务器上生成的核心转储文件转储到云端，但是与之前的 CPU Profile 和堆快照的转储不一样的地方在于，核心转储文件的分析需要我们提供对应 Node.js 进程的启动执行文件，即 AliNode runtime 文件，这里简化处理为只需要设置 Runtime 版本即可：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552557818836-5d071f2e-edda-4b41-a6e2-6116ed8bd924.png#align=left&display=inline&height=155&name=image.png&originHeight=310&originWidth=2690&size=215536&status=done&width=1345)

点击 **设置 runtime 版本** 即可进行设置，格式为 `alinode-v{x}.{y}.{z}` 的形式，比如 alinode-v3.11.5，版本会进行校验，请**务必填写你的应用真实在使用的 AliNode runtime 版本**。版本填写完成后，我们就可以点击 **转储** 按钮进行文件转储到云端的操作了：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552558071285-ef2c68e3-59f6-49ae-bfad-3d7274837387.png#align=left&display=inline&height=161&name=image.png&originHeight=322&originWidth=2696&size=244200&status=done&width=1348)

显然对于核心转储文件来说，Chrome devtools 是没有提供解析功能的，所以这里只有一个 AliNode 定制的分析按钮，点击这个 **分析** 按钮，即可以看到结果：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552558151404-01c27802-0195-4407-b220-26c2e68d7e93.png#align=left&display=inline&height=523&name=image.png&originHeight=1046&originWidth=2558&size=603577&status=done&width=1279)

这里第一栏的概览信息看文字描述就能理解其含义，所以这里就不再多做解释了，我们来看下比较重要的默认视图 **BackTrace 信息视图**，此视图下展示的实际上是 Node.js 应用在 Crash 时刻的线程信息，许多开发者认为 Node.js 是单线程的运行模型，其实这句话也不是完全错误，更准确的说法是 **单主 JavaScript 工作线程**，因为实际上 Node.js 还会开启一些后台线程来处理诸如 GC 里的部分任务。

绝大部分的情况下，应用的 Crash 都是由 JavaScript 工作线程引发的，因此我们需要关注的也仅仅是这个线程，这里显然 BackTrace 信息视图中将 JavaScript 工作线程做了标红和置顶处理，展开后可以看到 Node.js 应用 Crash 那一刻的错误堆栈信息：<br />
![image.png](https://cdn.nlark.com/yuque/0/2019/png/155185/1552558475521-233915a4-bc55-4a1e-adc2-758adddd4429.png#align=left&display=inline&height=462&name=image.png&originHeight=924&originWidth=1988&size=782494&status=done&width=994)

因为就算在 JavaScript 的工作线程中，也会存在 Native C/C++ 代码的穿透，但是在问题排查中我们往往只需要去看同样标红的 JavaScript 栈信息即可，像在这个简单的例子中，显然 Crash 是因为视图去启动一个不存在的 JS 文件导致的。

值得一提的是，核心转储文件的分析功能非常的强大，因为在预备节中我们提到其生成的途径除了 Node.js 应用 Crash 的时候由系统内核控制输出外，还可以由 `gcore` 这样的命令手动强制输出，而本小节我们又看到核心转储文件的分析实际上可以看到此刻的 JavaScript 栈信息以及其入参，结合这两点，我们可以在线上出现 CPU Profile 一节中提到的类死循环问题时直接采用 `gcore` 生成核心转储文件，然后上传至平台云端进行分析，这样不仅可以看到我们的 Node.js 应用是阻塞在哪一行的 JavaScript 代码，甚至引发阻塞的参数我们也能完整获取到，这对本地复现定位问题的帮助无疑是无比巨大的。

<a name="d1fb6ef9"></a>
## 结尾
本节其实给大家介绍了 [Node.js 性能平台](https://www.aliyun.com/product/nodejs) 的整套面向 Node.js 应用开发的监控、告警、分析和定位问题的解决方案的架构和最佳实践，希望能让大家对平台的能力和如何更好地结合自身项目进行使用有一个整体的理解。

限于篇幅，最佳实践中的 CPU Profile、堆快照和核心转储文件的分析例子都非常的简单，这部分的内容也更多的是旨在帮助大家理解平台提供的工具如何使用以及其分析结果展示的指标含义，那么本书的第三节中，我们会通过一些实际的生产遇到的案例问题借助于 [Node.js 性能平台](https://www.aliyun.com/product/nodejs) 提供的上述工具分析过程，来帮助大家更好的理解这部分信息，也希望大家在读完这些内容后能有所收获，能对 Node.js 应用在生产中的使用更有信心。
