# mastering-py-metrics

## 介绍
利用 Python 实现《精通计量：因果之道》（_Mastering 'Metrics_ by Joshua Angrist）的实证因果推断案例，基于业界常用的`statsmodels` 和 `DoWhy`因果推断包实现。

## 开始之前
### 环境配置
**注意，DoWhy 似乎目前不支持M系列 mac 芯片**
* macOS / Linux 环境下
```bash
conda create --name mastering-py-metrics python=3.11
conda activate mastering-py-metrics
which pip # make sure environmnet is activated
pip install -r requirements.txt
```

### 下载数据集
* macOS / Linux 环境下
```bash
# Chapter 1
cd ch1*/data
curl -o Data.zip https://www.masteringmetrics.com/wp-content/uploads/2021/03/Data.zip
unzip -j Data.zip -d . && rm -rf ._* && rm Data.zip
cd ../..


# Chapter 2
# No avialable data

# Chapter 3
cd ch3*/data
curl -o mdve.dta https://masteringmetrics.com/wp-content/uploads/2015/02/mdve.dta 
cd ../..

# Chapter 4
cd ch4*/data
curl -o mlda.dta https://masteringmetrics.com/wp-content/uploads/2015/01/AEJfigs.dta
cd ../..

# Chapter 5
cd ch5*/data
### Bank failures in Mississippi in 1930
curl -o banks.csv https://masteringmetrics.com/wp-content/uploads/2015/02/banks.csv
### MLDA
curl -o deaths.dta https://masteringmetrics.com/wp-content/uploads/2015/01/deaths.dta
cd ../..

# # Chapter 6
# cd ch6*/data
# # Twinsburg
# curl -o AA_small.zip https://masteringmetrics.com/wp-content/uploads/2015/02/AA_small.dta_.zip
# unzip -j AA_small.zip -d . && rm ._* && rm AA_small.zip
# # IV analysis
# curl -o ak91.dta https://masteringmetrics.com/wp-content/uploads/2015/02/ak91.dta
# # Sheepskin Effects
# curl -o clark_martorell_cellmeans.dta https://masteringmetrics.com/wp-content/uploads/2015/02/clark_martorell_cellmeans.dta
# cd ../..
```

## 利用 Python 因果推断
### `statsmodels`
计量经济学的主要武器（双重差分、断点回归等）都是基于回归进行的，而statsmodels就提供了一系列对回归分析的支持。与scikit-learn更侧重回归预测相比，statsmodels确实更加侧重标准误的计算以及对统计显著性的分析，而且许多回归结果的标准误计算和`STATA`几乎是一样的（在这点上，R做的并不出色）。事实上，大多数`STATA`能做到的事，`statsmodels`也能做到。尤其是`statsmodels.formula.api`提供了类似 `R` 语言的回归公式 API 支持，可以通过公式直接指定类别变量、添加交互项等，避免了手动清洗数据和特征工程的麻烦。

美中不足的是，statsmodels的API设计和scikit-learn的API设计不太一致，这会让有机器学习经验的开发者在使用`statsmodels`时感到不太适应。此外，`statsmodels`的类型注释做的并不是特别好，因此IDE的代码补全功能也不是特别友好，在开发过程中可能需要人为记忆不少常用的函数和方法。

#### 回归公式支持
大多数网上的教程会告诉你这么使用statsmodels来跑回归：
``` python
import statsmodels.api as sm

data = pd.read_csv()

y = data['target']
X = data[['treatment', 'control']]
X = sm.add_constant(X)

model = sm.OLS(y, X, missing='drop')
results = model.fit()
print(results.summary())
```
这当然没错。但是这样的做法有如下问题：
1. 需要预先特征工程。如果数据集中存在类别变量，需要转换为0-1哑变量，则需要调用`pd.get_dummies`函数，再对表格拼接处理。而在`STATA`中，只需要`reg y x i.categorical`即可。
2. 不直观。相比于`STATA`中`reg y x control, robust`语法，上述代码很难一眼看出谁对谁跑回归。
3. 造成太多的模板代码（boilerplate）。比如每次跑回归都要调用`sm.add_constant`函数，而这点在`STATA`中是默认的。

