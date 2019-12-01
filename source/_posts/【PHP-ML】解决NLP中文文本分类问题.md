---
title: 【PHP-ML】解决NLP中文文本分类问题
date: 2019-11-30
categories:
  - 技术
tags: 
  - PHP
  - PHP-ML
  - 机器学习
  - NLP
  - 文本分类
---

## php-ml

[php-ml](https://php-ml.readthedocs.io/en/latest/)是波兰开发者[Arkadiusz Kondas](https://arkadiuszkondas.com/)的作品，前段时间刚翻译了他关于PHP在机器学习领域的看法「3 Reasons Why PHP is Not Yet Perfect for Machine Learning」。php-ml的出现丰富了PHP生态，让PHP开发者也能写机器学习应用，这篇文章讲一讲文本分类问题在php-ml中是怎么解决的。

本文的实践例子已经放在Github：[php-text-classification](https://github.com/funsoul/php-text-classification)

## 数据集

本文采用[中文文本分类数据集](https://github.com/fatecbf/toutiao-text-classfication-dataset)

- 数据规模：共382688条，分布于15个分类中。
- 采集时间：2018年05月

分类code与名称：


```vim
100 民生 故事 news_story
101 文化 文化 news_culture
102 娱乐 娱乐 news_entertainment
103 体育 体育 news_sports
104 财经 财经 news_finance
106 房产 房产 news_house
107 汽车 汽车 news_car
108 教育 教育 news_edu 
109 科技 科技 news_tech
110 军事 军事 news_military
112 旅游 旅游 news_travel
113 国际 国际 news_world
114 证券 股票 stock
115 农业 三农 news_agriculture
116 电竞 游戏 news_game
```


## Classification

php-ml有多种文本分类模型

- SVM
- KNN
- NavieBayes
- MLPClassifier

