# Software Practitioners and Technical Debt
## Abstract
- 架构决策是技术债务的重要来源
- 现有工具不足以支撑缓解技术债务细节

## Introduction
技术债务是一个多方面的综合问题，想要有效的解决它需要了解软件演进、风险管理、质量评估、软件指标、项目分析和软件质量等多方面研究。解决技术债务首先要识别项目的痛点，只有深刻理解项目的背景，才能给出有效的改进措施。

这里提出三个问题:
- 通常所讲的技术债务是如何理解的?
- 有多少技术债务实质上时架构问题导致的？
- 业界一般使用什么管理方法或工具来解决技术债务？

经过调研我们发现，一个重要的技术债务来源，是在软件生命周期刚刚开始时的架构决策。而目前缺少能够准确分析架构债务的工具，大家还都在关注代码层面的技术债务。

我们提出了一种时间维度，更容易辨别债务出现和后续导致的问题的方法。

## Discussion
- 技术债务这个概念助于在抽象层面理解逐渐增加的软件开发维护成本
- 除了架构外，大家对于其他可能导致技术债务的原因没有共识
- 架构问题是技术债务的最大来源
- 架构问题因为常年积累，很难解决
- 监测并记录原始架构设计很重要
- 工具难以捕捉到技术债务中累积的关键问题
- 从开发者角度，管理层大多数没有意识到技术债务这个问题，以及管理它的价值

### Technical Debt Timeline
我们提出一种简单时间轴方式，来辅助识别和理解技术债务。

1. 技术债务开始出现，此时很多存在的问题不易识别；
2. 可以识别出部分技术债务。我们发现分析工具大量用于分析代码债务，团队经常被检测结果充斥忙碌；
3. 重新设计架构并解决债务的好时机。此时因债务导致开发成本在逐渐增加。
4. 团队决定清理部分或全部技术债务，此时团队意识到债务导致的低效率。
5. 团队决定有意识管理目前和未来的债务。