其中，第一点问题是尤其致命的。幸运的是，在`statsmodels`的大多数回归模型中，利用回归公式可以解决上述三个问题：
```Python
import statsmodels.formula.api as smf

data = pd.read_csv()
model = smf.ols(
    formula='target ~ treatment + control',
    data=data,
)
results = model.fit()
print(results.summary())
```
上述回归公式一目了然。在`~`左边是因变量，右边是自变量，用`+`拼接。

回归公式的细节如下：
* 利用`C(c)`将类别变量自动转化为0-1变量。比如，`outcome ~ treatment + C(c)`，等价于`STATA`中的`reg outcome treatment i.c`
* 利用`c1 : c2`生成交互项，等价于`reg outcome treatment c1#c2`
* 利用`c1 * c2`额外添加交互项，即整个回归方程会同时包含`c1, c2`以及两者形成的交互项，等价于`reg outcome treatment c1*c2`

#### 聚类标准误

statsmodels还允许计算聚类标准误，而非普通的稳健标准误，这只需要在调用`fit`方法时设定`cov_type`参数即可。例如，在本repo的[deaths.ipynb](https://github.com/DWizzi/mastering-py-metrics/blob/master/ch5_differences_in_differences/notebooks/deaths.ipynb)中，在跑双重差分时，需要基于州计算聚类标准误：

```Python
formula = 'mrate ~ legal + beertaxa + C(state) * year + C(year)'

model = smf.ols(
    formula=formula,
    data=data
)

# regressing with clustered standard error
results = model.fit(
    cov_type='cluster',
    cov_kwds={'groups': data['state']}
)
```

### `DoWhy`
`DoWhy`是一个强大的因果推断和因果探索包。和计量经济学派侧重简约式回归（Regression in reduced form）不同，它是基于因果图模型来进行因果推断的。因此，`DoWhy`在工业界似乎更加侧重于基于S/T/X-Learner学习CATE（即条件平均处理效应，或ITE/HTE，个体处理效应/异质处理效应）。在计量经济学派进行政策评估、计算ATE（平均处理效应）时，其使用的大多数方法都属于后门准则（backdoor）提供的estimand的具体estimator，其中大部分具体方法就是通过`statsmodels`来实现的。

在计量经济学应用中，大致上可以分为如下几步：

#### 1. 建立因果图模型
```Python
from dowhy import CausalModel

data = pd.read_csv()
graph = """
digraph {
    from_node -> to_node;
    ...
}
"""

model = CausalModel(
    data=data,
    treatment="treatment_variable",
    outcome="outcome_variable",
    graph=graph,
)

model.view_model()
```
`view_model`方法会将`graph`所定义的DAG可视化出来。在进行具体的因果效应计算之前，最好检查DAG是否有误。

#### 2. 识别estimand
```Python
identified_estimand = model.identify_effect()
identified_estimand.estimands
```

#### 3. 计算estimates
```
causal_estimate = dowhy_did_model.estimate_effect(
    identified_estimand,
    method_name=method_name,
    test_significance=True
)

print(causal_estimate_reg)
```

[常见的`method_name`](https://www.pywhy.org/dowhy/v0.9/example_notebooks/dowhy_estimation_methods.html)如下：
* "backdoor.linear_regression"
* "backdoor.distance_matching"
* "backdoor.propensity_score_stratification"
* "backdoor.propensity_score_matching"
* "backdoor.propensity_score_weighting"
* "iv.instrumental_variable"
* "iv.regression_discontinuity"

需要注意的是，_Mastering 'Metrics_ 本书中，第四章的断点回归方法并不是上述`method_name`中的"iv.instrumental_variable"。因为，要想DoWhy识别estimand中存在IV方法，工具变量必须满足如下的因果通路：
```Python
graph = "digraph { iv -> treatment; treatment -> outcome;}"
```
而在本书第四章的[最低合法饮酒年龄的案例](https://github.com/DWizzi/mastering-py-metrics/blob/master/ch4_regression_discontinuity_designs/notebooks/mlda.ipynb)中，年龄作为分配变量（running variable），实际上是后门准则意义上的confounder而非工具变量。即满足分叉（Fork）结构的因果图：
```Python
graph = "digraph { age -> death_rate; age -> over21; over21 -> death_rate;}"
```
因此，应当采用"backdoor.linear_regression"来识别因果效应。