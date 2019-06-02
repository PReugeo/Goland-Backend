## Scikit-learnAPI

数据集通过其提供的api获取



### 获取数据集

#### 小数据

```python
sklearn.datasets.load_*(数据集名)
```

#### 大规模数据

```python
Sklearn.datasets.fetch_*
```



### 特征抽取

1. 字典特征提取

    ```python
    sklearn.feature_extraction.DictVectorizer
    ```

2. 文本特征提取

    ```python
    sklearn.feature_extraction.text.CountVectorizer
    sklearn.feature_extraction.text.TfidVectorizer
    ```

    针对中文需要先进行分调

3. 图像特征提取

### 特征预处理(数值型数据的无量纲化)

#### 归一化

```pyhton
sklearn.preprocessing.MinMaxScaler
```

最大值最小值非常容易受异常点影响



#### 标准化

```python
sklearn.preprocessing.StandardScalar
```

少量异常点影响不大



### 特征降维

#### 特征选择

1. 低反差特征过滤
2. 相关系数

嵌入式方法



#### 主成分分析

```
sklearn.decomposition.PCA
```

