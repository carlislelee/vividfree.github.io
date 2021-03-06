---
layout: post
title: 面向稀有事件的 Logistic Regression 模型校准
category: 机器学习
tagline: 
tags: ["logistic regression", LR, 逻辑回归, "rare event", 稀有事件, intercept, 截距, "calibration", 校准]
theme :
  name : bootstrap-3
---
{% include JB/setup %}

## 1. 引言

对分类问题的研究大部分是在样本分布均衡的情况下开展的。比如对二分类，一般研究的是正样本和负样本的数量相当（比如各占50%）或者相差不是那么大（比如一类样本占30%，另一类样本占70%）。但在实际应用时，经常会碰到样本倾斜问题，对于二分类而言，就会是某类的样本比例远小于另一类的样本比例。在学术界，一般称样本少的那类为正类，称正类代表的随机事件为稀有事件。稀有事件的例子并不少，比如发生战争的概率，染上某种不常见疾病的概率，磁盘出现故障的概率，网络传输数据丢失的概率等。在互联网领域，搜索业务、个性化推荐业务和广告定向投放业务都会碰到一个正样本比例远小于负样本比例的例子，那就是点击率预估。很多内容或者广告的点击率在万分位到千分位之间，存在点击的内容或者广告是正样本，未被点击的内容或者广告是负样本。比如说某广告的点击率为千分之一，这就是说平均投放出1000次广告，才有1次点击，剩下的都是未点击。

无论在学术界，还是在工业界，Logistic Regression(LR, 逻辑回归)模型[1]是常用的分类模型，被用于各种分类场景和点击率预估问题等，它也是Max Entropy(ME, 最大熵)模型[2]，或者说Softmax Regression模型[3]，在二分类的一种特例。

用\\(X\ =\ (x_1, x_2, ..., x_k)\\)表示*k*维样本，用\\(\mathbf{\beta} = (\beta_0, \beta_1, \beta_2, ..., \beta_k)\\)表示*k+1*维模型变量，其中\\(\beta_0\\)表示截距项，那么LR模型的model function如下表示：

\begin{equation}\mathbf{f}(z) = \frac 1{1 + e^{-z}}\end{equation}

\begin{equation}z = \beta_0 + \beta_1 \times x_1 +  \beta_2 \times x_2 + ... +  \beta_k \times x_k\end{equation}

说明：公式1的学名为logistic function。

通常情况下，准备好标注样本和特征后，经过参数训练，最终会得到机器学习模型，对LR模型而言，得到的是分类超平面。但样本不均衡的情况下，通常的参数训练方法会带来模型估计上的偏差。对LR模型而言，通常的参数训练方法会造成正样本的估计概率低于真实值。论文[4]在5.1一节，以及书籍[5]在13.5.3一节，对此作了论述。在文献中从只有1个特征且该特征的特征权重大于0的LR模型出发来解释。在均值和方差都未知的情况下，用极大似然估计（Maximum Likelihood Estimator, MLE）估计高斯分布的方差是有偏(bias)但是满足一致性(consistency)[6][7]，MLE估计出的方差的分母是样本数，但方差的无偏估计是样本数减1 [8]。也就是说，MLE估计出的方差比真实值小。正样本因为样本少使得估计就更不准，而且会使得分界面会比真实的分界面更偏右一些，这样使得正样本的估计概率低于真实值。

## 2. 稀有事件下LR模型的校准方法

对稀有事件应用通常的LR模型训练方法会造成估计值低于真实值，有2类校准思路：

1. 不走负采样，而直接对普通的LR模型训练出的模型做校准

2. 走负采样，然后做模型校准

当样本量大时，比如在广告定向投放中的点击率预估问题中会遇到数亿的样本量，因为第2种方案要处理的样本量比第1种方案少若干个数量级，自然它的训练效率更高。考虑到在工业界，常见大规模数据下的模型训练，在下文就围绕"负采样 + 模型校准"方案，介绍2种具体的校准方法。

### 2.1 Prior Correction

"Prior Correction"策略可以理解为基于先验分布的校准。进行负采样后经过模型训练会得到模型参数，因为采样前后正样本的比例发生变化，所以得到的模型是有偏的，而且对正样本的估计概率偏高。需要把正样本比例作为先验信息，对模型参数做校准。使用MLE估计出的模型参数中，估计出的非截距项的参数与真实参数是一致的，不用校准，但对估计出的表示截距项的参数只需按照公式3进行校准，即可得到截距项的一致性估计。

\begin{equation}{\hat {\beta}_0} - ln\left[\left(\frac {1 - \tau}{\tau}\right)\left(\frac {\overline y}{1 - \overline y}\right)\right]\end{equation}

说明：\\({\hat {\beta}}_0\\)是进行负采样后的通过LR模型训练出的模型参数。\\(\tau\\)是未进行负采样时正样本的比例，\\(\overline y\\)是进行负采样后的正样本的比例。公式3在论文[4]的4.1一节也有论述，并在论文的附录B中有公式推导。

