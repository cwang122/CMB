# On Binary Data Analysis/对二元数据的分析
> William Wang --- Dept. Combined Interdisciplines and Engineering

## Preface

从直观上看，对于这份社区数据集，可以总结出如下特点：
 - 它拥有较多的变量，并且所有变量都具有单一的特征：即所有关于这些变量的数据都只由0或者1构成。也就是说，我们面对的是一个二元数据集。
 - 这个数据集由两个部分组成，一部分是对于交易类别的记录，另一部分是对于app栏目浏览类别的记录，对于出现过以上行为的用户，在对应的类别变量里记为1，没有则记为0.
 - 这两部分数据，既互相独立，又互相关联，所以在处理时应铭记这一点。
 
## 对于聚类分析方法的选择与考虑

传统的聚类算法多种多样，而目前的主流是计算向量距离为核心的聚类算法，如K-means，K-modes以及Hierarchial clustering等等。对于这种二元数据集，在实际应用的过程中，主流算法暴露出了原理上的缺陷。尽管最后可以得出较符合聚类思路的结果，但是我们依旧有理由质疑其精度与普适性。

以简单的K-means聚类为例，当我们把整个一行数据写作向量，并且计算其均值，比较其距离的时候，由于0和1的存在，这种均值并没有多大意义。而聚类完成的向量依旧是一个二元向量（即永远为0和1组成）。这种距离的计算对于二元数据的向量难以产生效果。

基于以上的原因，有两种思路产生了，第一种是先用一种方法变动数据，再进行K-means等算法分析，第二种是直接换用基于不同原理的算法。我将在这份报告中讨论这两种思路。

第一种思路的执行方案是，先通过MCA分析，基本处理数据，然后通过k-means聚类。

第二种思路的执行方案是，放弃以距离计算作为原理的聚类方法，改为以条件概率作为计算依据，依靠这种原理的聚类算法的代表，就是LCA聚类（Latent Class Analysis）和LPA聚类（Latent Profile Analysis）。在这里我们采用LCA聚类方法。

LCA分析，即潜在类别分析，是通过条件概率公式，反推出具有相似条件概率的聚类，相比于距离计算更加适合变量较多的二元数据集。潜在变量是离散的，通过条件概率计算出的结果指示出变量对于特定值的可能性。

首先，我们会预先设定LCA的聚类数量，然后算法会根据每个用户表现出这些聚类行为的概率大小（即条件概率），将用户归属到这些聚类当中，这样，我们不仅能够进行普通的聚类，更能分析每个单独用户的行为，以及这名用户属于那一类


## 使用MCA再进行K-means的聚类分析方法

## 使用LCA模型的聚类分析方法

按照前文所叙的思路，我们现在将要实现这个模型。

python中没有直接进行LCA模型分析的库，一方面是LCA模型比较小众，另一方面是因为对于LCA模型涉及到的条件概率计算来说，python并非最好的选择。这个时候我们需要用到另一种数据分析软件 - R/RStudio以及R Package DepMixS4 (v. 1.4.0)

```
depmixS4: Dependent Mixture Models - Hidden Markov Models of GLMs and Other Distributions in S4
URL: https://cran.r-project.org/web/packages/depmixS4/index.html
官方文档：https://www.rdocumentation.org/packages/depmixS4/versions/1.4-0

```

R Package DepMixS4是设计用于潜在马尔可夫模型类的建模处理的，因此LCA模型用它来做非常合适。

* 此处应该注意，当没有连接VPN时，这个package无法安装。我们有理由相信，大量像DepMixS4这样不错的package因为这个原因无法出现在大众的视野当中，倘若这些package善加利用，R的用处应该会更大。

### Get Started!

首先，安装并Library DepMixS4 Package：

```{r}
install.packages("depmixS4")
library("depmixS4")
```

对于这个LCA模型，我们需要预设潜在的class数量，也就是说，我们需要设定有多少种类型的用户，建模会围绕这个数量展开。这一步代码如下：

```
#model 1 with class = 6 / 设定Model 1拥有6种潜在客户类型

lca <- read.table("community_related_lca.csv", sep=",") #读取csv数据集

names(lca) <- c( "cash_n", "zf_n", "cf_n", "zz_n" ,"cc_n", "jf_n", "dk_n","sz_pv","zz_pv","zh_pv", "chc_pv",  "lc_pv","jj_pv",xd_pv","sh_pv","hd_pv") #将全部变量名称写入向量之中。

summary(lca) # 输出对数据集的大致描述。
```

其中，summary(lca)的输出结果是这样：

![r_output_1](https://github.com/cwang122/CMB/blob/master/r_output_screenshot.png)

图片中的xxx_n和xxx_pv即是指上面代码中names()里的16中变量，意指用户表现出的16种行为。0为没有表现，1为曾经表现过。不难看出，大多数变量的0都远远超过1的数量，这种数据集即可被形容为“sparse”（稀疏的）数据集。

> "Sparse" in binary dataset, means there are few 1s, with an ocean of 0s.  --- Anonymous on ResearchGate
> 二元数据集里的所谓“稀疏”，即意味着少部分的1，和汪洋大海一般的0. --- ResearchGate上某位老哥

### 关于LCA聚类的快速Q&A~

Q: 你们并不知道这些人中有几种类型的用户，和具体的用户偏好以及背景，强行设定聚类数量的话，如何解释聚类结果并且保证其正确呢？

A：如同大多数问卷调查一样，从心理学到社会学，都并非事先知道用户的属性，而是通过归纳的方法得出。严格来说，聚类的终极形态，是把每一个用户归为一类，这样是最为精确，最为自然的。聚类正是为了有目的地模糊个体之间的差异而做出的一种妥协，聚类是否有效，是否能够解释，取决于算法的输出结果是否在各变量间拥有足够大的差异，以及在数据所拥有的背景情境下能否说得通。

Q: 聚类数量设定多少算合适呢？

A: 和上面的回答相似，只要在具体环境下，这个聚类的行为概率构成可以被解释的通，并且每个用户对于不同行为聚类发生概率的差异足够明显就可以了。这个设定完全处于实际情境的需要。





