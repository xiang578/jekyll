---
layout: post
title: 决策树
tags:  [machine learning, decision tree]
status: publish
type: post
published: true
---

之前听过决策树的名字，以为只是暴力判断。通过这一章的学习，了解到决策树是从数据中提取规则，然后对数据进行分类。所以，决策树在构建中理解了数据中所蕴含的知识信息。

## 3.1 决策树的构建

本文构建的决策树使用 ID3 算法。
决策树每一次决策都遵循一个原则：将无序的数据变得更加有序。所以，使用信息论中的熵（entropy）来度量信息。简单来说，系统越有序，熵越低。在数据划分的过程中产生的变化称为信息增益。

根据公式 ${H=-\sum_{i=1}^{n}{p(x_i)}log_2 p(x_i)}$ 计算信息期望值，其中 ${p(x_i)}$ 是选择某一个分类的概率。`dataset` 是一个 ${n*m}$ 维向量，代表输入的数据。

```python
def calc_shannon_ent(dataset):
    numentries = len(dataset)
    labelcounts = {}
    for featvec in dataset:
        currentlabel = featvec[-1]
        if currentlabel not in labelcounts.keys():
            labelcounts[currentlabel] = 0
        labelcounts[currentlabel] += 1  # 统计标签的种类以及数量
    shannonent = 0.0
    for key in labelcounts:
        prob = float(labelcounts[key])/numentries
        shannonent -= prob * log(prob, 2)  # 计算熵
    return shannonent
```

划分数据集：构建决策树过程中，删除输入数据的某一列值时使用。

```python
def split_dataset(dataset, axis, value):
    retdataset = []  # 划分之后的子集
    for data in dataset:
        if data[axis] == value:
            reducefeatvec = data[:axis]  # 去除 axis 相关的特征
            reducefeatvec.extend(data[axis+1:])
            retdataset.append(reducefeatvec)
    return retdataset
```

决策过程：枚举每一种划分方案，并计算之后的信息增益，选择信息熵最小（越有序）的一种方案，并返回这个特征名。
信息增益是熵减少的或者是数据无序度的减少。

```python
def choose_best_feature_to_split(dataset):
    numfeatures = len(dataset[0]) - 1  # 每一条数据的特征数，最后一个为标签
    baseentropy = calc_shannon_ent(dataset)  # 计算初始数据中的熵
    bestinfogain = 0.0
    bestfeature = -1
    for idx in range(numfeatures):
        featlist = [data[idx] for data in dataset]  # 列表推导，计算出某个特征的全部值范围，并在下一步去重
        uniquevals = set(featlist)
        newentropy = 0.0
        for value in uniquevals:
            subdataset = split_dataset(dataset, idx, value)
            prob = len(subdataset)/float(len(dataset))
            newentropy += prob * calc_shannon_ent(subdataset)
        infogain = baseentropy - newentropy
        # print baseentropy, newentropy
        if (infogain > bestinfogain):
            bestinfogain = infogain
            bestfeature = idx
    return bestfeature
```

构建决策树：利用递归调用构建，直到程序遍历完数据集的所有属性，或者每个分支下面的所有实例都具有相同的分类。

当所有属性都被遍历，但是某一个分支下面的数据标签不完全一样，采用多数表决的方法决定该叶子节点的分类。

```python
def majoritycnt(classlist):
    classcount = {}
    for vote in classlist:
        if vote not in classcount.keys():
            classcount[vote] = 0
        classcount[vote] += 1
    sortedclasscount = sorted(classcount.iteritems(), key=operator.itemgetter(1), reverse=True)
    return sortedclasscount[0][0]
```

递归调用，利用 dict 来存储树

```python
def create_tree(dataset, labels):
    classlist = [example[-1] for example in dataset]  # 统计数据集中的标签信息
    if classlist.count(classlist[0]) == len(dataset):  # 结束递归条件：数据集中的标签全部相同，或者数据集中只剩下一个数据
        return classlist[0]
    if len(dataset[0]) == 1:
        return majoritycnt(classlist)
    bestfeature = choose_best_feature_to_split(dataset)  # 选择分类的标签
    bestfeaturelabel = labels[bestfeature]
    mytree = {bestfeaturelabel: {}}  # 利用mytree保存结果
    del(labels[bestfeature])
    featvalues = [example[bestfeature] for example in dataset]
    unqiuevals = set(featvalues)
    for value in unqiuevals:  # 利用特征的取值范围划分本层
        sublabels = labels[:]
        mytree[bestfeaturelabel][value] = create_tree(split_dataset(dataset, bestfeature, value), sublabels)
    return mytree
```

## 3.2 L用 Matplotlib 画图

