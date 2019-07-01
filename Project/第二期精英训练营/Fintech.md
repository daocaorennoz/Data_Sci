# 基于房屋特征的交易价格预测复盘

[TOC]

## 按照典型机器学习流程来整理

### 问题抽象

通过房屋的特征来预测房屋交易的价格，那么这可以被抽象为回归问题。

### 获取数据

这一部分不需要进行，数据集已经给出。
简单罗列一下：

主表：

训练数据总共有2W多条，测试数据为600多条。

初始维度为21维，其中price维是预测目标。

初始维度list如下：

    id                 int64            # 编码
    data              object            # 售出日期
    price            float64            # 价格
    bedrooms           int64            # 卧室数目
    bathrooms        float64            # 盥洗室数目
    sqft_living      float64            # 居住面积
    sqft_lot         float64            # 地皮面积
    floors           float64            # 层数
    waterfront         int64            # 是否是海景房
    view               int64            # 被看过多少次
    condition          int64            # 
    grade              int64            # 评分系统给定的评分
    sqft_above       float64            # 地上面积
    sqft_basement    float64            # 地下室面积
    yr_built           int64            # 建造年份
    yr_renovated       int64            # 是否被再次装修过
    zipcode            int64            # 邮编
    lat              float64            # 纬度
    long             float64            # 经度
    sqft_living15    float64            # 表示2015年的居住面积
    sqft_lot15       float64            # 表示2015年地皮面积

附表：

包括沼泽地、公园、公立学校、私立学校、轻轨站、垃圾填埋场、抢劫信息等信息


### 特征工程

研究过题目，对于美国西雅图房屋的购买这个场景，我们并不是非常熟悉，对于其中因素的重要程度不得而知，所以我们通过参考美国专业房屋交易平台Zillow来探寻该问题的重要特征。

简要将所有的特征分为三个部分：
    
    1. 房屋固有特征：面积，房间等
    2. 房屋地理属性特征：周围交通、教育、安全等因素。
    3. 交易特征： 其实就是交易时间

特征工程也相应的从这三方面来进行：

固有特征：
    
- 庭院总面积

```python
df_train['total'] = df_train.sqft_above + df_train.sqft_basement
df_train['totalb10'] = 0 
df_train['totalb10'].loc[df_train['total']>10] = 1
```

- 计算户外花园面积

```python
df_train['garden'] = df_train.sqft_lot - df_train.sqft_living /df_train.floors
```

- 居住面积占比

```python
df_train['living_pro'] = df_train['sqft_living'] / df_train['sqft_lot']
```

- 房间数目：
    1. 数据分析可得bathrooms的数据不是整数，有0.25,0.5,0.75的小数部分，于是从这方面入手：查资料可得美国按照盥洗室的设施配置将bathrooms划分成四种：
    - full： 代表sink+bathtub+toilet+shower
    - 0.75： 上述四个中有三个
    - 0.50： 上述四个中有两个
    - 0.25： 上述四个中只有一个
    因为数据集中只有bathrooms的总数，所以只能认为有小数的记录肯定有两种bathrooms构成
    example：
        1.75 = 1 + 0.75 表示共有一个完整的bathroom和一个有三个设备的bathroom
        而不能认为 1.75 = 1 + 0.25 + 0.5
    于是将原始特征中的bathrooms拆分为 四个特征维，然后将bathrooms删掉

```python
df_train['fullbath'] = df_train['bathrooms'].apply(lambda x:int(x))
df_train['elsebath'] = df_train['bathrooms'] - df_train['fullbath']
df_train['threequaterbath'] = 0
df_train['twoquaterbath'] = 0
df_train['onequaterbath'] = 0
df_train['threequaterbath'].loc[df_train['elsebath'] == 0.75] = 1
df_train['twoquaterbath'].loc[df_train['elsebath'] == 0.50] = 1
df_train['onequaterbath'].loc[df_train['elsebath'] == 0.25] = 1
df_train.drop(['elsebath','bathrooms'],axis=1,inplace=True)
```

    2. 计算房屋总数目： bedrooms + 拆分的bathrooms的个数

- 每个屋子的平均大小

```python
df_train['averroomsize'] = df_train['sqft_living'] / df_train['total_room']
```

- 经纬度处理

```python
df_train['location'] = df_train['lat'] + df_train['long']
df_train['location_1'] = df_train['lat'] * df_train['long']
```



```python
def area_extract(df_train,flag):
    # 计算庭院总面积 
    df_train['total'] = df_train.sqft_above + df_train.sqft_basement
    df_train['totalb10'] = 0 
    df_train['totalb10'].loc[df_train['total']>10] = 1
    
    # 计算房间数
    df_train['total_room'] = df_train['bathrooms'] + df_train['bedrooms']
    
    # 计算平均roomsize
    df_train['averroomsize'] = df_train['sqft_living'] / df_train['total_room']
    
#     # 没有bathroom的bedrooms个数
#     df_train['nobath'] = df_train.bedrooms-df_train.bathrooms
#     df_train['nobath'].loc[df_train['nobath'] < 0] = 0
    
    # 户外花园占比
#     df_train['garden_pro'] = (df_train.sqft_lot - df_train.sqft_living) / df_train.sqft_lot
    df_train['garden'] = df_train.sqft_lot - df_train.sqft_living /df_train.floors
    
    # 居住面积占比
    df_train['living_pro'] = df_train['sqft_living'] / df_train['sqft_lot']
    
    # 经纬度处理
    df_train['location'] = df_train['lat'] + df_train['long']
    df_train['location_1'] = df_train['lat'] * df_train['long']

    # 拆分bathroom
    df_train['fullbath'] = df_train['bathrooms'].apply(lambda x:int(x))
    df_train['elsebath'] = df_train['bathrooms'] - df_train['fullbath']
    df_train['threequaterbath'] = 0
    df_train['twoquaterbath'] = 0
    df_train['onequaterbath'] = 0
    df_train['threequaterbath'].loc[df_train['elsebath'] == 0.75] = 1
    df_train['twoquaterbath'].loc[df_train['elsebath'] == 0.50] = 1
    df_train['onequaterbath'].loc[df_train['elsebath'] == 0.25] = 1
    df_train.drop(['elsebath','bathrooms'],axis=1,inplace=True)
    
    if flag:
        df_train['averprice'] = df_train['price'] / df_train['sqft_living']
```

地理属性特征：

- 轻轨站

    按照给定的地理位置经纬度信息，计算出房屋到每个地点的距离，设定阈值来衡量该地的相应情况。

按照上述的方法处理其余的附表信息。

交易属性特征：

- 时间的处理：

按照建房时间和交易时间得到差值，成为房屋的折旧时间。

```python
df_train['date_format']= pd.to_datetime(df_train.date,format = '%Y-%m-%d')
df_train['year'] = df_train['date_format'].apply(lambda x: x.year)
df_train['month'] = df_train['date_format'].apply(lambda x:x.month)
df_train['day'] = df_train['date_format'].apply(lambda x:x.day)
df_train['deltaY'] = df_train.year - df_train.yr_built + 0.083 * df_train.month + 0.033 * df_train.day
df_train.drop(['date_format','year','month','day'],axis = 1,inplace = True)
```

并按照时间进行年份和月份的交叉。

完成上述的特征工程之后，我们得到了4092维特征。

接下来进行特征的筛选。

### 特征筛选


### 模型调优


### 模型融合


### 展望

### 尝试的其他思路

利用经纬度来