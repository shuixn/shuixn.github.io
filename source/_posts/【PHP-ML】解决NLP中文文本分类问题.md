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

<!-- more -->

本文的实践例子已经放在Github：[php-text-classification](https://github.com/funsoul/php-text-classification)

## 数据集

本文采用[头条新闻数据集](https://github.com/fatecbf/toutiao-text-classfication-dataset)

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

- SVM（依赖libsvm库）
- KNN
- NavieBayes
- MLPClassifier

## 数据预处理

php-ml对分类模型保持高度一致的接口，定义了统一的输入和输出。下面的代码贯穿全文，只需要选取合适的模型，把样本数据集``Samples``和类别对象``Labels``灌入训练API，即可进行训练。

```php
$samples = [[5, 1, 1], [1, 5, 1], [1, 1, 5]];
$labels = ['a', 'b', 'c'];

$classifier = new ClassificationModel();
$classifier->train($samples, $labels);
```

在这里，ClassificationModel可以是SVM，也可以KNN或者其他分类模型。区别在于，各模型存在算法、核函数或超参差异。当进行深度拟合数据、提高模型分类效果时，可进行调整。然而，php-ml没有交叉验证或者网格搜索等方法，需要自己设计程序进行调参。

回到主题，我们需要把不同类别的中文文本数据进行训练，中文文本的``形式化处理``是关键。

先看原始数据形式，截取第一条数据出来

```vim
6551700932705387022_!_101_!_news_culture_!_京城最值得你来场文化之旅的博物馆_!_保利集团,马未都,中国科学技术馆,博物馆,新中国
```

数据以``_!_``分隔，对本文来说，需要获取类别ID、类别名称和句子，如下：

- category_id: 101
- category_name: news_culture
- text: 京城最值得你来场文化之旅的博物馆

读取文件，进行分词和停用词过滤后，得到如下的数据结构

```php
[
  [
    category_id: 101,
    category_name: news_culture,
    text: 京城最值得你来场文化之旅的博物馆
  ]
  // 此处省略382687行
]
```

## 特征提取

### 分词

分词方式有多种：

1. 直接调用php jieba分词库[fukuball/jieba-php](https://github.com/fukuball/jieba-php)
2. 直接调用php jieba分词扩展[jonnywang/phpjieba](https://github.com/jonnywang/phpjieba)
3. 使用swoole+jieba分词，提供一个http服务。参考之前写的这篇文章{% post_link Swoole加速结巴分词 %}
4. 使用python+aiohttp+jieba

### 过滤

1. 停用词过滤，可以使用[goto456/stopwords](https://github.com/goto456/stopwords)
2. 对于英文文本，单个字符会被过滤。对于中文同样适用，单个词没有太大意义

### 语料库（词袋）

进行分词和过滤后，将获得特征。

```
['京城', '值得',  '来场' , '文化', '之旅', '博物馆']
```

将特征以空格分隔，合并成句子

```
京城 值得 来场 文化 之旅 博物馆
```

使用``WhitespaceTokenizer``进行文本样本集的向量化

```php
$vectorizer = new TokenCountVectorizer(new WhitespaceTokenizer());
$vectorizer->fit($trainX);
$vectorizer->transform($trainX);
```

词袋

```php
$vectorizer->getVocabulary();
// ['之旅', '京城', '值得', '博物馆', '文化', '来场']
```

语料库

```php
[[1 1 1 1 1 1]]
```

### TF-IDF

>tf-idf是一种统计方法，用以评估一字词对于一个文件集或一个语料库中的其中一份文件的重要程度。字词的重要性随着它在文件中出现的次数成正比增加，但同时会随着它在语料库中出现的频率成反比下降。

```php
$transformer = new TfIdfTransformer($trainX);
$transformer->transform($trainX);
```

在这里，由于只有一句话，且这句话里面所有的字词都只有一个，所以权重是一样的。

```php
[[0.40824829 0.40824829 0.40824829 0.40824829 0.40824829 0.40824829]]
```

## 训练

进行特征处理后的``特征集``和``类别对象``传入模型的构造方法，即可进行训练。注意，这里可以对比``未使用tfidf``和``使用tfidf``前后的效果。

这里使用朴素贝叶斯举例：

```php
$model = new NaiveBayes($trainX, $trainY),
$classifier = $model->train();
```

## 预测

准备新的样本文本，并进行分词、过滤和特征提取后，传入即可

```php
$classifier->predict($testSample);
```

## 持久化

如果想要保存训练结果，避免多次训练，可以将模型持久化到本地。需要使用时，将模型重新导入内存即可使用。

```php
// 模型导出
$filepath = '/path/to/store/the/model';
$modelManager = new ModelManager();
$modelManager->saveToFile($classifier, $filepath);

// 模型导入
$restoredClassifier = $modelManager->restoreFromFile($filepath);
$restoredClassifier->predict([3, 2]);
```

## 评估指标（Metric）

仓库代码中，我将样本通过``StratifiedRandomSplit``划分为``训练集``和``测试集``，用于评估模型效果。

```php
$split = new StratifiedRandomSplit($dataset, 0.2);
```

对测试集进行预测

```php
$predictY = [];
foreach ($testX as $test) {
    $testSampleText = [$test];

    $vectorizer->transform($testSampleText);

    $predictY[] = current($classifier->predict($testSampleText));
}
```

### Score

通过``Accuracy``得到预测结果的正确率

```php
Accuracy::score($testY, $predictY)
```

### Confusion Matrix

通过``ConfusionMatrix``得到预测结果的错误情况

```php
$text = new Text($textFile);
$categoryIds = $text->getCategoryIds();
ConfusionMatrix::compute($testY, $predictY, $categoryIds);
```

### Classification Report

通过``ClassificationReport``得到整体分类报告（score、f1、recall）

```php
$report = new ClassificationReport($testY, $predictY);
$report->getAverage();
```

## Pipeline

还可以使用``Pipeline``来管线化工作流，有两个好处：

1. 代码少很多，阅读更清晰
2. 内存占用更低

```php
$transformers = [
    new TokenCountVectorizer(new WhitespaceTokenizer()),
    new TfIdfTransformer()
];

$pipeline = new Pipeline($transformers, new NaiveBayes());
$pipeline->train($trainX, $trainY);
$predictY = $pipeline->predict($testX);
Accuracy::score($testY, $predictY);
```