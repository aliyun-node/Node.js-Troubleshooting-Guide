# Node.js 应用故障排查手册

本手册主要的目的是帮助广大的 Node.js 开发者应对开发和线上部署中遇到的问题：

* 线上/线下应用出现故障时如何更快地定位问题
* 应用部署前压测中遇到的性能不足问题调优

## 目录

* [序章：本书定位与大纲](0x00_序章.md)
* 第一部分：预备篇 - 案发现场的蛛丝马迹
  * <a href="/0x01_%E9%A2%84%E5%A4%87%E7%AF%87_%E5%B8%B8%E8%A7%84%E6%8E%92%E6%9F%A5%E7%9A%84%E6%8C%87%E6%A0%87.md">常规排查的指标</a>
  * <a href="/0x02_%E9%A2%84%E5%A4%87%E7%AF%87_%E6%A0%B8%E5%BF%83%E8%BD%AC%E5%82%A8%EF%BC%88Core%20dump%EF%BC%89.md">核心转储（Core dump）</a>
* 第二部分：工具篇 - 玩转排查利器
  * <a href="/0x03_%E5%B7%A5%E5%85%B7%E7%AF%87_%E6%AD%A3%E7%A1%AE%E6%89%93%E5%BC%80%20Chrome%20devtools.md">正确打开 Chrome devtools</a>
  * <a href="/0x04_%E5%B7%A5%E5%85%B7%E7%AF%87_Node.js%20%E6%80%A7%E8%83%BD%E5%B9%B3%E5%8F%B0%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97.md">Node.js 性能平台使用指南</a>
* 第三部分：第三部分：实践篇之一 - 专治疑难杂症
  * <a href="/0x05_%E5%AE%9E%E8%B7%B5%E7%AF%87_%E5%88%A9%E7%94%A8%20CPU%20%E5%88%86%E6%9E%90%E8%B0%83%E4%BC%98%E5%90%9E%E5%90%90%E9%87%8F.md">利用 CPU 分析调优吞吐量</a>
  * <a href="/0x07_%E5%86%97%E4%BD%99%E9%85%8D%E7%BD%AE%E4%BC%A0%E9%80%92%E5%BC%95%E5%8F%91%E7%9A%84%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA.md">冗余配置传递引发的内存溢出</a>
* 附录一：更多 Node.js 性能优化实践分享
  * <a href="/0x06_%E5%AE%9E%E8%B7%B5%E7%AF%87_Teambition%20%E5%90%8E%E7%AB%AF%E6%9C%8D%E5%8A%A1%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E6%80%BB%E7%BB%93.md">Teambition 后端服务性能优化总结</a>

// TODO: 未完待续

## 相关专栏

本手册也会在云栖社区同步更新，我们的云栖社区 **Node.js 性能平台** 专栏有更多 Node.js 相关的技术分享，欢迎订阅，地址：https://yq.aliyun.com/teams/210

知乎 Egg.js 核心团队运营的专栏也有许多 Egg.js/Node.js 各路大神的分享，欢迎关注，地址：https://zhuanlan.zhihu.com/eggjs

## 贡献方式

本手册目的是帮助 Node.js 开发者能更好地来应对开发过程中的各种问题，能更多的建立起使用 Node.js 踏足服务端领域的信心，也希望能让天下没有难用的 Node.js。我们非常欢迎大家有合适的案例和最佳实践来和大家一起分享。

参与方法如下：

* Fork 本仓库至你自己的 Github 仓库列表中
* Clone 你 Fork 出来的仓库至本地开发
* 添加你的案例或者最佳实践至合适的位置
* 添加 README.md 中对应的描述，提交至你的远程仓库
* 在 [PR](https://github.com/aliyun-node/Node.js-Troubleshooting-Guide/pulls) 页面选择 **New Pull Request**，继续选择 **compare across forks**，在列表中选中你的 Fork，然后创建新的 PR

我们会在 Review 后选择合并至本仓库内，贡献者也会在 README.md 中的贡献者列表中展示。

## 贡献列表

以下贡献数据来自 `git summary`:

* [hyj1991](https://github.com/hyj1991)  <small>90.0% ( 9 commits )</small>
* [richardwei195](https://github.com/richardwei195) <small>10.0% ( 1 commits )</small>

## LICENSE

本书采用 **保持署名—非商用** 创意共享 4.0 许可证。

只要保持原作者署名和非商用，您可以自由地阅读、分享、修改本书。

详细的法律条文请参见 [创意共享](https://creativecommons.org/licenses/by-nc/4.0/) 网站。