有点复杂，自我感觉和后面的关系不是很大，所以只是写了一下简单部分的代码。这里面的有一个技巧是利用 `type()` 函数判断对象是不是 dict，以此来确定是否是决策树的底层。

```python
# -*- coding:utf-8 -*-
# author: xiang578
# email: i@xiang578
# blog: www.xiang578.com


import matplotlib.pyplot as plt


# 定义画图相关的属性
decisonnode = dict(boxstyle="sawtooth", fc='0.8')
leafnode = dict(boxstyle='round4', fc='0.8')
arrow_args = dict(arrowstyle='<-')


def plot_node(nodetext, centerpt, parentpt, nodetype):
    create_plot.ax1.annotate(nodetext, xy=parentpt, xycoords='axes fraction', xytext=centerpt,
                             textcoords='axes fraction',
                             va="center", ha="center", bbox=nodetype, arrowprops=arrow_args)


def create_plot():
    fig = plt.figure(1, facecolor='white')
    fig.clf()
    create_plot.ax1 = plt.subplot(111, frameon=False)  # 定义一个函数的属性，属于全局变量
    plot_node('decisonnode', (0.5, 0.1), (0.1, 0.5), decisonnode)
    plot_node('leafnode', (0.8, 0.1), (0.3, 0.8), leafnode)
    plt.show()


def test_create_plot():
    create_plot()


def retrieve_tree(i):
    istoftrees = [{'no surfacing': {0: 'no', 1: {'flippers': {0: 'no', 1: 'yes'}}}},
                  {'no surfacing': {0: 'no', 1: {'flippers': {0: {'head': {0: 'no', 1: 'yes'}}, 1: 'no'}}}}
                  ]
    return istoftrees[i]


def get_num_leafs(mytree):
    numleafs = 0
    firststr = mytree.keys()[0]
    seconddict = mytree[firststr]
    for key in seconddict.keys():
        if type(seconddict[key]).__name__ == 'dict':
            numleafs += get_num_leafs(seconddict[key])
        else:
            numleafs += 1
    return numleafs


def get_tree_depth(mytree):
    maxdepth = 0
    firststr = mytree.keys()[0]
    seconddict = mytree[firststr]
    for key in seconddict.keys():
        if type(seconddict[key]).__name__ == 'dict':
            thisdepth = 1 + get_tree_depth(seconddict[key])
        else:
            thisdepth = 1
        if thisdepth > maxdepth:
            maxdepth = thisdepth
    return maxdepth


def test_get():
    my_tree = retrieve_tree(0)
    print my_tree
    print get_tree_depth(my_tree)
    print get_num_leafs(my_tree)


if __name__ == '__main__':
    # test_create_plot()
    test_get()
```

## 3.3 测试和存储分类器

利用决策树分类，沿着构造好的决策树一直往下递归，直到遇到的节点数据类型不是 dict 。

```python
def classify(inputtree, featlabels, testvec):
    firststr = inputtree.keys()[0]
    seconddict = inputtree[firststr]
    # print firststr, featlabels
    idx = featlabels.index(firststr)
    key = testvec[idx]
    if type(seconddict[key]).__name__ == 'dict':
        label = classify(seconddict[key], featlabels, testvec)
    else:
        label = seconddict[key]
    return label
```

利用 Python 模块中的 pickle 序列化对象，实现决策树的可持久化。

```python
def store_tree(inputtree, filename):
    import pickle
    fw = open(filename, 'w')
    pickle.dump(inputtree,fw)
    fw.close()


def grab_tree(filename):
    import pickle
    fr = open(filename)
    return pickle.load(fr)


def t_pickle():
    mydat, labels = creat_dataset()
    mytree = create_tree(mydat, labels)
    store_tree(mytree, 'clasaifyierStorage.txt')
    print grab_tree('clasaifyierStorage.txt')
```

## 3.4 使用决策树预测隐形眼镜类型

跟上面提到的鱼分类类似，关键是从 txt 中读取数据，然后调用分类的算法。书中也没有提供其他过多的测试数据。

```python
def t_Lenses():
    fr = open('lenses.txt')
    lenses = []
    for inst in fr.readlines():
        line = inst.strip()
        tmp = inst.strip().split('\t')
        lenses.append(tmp)
    # lenses = [inst.strip('\n').split('\t') for inst in fr.readline()]
    # print lenses
    lenseslabels = ['age', 'prescript', 'astigmatic', 'tearrate']
    lensestree = create_tree(lenses, lenseslabels)
    print lensestree
```

## 参考
[MachineLearning/3.决策树.md at master · apachecn/MachineLearning](https://github.com/apachecn/MachineLearning/blob/master/docs/3.%E5%86%B3%E7%AD%96%E6%A0%91.md)
