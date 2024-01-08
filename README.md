# mastering-py-metrics

## 介绍
利用 Python 实现《精通计量：因果之道》（_Mastering 'Metrics_ by Joshua Angrist）的实证因果推断案例，基于业界常用的`statsmodels` 和 `DoWhy`因果推断包实现。

## 开始之前
### 环境配置
**注意，DoWhy 似乎目前不支持 M 系列 mac 芯片**
* macOS / Linux 环境下
```bash
conda create --name mastering-py-metrics python=3.11
conda activate mastering-py-metrics
which pip # make sure environmnet is activated
pip install -r requirements.txt
```

### 下载数据集
* Linux 环境下
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
cd ../..
```