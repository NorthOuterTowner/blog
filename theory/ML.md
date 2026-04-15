# 介绍
近期在做 Machine Learning 相关的内容，发现对于很多ML的模型只是知道名字，但是对于其中的原理和应用很不了解，因此写一篇blog来简单记录当前使用的几种集成学习的原理以及对应的使用方法。

## XGBoost

### 总体训练过程
XGBoost采用前向分步算法，在每一棵树中始终以修复之前所有树累加之后的残差作为训练的目的。
同时，通过将树进行参数化，在正则化项中引入了 $\lambda$ 和 $\gamma$,来控制叶子节点的数量，防止树长得太深，同时控制叶子权重，避免某一棵树权重过大。
对于每一个待分裂的节点，尝试所有可能的特征和切分点计算分裂后的增益。
另外，XGBoost这种方法也有学习率的概念，通过学习率，来确定每棵树准备学习的残差的量，而不是让一棵树尝试去学习所有的结构，确保每一棵树只学习一部分的结构化信息，避免陷入局部最优解。

### 使用XGBoost进行训练
我仍然认为明确一种算法的使用场景，即在什么情况下使用，解决什么样的问题，大致以什么样的方式来使用，因此将着重讨论模型如何训练和如何预测的问题。
```python
import xgboost as xgb
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

# 1. 加载数据
data = load_breast_cancer()
X, y = data.data, data.target
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# 2. 初始化模型 (Scikit-Learn 风格)
model = xgb.XGBClassifier(
    n_estimators=100,       # 树的棵数
    learning_rate=0.1,      # 学习率 (eta)
    max_depth=6,            # 树的最大深度
    subsample=0.8,          # 训练每棵树时使用的样本比例（防止过拟合）
    colsample_bytree=0.8,   # 训练每棵树时使用的特征比例
    random_state=42,
    use_label_encoder=False,
    eval_metric='logloss'   # 损失函数
)

# 3. 训练模型
model.fit(X_train, y_train)

# 4. 预测
y_pred = model.predict(X_test)
print(f"准确率: {accuracy_score(y_test, y_pred):.4f}")
```

### 使用Pandas库
在数据分析的过程中，很多情况下使用`Pandas`库进行数据的读取。如读取`csv`文件：
```python
df = pd.read_csv('medical_data.csv')
X = df.drop('label', axix=1)
y = df['label']
```
读取后得到`DataFrame`对象，这是一个带有一些元数据的二维矩阵，我个人的理解是将整个表的内容，连同他的index和每个列的名称，全部读入了这个`DataFrame`中。
通过Column的名称，可以直接读取到一个“列”，如上述的`y=df['label']`就是这种做法，通过这种方式读取到的数据，其类型是`Series`。

在`Series`中具有`dtype`属性，返回这一类的值，此处有一个需要进行注意的点，即在较老的`pandas`版本中，要么是数值型`int64`（也可能是`float64`），要么是非数值型，任何非数值型都会显示为`object`，但在较新的版本中，如果内部均为`str`类型，则会显示为`str`，因此通过是否`object`的方法难以直接判断。如果你也遇到了类似的问题，可以尝试使用`Series.apply()`方法：
```python
series = df['label']

def transform(val):
    if isinstance(val, (str, bytes)):
        return str(val)
    else:
        return ''

s_new = series.apply(transform)
```
通过这样的方法对`Series`的内容逐个调用自定义的转换函数，从而确保不会出现问题。

## LightGBM

## CatBoost

## TabNet