\\(\tau\\)虽然需要先验知识来确定，但这在一些问题上比较好定，比如在广告定向投放的点击率预估问题中，就可以用统计CTR来近似替代。

"Prior Correction"策略的优点是现有的模型训练工具不用修改，还是可以使用，只不过需要加个对截距项的后处理计算。缺点是如果所建立的模型有偏差，比如少了一些应该有的特征，那这个方法的鲁棒性就不如下文将介绍的Weighting策略。该方法在Facebook的广告定向投放业务中有应用[9]，并做了一定简化，因为广告点击率在百分位甚至更低，\\(\tau\\)和\\(\overline y\\)都很小，所以\\(\frac {1 - \tau}{1 - \overline y}\\)可以近似认为等于1。加上这个近似，从而得到公式4。在Facebook其实用的正是公式4[9]：

\begin{equation}{\hat {\beta}_0} - ln\left(\frac {\overline y}{\tau}\right)\end{equation}

公式4其实与Facebook在论文[9]的6.3一节的式子是等价的。<font color='red'>强调一下，是等价，而不是近似。</font>下面做简单证明，记校准前的预测概率为\\(p\\)，校准后的预测概率为\\(q\\)，那么根据公式4有公式5成立。

\begin{equation}q = \frac 1{1 + e^{-x_i\beta + {ln\left(\overline y/\tau\right)}}} = \frac 1{1 + e^{-x_i\beta} \times \left(\overline y/\tau\right)}\end{equation}

容易知道\\(e^{-x_i\beta} = \frac {1-p}{p}\\)，所以式子5可以化简为

\begin{equation}q = \frac 1{1 + ((1 - p) / p) \times (\overline y / \tau)} = \frac p{p + (1 - p) \times (\overline y / \tau)}\end{equation}

论文里\\(w\\)表示采样率，所以等式\\(\frac 1w = \frac {\overline y}{\tau}\\)成立，代入上式即可得到Facebook在论文[9]的校准公式。

不过公式4是离线算好的校准后的截距项，就不用实时计算校准，能减少实时计算量。

### 2.2 Weighting

相比"Prior Correction"策略，Weighting策略应该更容易理解。在进行负样本采样后，模型训练前，对正样本、负样本赋以不同的样本权重。在文章[4]给出的样本权重是这样的，正样本的权重为\\(\tau / \overline y\\)，负样本的权重为\\((1 - \tau) / (1 - \overline y)\\)。因为\\(\tau\\)和\\(\overline y\\)都很小，所以\\((1 - \tau) / (1 - \overline y)\\)会近似等于1，权重的配置就可以调整为正样本的权重为1，负样本的权重为\\(\overline y / \tau\\)。这个正是Google在文章[10]所用的策略。

当建立的模型本身有偏差时，比如少了一些该有的特征，这种方法的鲁棒性要好于"Prior Correction"策略。另外，一些实验显示在应用FTRL算法的基础上，使用Weighting的策略并不好，可能的原因在于在线学习算法需要样本随机打散，但对于Weighting策略，样本权重在一定程度上阻碍了样本的随机性。

### 3 其他

论文[4]的附录B还推导给出了多类别的校准公式，这个就可以用于Softmax Regression等模型中。

## 参考文献

[1] [Logistic Regression](https://en.wikipedia.org/wiki/Logistic_regression) (来自Wikipedia)

[2] [Max Entropy](https://en.wikipedia.org/wiki/Maximum_entropy_probability_distribution) (来自Wikipedia)

[3] [Softmax Regression 或者 Multinomial Logistic Regression](https://en.wikipedia.org/wiki/Multinomial_logistic_regression) (来自Wikipedia)

[4] Gary King, Langche Zeng. Logistic Regression in Rare Events Data. Political Methodology. 2002

[5] 刘鹏, 王超. 计算广告. 2015

[6] [Consistent estimator](https://en.wikipedia.org/wiki/Consistent_estimator) (来自Wikipedia)

[7] [What is the difference between a consistent estimator and an unbiased estimator](http://stats.stackexchange.com/questions/31036/what-is-the-difference-between-a-consistent-estimator-and-an-unbiased-estimator) (来自stackexchange)

[8] [极大似然法计算出的高斯分布的方差为什么会产生偏差](https://www.zhihu.com/question/28751472) (来自知乎)

[9] Xinran He, et al. Practical Lessons from Predicting Clicks on Ads at Facebook. ADKDD2014

[10] H. Brendan McMahan, et al. Ad Click Prediction: a View from the Trenches. KDD2013

* * *

**原创文章，转载请注明：**转载自[{{ site.title }}]({{ site.production_url }})

**本文链接地址：**[{{ page.title }}]({{ site.production_url }}{{ page.url }})

* * *
