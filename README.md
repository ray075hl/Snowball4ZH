# Snowball4ZH
适用于中文的半监督开放关系提取方法。
原文是2000年左右的一篇[Snowball: Extracting Relations from Large Plain-Text Collections](http://www.cs.columbia.edu/~gravano/Papers/2000/dl00.pdf)， 思路比较有意思，稍作改动感觉也适用于中文(实验结果在最后的段落)。

Readme可以当作是对这篇文章的简单解读。

## 1. 前言
Snowball是一种启动代价较小的关系抽取方法；利用少量的种子关系对，采用自举式（bootstrap）的方式，迭代挖掘新规则、新种子关系对。类似**滚雪球**，所以取名**雪球**可以说是相当的形象了。



## 2. 大致的原理和流程

术语对齐： 
* 种子对 = tuple/seed
* 规则 = pattern
* cos/sim = 内积/cosine相似度
* before 缩写为 bef
* between 缩写为 bet
* after 缩写为 aft

```mermaid
flowchart LR
A[人工种子对] -->|扫描语料| B(语料)
B --> C(pattern生成+评价+过滤+聚类合并)
C --> D[算法生成的种子对+评价+过滤]
D --> |如果小于max_iter| B(语料)
D --> E(挖掘出来的关系词对)
```

整个流程包含4个功能模块

* pattern generation [规则生成]
* tuple/new seed generation [新种子对的生成]
* pattern evaluation + filter [规则模板的评价与删除]
* tuple evaluation + filter [新种子对的评价与删除]



### 2.1 运行流程
在seed_positive.txt 中 定义了实体的名称和类型共15对（这是初始化启动的必要知识），这些词对需要反映出待挖掘的关系，如同义关系“痤疮;青春痘”；上下位关系“灰指甲;皮肤病”。
```
e1:DIS
e2:DIS

痤疮;青春痘
梦游;睡行症
脑卒中;中风
```
seed_negative.txt 可以为空
#### 2.1.1 使用每一个关系pair，对所有的语料进行扫描；
语料示例：
```
<SYM>精神</SYM> 科 专家 指出 ， <DIS>脑卒中</DIS> 就是 俗称 的 <DIS>中风</DIS> ， 是 老年人 <DIS>脑血管常见病</DIS> ， 会 影响 老年人 的 行动 、 语言 沟通 能力 、 思维 等

<DIS>痤疮</DIS> ， 俗称 <DIS>青春痘</DIS> 的 发病 原因
```
通过使用种子词对对语料的扫描

得到如下pattern (5元组) （before, <entity_1>, between, <entity_2>, after）

* ( "专家 指出" ， [DIS], "就是 俗称 的", [DIS], "， 是 老年人" ）
* ( ""， [DIS], "， 俗称" , [DIS], "的 发病 原因" ）

#### 2.1.2 计算每个pattern的分数

举例：

使用pattern1 扫描所有文档， 得到候选{tuple_1, tuple_2, ... ,tuple_n)；

假设1~n个关系对中有10个tuple在原始的seed_positive.txt中，则pattern1的得分为0.66（10/15）； 如果大于人为设定的阈值，则保留该规则；
对应论文中的公式**Definition 3:** 
$$cond(P)=\frac{P_{positive}}{P_{positive}+P_{negative}}$$


#### 2.1.3 计算tuple的分数

公式如下： 大致的理解就是 如果这个tuple由分数高的pattern产生的，且由多个pattern共同生成（反映在阶乘），那么这个tuple的可信度就越高。对应文中的公式**Definition 5:**

$$Conf(T) = 1-\prod_{i=0}^{|P|}(1-Conf(P_{i})*Match(Ci,Pi))$$

#### 2.1.4 合并pattern

pattern 是一个5元组

（before, <entity_1>, between, <entity_2>, after）

( "专家 指出" ， [DIS], "就是 俗称 的", [DIS], "， 是 老年人" ）
( ""， [DIS], "， 俗称" , [DIS], "的 发病 原因" ）


为了计算pattern之间的相似度， 必须将pattern进行向量话， 这里的向量化是针对一个pattern的before，between, after; 三个部分的向量， 这里可以采用tf-idf 对这个三个字符串进行向量化；

$$sim(pattern1, pattern2) = 0.2 * V1_{bef} * V2_{bef} + 0.6 * V1_{bet} * V2_{bet}+ 0.2 * V1_{aft} * V2_{aft}$$

中间的词汇权重为最大（0.6）比较重要，这是符合常识的。

sim(pattern1, pattern2) > 阈值， 那么pattern1和pattern2 会归到同一个pattern组内。


重复上述过程，直至迭代次数消耗完毕。

## 3. 实验参数
迭代轮次5次， 规则置信度0.6; 详见parameters.cfg

本次实验中，正样本种子给了15个；
```
痤疮;青春痘
梦游;睡行症
脑卒中;中风
脂肪性肝炎;脂肪肝
抑郁症;抑郁障碍
颈椎病;颈椎综合征
膝关节软骨磨损;膝关节退行性变
灰指甲;甲真菌病
皮肤癌;基底细胞癌
心肌病;TCM
心肌病;一过性心脏综合征
灰指甲;皮肤病
房颤;心律失常
小儿脑性瘫痪;小儿大脑性瘫痪
小儿大脑性瘫痪;急性鼻炎
荨麻疹;风疹块
包含同义关系（痤疮;青春痘）、上下位关系（灰指甲;皮肤病）
```
## 4. 引用
英文版本地址：https://github.com/davidsbatista/Snowball

## 5. 其它
数据输入的格式： 可以参考如下样例数据, 后处理ner的结果，采用jieba分词，再以空格符合并
```
<SYM>精神</SYM> 科 专家 指出 ， <DIS>脑卒中</DIS> 就是 俗称 的 <DIS>中风</DIS> ， 是 老年人 <DIS>脑血管常见病</DIS> ， 会 影响 老年人 的 行动 、 语言 沟通 能力 、 思维 等
<DIS>痤疮</DIS> ， 俗称 <DIS>青春痘</DIS> 的 发病 原因
```

运行：
```
python Snowball.py parameters.cfg ./corpus/sentences.txt ./corpus/seeds_positive.txt ./corpus/seeds_negative.txt 0.6 0.6
```


## 6. 实验结果(部分)
在本人的语料中（语料不能公开），结果精准率约为91%（人工抽样验证）；
instance： **挖掘出来的关系对**
sentence： **关系来源的出处**
pattern_bet： **算法生成的模板中的between部分**

```
instance: 狂犬病	狂犬病病毒感染	 score:0.09701501354609915
sentence: <DIS>狂犬病</DIS> 是 一种 有 <DIS>狂犬病病毒感染</DIS> 引起 的 <DIS>急性传染病</DIS> ， 也 是 现在 世界 上 致死率 最高 的 <DIS>传染病</DIS> 之一
pattern_bet:  是 一种 

instance: 类风湿	自身免疫性疾病	 score:0.08391098070410352
sentence: <DIS>类风湿</DIS> 是 一种 <DIS>自身免疫性疾病</DIS> ， 不 只 可 引起 <DIS>关节病</DIS> 变 ， 同时 对 组织 及 身体 其它 器官 也 可 造成 不可 程度 的 危害 ， 因此 当 患上 <DIS>类风湿</DIS> 时 ， 要 及时 治疗 ， 同时 注意 做好 相关 的 <MED>护理</MED> 工作 ， 如 在 饮食 上 ， <DIS>类风湿</DIS> 患者 要 注意 合理 选择 食物 ， 多 吃 一些 有利 食物 ， 以 帮助 缓解 症状
pattern_bet:  是 一种 

instance: 骨质疏松症	骨骼慢性疾病	 score:0.005739479400272667
sentence: <DIS>骨质疏松症</DIS> 是 最 常见 的 <DIS>骨骼慢性疾病</DIS>
pattern_bet:  ， 是 最 常见 的 

instance: 灰指甲	甲真菌病	 score:0.01
sentence: <DIS>灰指甲</DIS> ， 也 称 <DIS>甲真菌病</DIS> ， 是 <BOD>指甲</BOD> 最常患 的 疾病 ， 占 <DIS>指甲病</DIS> 的 一半 以上
pattern_bet:  ， 也 称 

instance: 月经失调	月经不调	 score:0.01
sentence: <DIS>月经失调</DIS> 也 称 <DIS>月经不调</DIS> ， 是 妇科 常见疾病 ， 表现 为 月经周期 或 <SYM>出血</SYM> 量 的 异常 ， 可伴 月经 前 、 经期 时 的 <SYM>腹痛</SYM> 及 全身 症状
pattern_bet:  ， 也 称 

instance: 狂犬病	恐水病	 score:0.10160113364224838
sentence: <DIS>狂犬病</DIS> 的 典型 临床表现 为 <DIS>恐水症</DIS> ， 故 <DIS>狂犬病</DIS> 又称 <DIS>恐水病</DIS>
pattern_bet:  ， 又称 ,  又称 

instance: 房颤	心律失常	 score:0.1679828582189843
sentence: <DIS>心房颤动</DIS> ， 简称 <DIS>房颤</DIS> ， 是 临床 上 常见 的 <DIS>心律失常</DIS>
pattern_bet:  ， 是 临床 上 常见 的 

instance: 排卵障碍	不排卵	 score:0.19877128633481048
sentence: <DIS>排卵障碍</DIS> ， 又 称为 <DIS>不排卵</DIS> ， 是 女性 不孕症 的 主要 原因 之一 ， 约 占 25% ～ 30%
pattern_bet:  ， 又 可以 称为 ,  ， 又 被 称为 
```
