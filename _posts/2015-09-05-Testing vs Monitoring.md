---
layout: post
title: 译-Episode 388:Testing vs Monitoring 测试vs监控
date: 2015-09-05 18:29
disqus: y
---

[Tott的文章](https://testerhome.com/topics/3245)

A user-facing bug causes search results to be unavailable for your service. Someone suggests adding a prober to monitor the service and, if search results are unavailable, notify the team via bug report. Google has plenty of prober options available; if we pick one and use it we’re done, right?

一个用户交互缺陷使你的服务得不到搜索结果。有人建议添加一个探测器监控服务，在无法搜索时把缺陷报告通知给团队。谷歌有大量的探测器可供选择；我们挑选一个使用，就可以了吗？

Not necessarily. Just because monitoring could detect a bug does not mean it is the best, or only, solution. For any given bug, you should consider which mixture of monitoring and testing is appropriate. Monitoring and testing each have pros and cons, and solve slightly different problems.
Monitoring observes—and sometimes interacts with—user-facing production systems. Monitoring is useful for detecting:

未必。监控可以发现缺陷不意味着它就是最好的或唯一的解决方案。对任何给定的缺陷，你需要考虑怎样混合使用监控和测试是恰当的。监控和测试各有利弊，可以解决的问题有微妙的不同。
监控观察面向用户的生产系统，有些时候和生产系统交互。监控在寻找下述问题时是有帮助的：

- Load issues: Real users accessing real services induces load on real servers. The only way to measure the effect of production traffic is to directly measure the servers themselves. Common measurements include QPS, RPC response times, memory usage, and disk usage.     
    

- 负载问题：真用户访问实时服务会在真服务器上产生负载。衡量产品业务效果的唯一方式是直接通过服务器测量。常见数据有QPS，RPC响应时间，内存和磁盘的使用情况。 

- Service unavailability: A service might become unavailable because (1) the service itself is down, or (2) the services on which it depends are unavailable. Monitoring end-user experiences (e.g., “Does web search return results?”) is a great way to detect and alert about this situation.

- 服务不可用：原因可能是（1）它自己挂了，或（2）它依赖的服务不可用。监控使用者的体验（例如，“网页搜索返回结果了吗？”），在这种情况下发出警报是一个很好的方式。

- Unanticipated user behavior: Even the most well-designed test scenarios can fail to anticipate real user behavior. Monitoring can inform your quality strategy by observing and measuring real-world behavior.

- 预料之外的用户行为：再精心设计的测试场景也不能预测真实用户的所有行为。监控通过观察和测量控真实世界的行为，给你的质量策略提供情报。
            
- Version incompatibility: Different binary versions may interact incompatibly in ways that are hard to detect without production data. Monitoring can detect unanticipated data inconsistencies.

- 版本不兼容：没有生产环境的数据很难发现不同版本的二进制包产生的兼容性问题。监测可以发现没预想到的数据不一致。

- Data changes: User-facing data can change over time, sometimes in bad ways. Monitoring can statistically characterize data, diff them against previous data state, and alert on outside-of-threshold changes.

- 数据更改：随着时间推移，用户数据可能以恶劣的方式更改。监控可以统计特征数据，和以前的数据对比，提醒阈值外的更改。

Testing isolates components in a non-production environment and verifies components’ behavior. Since it occurs prior to release, it reduces the cost of fixing a bug. Testing is useful for ensuring:

测试，在隔离组件的非生产环境验证组件行为，由于这在在发布之前，测试节省了修复缺陷的成本。测试在确保下述方面时是有帮助的：

- Functional correctness: A hermetic unit test remains the best way to prove that a small piece of code logic fulfills its interface contract.

- 功能正确性：一个和其他部分隔离的单元测试仍然是证明一小段代码逻辑实现了接口约定的最佳方法。

- Inter-component compatibility: An integration test is an excellent way to ensure that two components (e.g., a client and a Stubby service) work together properly.

- 组件间兼容性：集成测试是一个很好的方式用于确保两个组件（例如，一个客户端和一个 stubby 服务）能在一起正常的工作。

When considering which techniques to employ, review the list above to determine which ones are appropriate. Worried about an individual vendor’s ad inventory suddenly dropping? Monitor ad volume for each vendor! Unsure if the price2value() function handles currency conversions? Write a unit test! Not sure how often users actually log into your system? Monitor login events! A judicious mix of monitoring and testing will speed up development and ensure that fewer bugs reach end users.

考虑使用哪种技术时，看看上面的列表，决定哪个更合适。担心一个商家的广告费用突然下降？监控每个商家的广告份额！不确定 price2value() 函数有没有处理货币转换？写一个单元测试！不知道用户多久登录一次？监控登录事件！明智的组合监控和测试可以加快开发，确保到达最终用户的缺陷更少。
