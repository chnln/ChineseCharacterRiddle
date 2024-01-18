# Semantic Computing and Knowledge Retrieval HW3

SCKR 2022 Spring HW3
using Sentence-Transformers + data augmentation to tackle Chinese Character Riddles

## 1 数据集介绍

本次作业需要模型对于给定谜面给出对应的谜底，允许给出最多top-5的结果。如“借才生财”的谜底是“贝”。

剔除数据集中明显不合理的成分后（训练集中存在谜底“<”、“(”、“_”、“♂”，测试集词典存在“`”、“《”，均予以剔除），数据分布如下（测试集未给出谜底）：

|                              | train | valid | test |
| :--------------------------: | :---: | :---: | :--: |
|       riddles（谜面）        | 16626 | 5480  | 5413 |
| charactets（谜底）（tokens） | 16626 | 5480  |  /   |
| characters（谜底）（types）  | 4367  | 1457  | 1456 |

按token计算的汉字（character）数和谜面的数量一样，由于存在不同谜面打同一个字的情况，按type计算的汉字数与按token相比少了很多。数据集在设计时，训练集、验证集和测试集的谜底互不重叠，这给模型训练带来了挑战。

## 2 实验环境

### 2.1 环境依赖

本实验采用的主要库、工具以及它们的版本为：

| package               | version |
| :-------------------- | :------ |
| sentence_transformers | 2.2.0   |
| transformers          | 4.19.2  |
| torch                 | 1.10.0  |
| cuda_toolkits         | 11.3    |

本实验在[featurize](featurize.cn)平台提供租用的单张RTX 3090显卡上运行。

### 2.2 文件说明

| Directory | Description                                                  |
| --------- | ------------------------------------------------------------ |
| /         | 包括预处理文件`preprocess.ipynb`，模型文件`roberta_st.ipynb`，函数文件`function.py`，验证集上表现最好的模型在测试集上的预测结果`test_predictions.txt` |
| data/     | 包括原有的数据集文件以及用于sentence transformers训练的、转换后的文件，包括数据增强后的文件 |
| resource/ | 利用的外部资源，包括zi-dataset、新华字典的单字部分，以及整理后的汉字信息`character_information.json` |
| output/   | 子文件名记录了训练时的超参数，子文件夹下的内容包括模型的预测结果`{split_name}_predictions.txt`和评测结果`{split_name}_metrics.txt` |

## 3 实验过程

### 3.1 评测指标

本实验采用MRR作为评测指标。具体来说，对于系统返回的最可信的top k个候选答案，即候选谜底，如果真正的答案在第1个，得分为1，如果真正的答案在第2个，那么得分为1/2，以此类推，根据可以纳入计分范围的答案的最大序号，有MRR Top-1、MRR Top-3和MRR Top-5三个指标。

### 3.2 实验思路

该任务初看是一个生成任务，需要根据给定的谜面生成相应的谜底（汉字），这也符合人类猜谜语的认知过程。但生成模型难度较高，谜面、谜底也较短，机器比较难学习到生成规则。

考虑到测试集词表已经给定，该任务可以转化成两个任务，（1）分类任务或（2）检索任务。

考虑分类任务。由于谜底（汉字）是封闭集合，可以将所有备选的谜底看成一个个标签，通过训练一个分类模型，对于输入的谜面，将其“分”给对应的谜底。但对于这个数据集，分类任务的难点有二：（1）分类标签过多，数据集总共包含7k多个谜底，抽取出的特征会很稀疏，（2）训练集、验证集和测试集的谜底无交叠，会导致在测试集上分类时，可能的分类结果在之前全部没有遇到过，可想而知训练效果会很差。

考虑检索任务。同样地，由于谜底是一个封闭集合，我们可以将输入的谜面看成检索请求query，要从所有备选的谜底中选择一个答案与其匹配。检索任务的解决思路更加合理，同时还需要考虑下面一些细节问题：

1. 由于MRR要求输入前Top-k个候选答案，在训练时，query和answer之间标签不能仅仅是匹配/不匹配的二分类标签，需要是连续变量，方便对于固定的query，给出answer内部的排序。
2. 训练集和验证集给出的谜面和谜底都是对应的，也就是都是正样本，缺少负样本。

### 3.3 算法介绍

本实验采用文本检索和QA领域的经典框思路，使用预训练模型对query（谜面）和answer（谜底）分别编码，并提供表示query和answer相关性的标签，通过余弦相似度损失函数的反馈不断调整预训练模型的表示，最终通过对备选谜底与谜面相似度进行排序，选出top-k个候选答案。思路可以概括为以下流程图。

![SBERT Siamese Network Architecture](https://www.sbert.net/_images/SBERT_Siamese_Network.png)

在模型方面，本实验选用[Sentence-Transformers](www.sbert.net)开放的双塔Bi-Encoder框架，使用HuggingFace和HFL提供的预训练好的[roberta模型](https://huggingface.co/hfl/chinese-roberta-wwm-ext)对文本进行编码。

考虑到谜语都是单个汉字，信息量很少，需要进行数据增强。字谜往往和汉字字形与释义，以及其部首、组成部分等有关，本实验利用网上公开的汉字数据集[zi-dataset](https://github.com/secsilm/zi-dataset)、新华字典数据[chinese-xinhua](https://github.com/pwxcoo/chinese-xinhua)和拼音库[pypinyin](https://github.com/mozillazg/python-pinyin)获取汉字的笔画数、拼音、部首、部首的笔画数、部首的拼音、组件、释义等信息，将它们拼接成对于单个汉字的“描述”，以此代替汉字本身输入预训练模型获得文本表示。以“圹”字为例，该汉字的相关信息如下。

![image-20220605233111294](/Users/nascent/Library/Application Support/typora-user-images/image-20220605233111294.png)

对于训练集、验证集都是正样本的问题，本实验对训练集中的每一对谜面-谜底进行数据增强操作，从训练集中先后随机选择一个谜底不同、一个谜面不同的谜面-谜底对，分别用后两者的谜底、谜面与前者交换，得到新的两个谜面-谜底对。显然，得到的这两个谜面-谜底对是负样本。只对训练集进行增强，增强后训练集有49878条数据。在训练集和验证集中，对于正样本，将label记为1，表示极为相关；对于负样本，将label记为0，表示不相关。

得到增强后的数据集后，即开始训练，相关超参数如下：

|  Hyper parameer  |                        Value                         |
| :--------------: | :--------------------------------------------------: |
|    batch size    |                          16                          |
|      epochs      |                         3、5                         |
| evaluation steps |            4988（size of train set / 10）            |
|   warmup_steps   | 1559（size of train set / batch size * epochs / 10） |

## 4 实验结果

在验证集上，MRR的最好表现（epochs为3）如下表所示，同时给出该模型在训练集和测试集上的表现：

|       | MRR Top-1 | MRR Top-3 | MRR Top-5 |
| ----- | --------- | --------- | --------- |
| train | 0.5314    | 0.6999    | 0.7481    |
| valid | 0.0113    | 0.0169    | 0.0186    |
| test  | 0.086     | 0.115     | 0.125     |

MRR指标和测试集上的预测结果记录在`output/`文件夹，根目录文件夹下的`test_predictions.txt`为验证集上训练效果最好的模型在测试集上的预测结果。

## 5 实验结论

1. 在猜字谜任务上，本实验的模型表现很差，这可能和猜字谜任务对参与者的认知能力要求高有关，猜谜者不仅需要对字的结构、笔画有清楚的了解，还需要掌握字的释义、理解谜面的含义（有时候类似一个“脑筋急转弯”）。总体来说，这是一个难度很大的任务，机器模型表现差在意料之中。

2. 本文的主要创新点在于：

   1. 将“猜字谜”这样一个生成任务转化成了一个检索/QA任务，利用了备选谜底是封闭集这一条件。用谜面和谜底相似度表示两者的对应关系的方法，也方便排序。
   2. 用替换谜面、谜底的方法对原数据集进行了数据增强，增加了原有数据集两倍规模的负样本。
   3. 用汉字的字形、读音、释义等信息拼接起来表示原有的汉字，挖掘了单个汉字所能承载的信息。

3. 随机抽取10个模型猜对的谜语：

   | riddle             | character |
   | ------------------ | --------- |
   | 晴空一色光闪闪     | 晃        |
   | 丝丝垂柳隔山来     | 幽        |
   | 冥冥之中有天意     | 日        |
   | 眼前一片土         | 睉        |
   | 曲中曲径通幽篁     | 麴        |
   | 岁暮不见有人来     | 仙        |
   | 来客进门须脱帽     | 阁        |
   | 一抹夕照，金宛满盈 | 鋺        |
   | 隐隐云山眼底收     | 眩        |
   | 月照四方心相联     | 腮        |

   发现他们基本都是和字形相关，比如对于“月照四方心相联”，将“月”、“四”、“心”拼接就可以得到“腮”，“眼前一片土”是“目”字右边有“土”字；而和读音、释义等关系较小。这一方面启示我们可以更多地从字形上下功夫，比如引入汉字的结构信息（是左右结构、上下结构哪类结构等），设计部首、偏旁的组合提升规则等；另一方面说明拼音、释义等方面的信息价值不够高。在拼接汉字信息时发现，新华字典的释义往往过长，这可能引入了大量不相关的信息，增加了噪声，对训练有损害。下一步应该优化释义的表示方式。
