# 数据和数据库

到目前为止我们学习的“数据”都是程序里处理的数据，也就是在内存中的数据，这一章我们要介绍“持久化”也就是把数据永久保存下来的技术，包括数据文件和数据库服务。这一章的内容颇多，有些内容也不是一下就能理解的，所以也是需要反复学习的一章，尤其是在实践中需要应用时可以再次回来温习。

## 程序内的数据

所谓数据，就是计算机记录下来的“事实（*facts*）”。计算机程序处理的任何事实都是数据，比如我们最经常碰到的“用户数据”：


```python
user = { 
    'uid': 1,
    'name': 'Neo',
    'password': '2733bbf0e6a7b9a919d7cca6a7c0f74a',
    'email': 'neo@matrix.net',
    'mobile': '101-0101'
}
```

这就是典型的数据，属于一个人的各种属性，数据类型各不相同，但都是关于这个人的。这里我们用了 Python 里的 *dict* 来记录这些数据，还有很多别的办法，比如用一个类和对象也可以：


```python
def salt_and_hash(password):
    # 对用户密码进行加密
    return password

class User:
    def __init__(self, uid, name, password, email, mobile):
        self.uid = uid
        self.name = name
        self.password = salt_and_hash(password) # 我们不能直接把用户的密码保存下来，所以保存之前需要先加密
        self.email = email
        self.mobile = mobile

user = User(1, 'Neo', 'thereisnospoon', 'neo@matrix.net', '101-0101')
user.email
```




    'neo@matrix.net'



我们在现实世界很少只面对一个对象，我们编写程序通常是为了处理不确定数量的一类对象。假设我们有 N 个用户，每个用户都对应一个上面这样的 *User* 类型的对象，我们可以把这 N 个对象放进一个 *list*，假定我们要打印出所有用户的用户名和 Email，只要这样：


```python
users = []
users.append(User(1, 'neo', 'thereisnospoon', 'neo@matrix.net', '101-0101'))
users.append(User(2, 'trinity', 'iloveyouneo', 'trinity@matrix.net', '101-1010'))
users.append(User(3, 'morpheus', 'ibelieve', 'morpheus@matrix.net', '101-1111'))

for user in users:
    print(f'uid={user.uid}: name={user.name}, email={user.email}')
```

    uid=1: name=neo, email=neo@matrix.net
    uid=2: name=trinity, email=trinity@matrix.net
    uid=3: name=morpheus, email=morpheus@matrix.net


如果我们还可以对这些数据进行各种各样的读写操作：


```python
# 找出用户名叫 ‘trinity’ 的用户的手机号码
for user in users:
    if user.name == 'trinity':
        print(user.mobile)
```

    101-1010



```python
# 找出用户名叫 ‘trinity’ 的用户，将其手机号码改为 ‘101-0001’
for user in users:
    if user.name == 'trinity':
        user.mobile = '101-0001'
```


```python
# 增加一个名叫 ‘merovingian’ 的用户
users.append(User(4, 'merovingian', 'iknoweverything', 'merovingian@matrix.net', '777-7777'))
```


```python
# 找到手机号码不以 ‘101’ 开头的用户并将其删除
for user in users:
    if not user.mobile.startswith('101'):
        users.remove(user)
```

我们经常把对数据的处理概括为 *CRUD*，即创建（*create*）、读取（*read*）、更新（*update*）和删除（*delete*），正如我们上面展示的。

通过现代编程语言提供给我们的各种强大的数据类型，我们可以轻易的表达和处理各种各样的数据，但是仔细思考一下，有一些事情并不那么好做。

**第一个问题**是“**持久化**（*persistency*）”。

我们会经常在技术文章中看到这个词儿和它的变体，这看上去是个很高深莫测的词儿，但它的含义其实很简单：就是如何让数据不会因为计算机断电而丢失——换句话说，我们需要一种办法能把内存里的数据（比如上面的 *dict* 或者 *objects*）长久的保存起来，并在需要时随时恢复到内存中去。

当然我们知道，可以把数据存到硬盘这样的“持久化设备”上。我们需要约定好保存的格式，程序知道怎么写，也知道怎么读，读出来之后恢复成上面那样的 *object* 就行了。

有一种最简单的数据文件格式叫做 *CSV*，*comma-separated values*，也就是用逗号隔开的值。

比如我们可以把 `users` 这样的列表变成的一个文本文件，列表中的每个元素变成文件中的一行，这一行是这个元素的各个属性，属性之间用逗号隔开；我们还可以在文件最开始加一行，记录每个属性的名字是啥。比如上面的 `users` 转成 *CSV* 格式的文件是这样子的：

```
uid, name, password, email, mobile
1, 'neo', 'thereisnospoon', 'neo@matrix.net', '101-0101'
2, 'trinity', 'iloveyouneo', 'trinity@matrix.net', '101-1010'
3, 'morpheus', 'ibelieve', 'morpheus@matrix.net', '101-1111'
```

*CSV* 格式非常好用，简单直接，我们不用借助特殊的程序就能读懂其内容，编写程序来读写 *CSV* 文件也比较容易，所以应用很广泛。很多主流软件比如微软开发的电子表格软件 Excel，都支持读写 *CSV* 格式的文件。

小结一下：我们解决持久化问题的方法是，设计一种格式，把内存中的数据保存成这种格式的文件，在需要时可以读取这种格式的文件还原成内存中的对象。这两个操作通常称为序列化（*serialization*）和反序列化（*deserialization*）。

**第二个问题**是“**并发**（*concurrency*）”。

这个问题更加麻烦很多倍。如果我们只有几千个用户，那么上面的 *object/dictionary* 加 *list* 的方法没啥问题，但如果我们有几百万几千万的用户，我们该怎么做呢？假定一个对象占用 100 字节（这算少的），那么 1000 万用户的数据会占用 10 亿字节，也就是 1GB，在我们这个计算机内存动辄 16GB 的时代听上去还好对吧？但二十年前计算机内存还只有几兆或者几十兆，而且别忘了这只是用户数据，我们的系统不可能只处理用户数据，还有用户的行为和关联数据，比如订单之类（订单和其他一些行为数据规模是远大于用户数据的）。

除了内存占用，操作效率也是个问题，看下我们在前面写的示范代码，如果为了找到一个用户的数据动不动就要对一千万个对象循环一次，我们的程序肯定会慢的没法用。这就是并发带来的“**规模化**（*scalability*）”问题，基本上是软件后端开发的终极问题，**驾驭不同规模的能力决定了一个后端程序员的价值**。

解决这个问题的办法和持久化有关：我们可以把完整的数据保存在硬盘上，每次只把需要的一小部分数据读进内存，需要时再更新回硬盘上，毕竟无论数据量有多大，我们总可以一个一个的处理。

总结一下，大型软件应用的开发要求我们设计一种数据的存储格式：
* 能够存储海量数据；
* 能够处理不同数据之间的关联（比如用户和用户的订单）；
* 能够快速找到我们关心的极小部分数据；
* 能够支持我们对数据进行 *CRUD* 操作；
* 无论数据量多大都能提供可接受的性能。

更进一步，由于这些数据很可能至关重要（关系到经济利益甚至生命安全），所以需要完善的安全和容错设计：数据不能丢失，不当心改错了可以恢复，最好还能提前避免大部分数据里的逻辑错误，等等。

人们把专门用于存放数据的文件系统称为“数据库（*database*）”，以区别于其他一般用途的文件。上面的任务实际上就是设计数据库的存储格式及其管理系统（*database management system, DBMS*）。

这个非常有挑战的任务在在上个世纪 70 年代有了一个相当好的解决方案，那就是 IBM 的计算机科学家 [E. F. Codd](https://en.wikipedia.org/wiki/Edgar_F._Codd) 设计的“关系型数据库（*relational database*）”，他在论文 “*A Relational Model of Data for Large Shared Data Banks*” 中详细阐述了 “*relational*” 的含义，这一理论被称为“**关系模型**（*relational model*）”，是关系型数据库设计实现和运用的基础，按照关系模型设计的数据库管理系统就叫做“关系型数据库管理系统（*RDBMS*）”。

后来为了避免数据库厂商滥用 *RDBMS* 这个招牌， E. F. Codd 还编写过一个“[十二法则](https://en.wikipedia.org/wiki/Codd%27s_12_rules)”（编号从 0 到 12，所以实际上有十三条）来界定“什么样的系统才算 *RDBMS*”，有兴趣可以读一下，但可能对初学者不是那么太友好——所以这里我们尝试用尽可能简单的方式解释下关系模型和 *RDBMS*。

## 关系模型

首先，我们要理解关系模型的一系列关键概念：

### 表

关系模型将数据模型抽象为若干二维表（*table*），每一张表对应一种实体（*entity*）类型，比如用户是一种实体类型，订单也是，这两个实体类型就在关系模型中对应——比如分别叫 `user` 和 `order` 的——两张表。

如果不好理解，就想象一下我们使用的 Excel 电子表格，基本上就是那样的表。

### 行与列

表由行（*row*）和列（*column*）组成：
* 每一行代表了这种实体类型下的一个实例（*instance*），比如 `user` 表的每一行就代表一个用户，而 `order` 表里的每一行就代表一张订单，行也可以叫做“记录（*record*）”；
* 而列代表了这个实体类型的一种特性，比如用户表里的用户名、Email、手机号码等，列也可以叫做属性（*attribute*）或者字段（*field*），一般我们在谈论一张表的时候会用 *column* 这个说法，而在提到具体一行数据时用 *attribute* 或者 *field* 的说法。

### 主键

每张表中必须有能唯一确定某一行数据的值，可能是一个字段或者几个字段组合起来的值，这个用来唯一确定一行数据的值称为“主键（*primary key*）”，依定义，每一行的主键值在表内必须唯一；这个概念是为了更容易实现 *RDBMS*，如果没有这个规则，我们要判断一个表里有没有重复记录就要比较每一行的每一个字段，而在这个规则下我们只需要针对少数（通常就是一个）字段比较就可以了。

有了主键之后我们可以非常容易的定位某一行数据，查询其内容，或者修改、删除这一行数据，也可以在增加数据时检查是否重复数据。

### 外键

两个表 A 和 B 之间可以建立关联，方法是在 B 表中增加一个列，这个列里存放 A 表的 *primary key* 值，这样 B 表的每一行就关联到了与之相关的 A 表的某一行，B 表里的这个列就称为“外键（*foreign key*）”。

以用户和订单表为例，我们假定用户表的主键是 `uid` 字段（每行用户数据的 `uid` 是唯一的），在订单表里加上一列称为“对应用户 `uid`”，这样每一个订单都可以通过这个 `uid` 对应到一个用户，并在用户表里找到该用户的具体信息（名字、电话和 Email 等），这些信息并不在订单表而在用户表里，通过外键就不同实体连接起来，组合成一个复杂的业务逻辑系统。

## RDBMS

顾名思义，*RDBMS* 就是管理关系型数据库的系统，它按照关系模型实现了数据的存储，然后提供标准化的接口让其他应用程序可以方便的使用关系型数据库，这里面主要是下面这些工作。

### 数据操作和事务处理

正如我们前面说的，对数据的操作可以概括为 *CRUD*，*RDBMS* 通过一种通用操作语言 *SQL*（*Structured Query Language*）来提供这些操作支持。

SQL 是目前需求最广泛的编程语言。这种语言用标准语法实现对关系模型里的数据以及数据结构的 *CRUD*。

不同 *RDBMS* 中 *SQL* 语法是基本一致的（各系统会有一些自己的扩展），这也让我们操作这些 *RDBMS* 变得更容易。

随着行业的进步，大部分现代编程语言都提供一些方法来封装对 *RDBMS* 的操作，很多时候我们都不需要直接使用 *SQL*，但适当了解一些基本知识和工具仍然是非常有益的。我们会在后面给出一些例子。

如果对数据的一组操作中涉及任何修改（包括增加、删除和修改数据或数据的结构等）的，这组操作就属于“事务（*transaction*）”。

因为大多时候一个数据库会被多个应用所共享，多个应用实例要修改同一组数据会带来很多麻烦，所以事务处理是一个非常敏感的部分，一般 *RDBMS* 都要做很多工作来保障事务处理的正常进行。这个问题我们后面还会讨论。

### 维护数据一致性

什么叫一致性（*consistency*）呢？

比如我们有一张订单，上面说这个是张三的，但用户表里没有张三这个人，这种糟糕的现象就叫“数据不一致”。为了避免数据不一致，需要对数据库进行良好的设计，还需要在使用过程中做很多事情，这里面 *RDBMS* 扮演了很重要的角色。

关系模型中的主键和外键有助于我们实现数据的一致性。通常 *RDBMS* 会在进行数据操作时对一系列数据约束（*constraint*）进行检查，例如：
* 增加一行数据时会检查，是否缺失必有的字段，比如没有主键是肯定不行的，还有其他数据库设计和管理者定义的不可缺失的字段；还会检查这行数据里如果有外键，其值是不是合法，它不能是关联表里不存在的值；
* 删除一行数据时会检查，这行数据是不是有被其他表用外键关联到，如果有，则不能删除（否则就会出现上面说的那种“荡空”数据）。

### 数据访问优化

数据库系统是针对海量数据的存储和访问设计的，所以性能方面的优化至关重要。一个优秀的 *RDBMS* 应该自动优化大部分访问的性能，并为应用提供方便的工具来优化数据读写效率。

一个典型例子就是**数据库索引**（*index*）。

假设我们经常针对某个列进行查询，比如查找 XX 年以后出生的用户，*RDBMS* 可以针对常用的列建立索引，极大地提升这类查询的效率，因为数据表可能很大，要把整个表过一遍来找到我们要的数据会比较慢，但如果事先知道我们会通过出生年份来查找，我们可以指示 *RDBMS* 提前把这一列数据做成一个很小的索引文件，查找时先在索引中找到我们要的数据行（的主键），再通过主键取出完整数据行；*RDBMS* 还会在增删改数据时自动更新索引文件——当然这会带来性能开销，所以索引也不能乱建，也要考虑经济实惠。 

### 数据库集群

随着数据量的不段增加，很多数据库很快会突破一台服务器能支撑的限度（运算或者存储能力），这时候我们就需要把逻辑上的一个数据库分开放在几台物理服务器上，有时候这需要我们在应用上写程序实现分片（某些数据在服务器A，某些在 B），也有些 *RDBMS* 提供对应用透明的集群实现，应用不管 *RDBMS* 怎么分服务器，就当一整个来使用。不同的方案各有利弊，但都需要 *RDBMS* 做出某种程度的支持。

### 数据库安全

前面提过，数据的安全至关重要，*RDBMS* 都会提供完善的数据安全解决方案，包括权限管理、主从备份、灾难恢复等。

### 主流 RDBMS

目前被广泛使用的 *RDBMS* 主要有：
1. 以 Oracle、Microsoft SQL Server、IBM DB2 为代表的大厂商出品的系统，一般来说用于非常大的企业应用系统，价格也非常高昂；
2. 以 MySQL、PostgreSQL 为代表的开源系统，广泛用于各种规模的互联网应用，因为开源，很多巨型互联网公司会对其进行修改定制以满足自己的需要；注意 MySQL 2008 年被 Sun 收购，2010 年 Sun 又被 Oracle 收购，所以现在 MySQL 实际上是属于 Oracle 的资产，但运作上有一定的独立性；
3. 以 SQLite 为代表的极轻量级系统，一般用于单机和嵌入式环境（iPhone app 中通常都有几个 SQLite 数据库）。

## RDBMS 以外

关系型数据库经过几十年发展，从 *RDBMS* 产品到设计方法论都已经非常成熟和完善，但其设计毕竟是四、五十年前的理念，在过去十几年互联网应用飞速发展中，人们也通过不少创新来弥补关系型数据库的短板，这里面最主要的是 *NoSQL* 和以 *Redis* 为代表的内存数据库的兴起。

关系型数据库主要针对企业应用和交易处理，对数据的一致性、交易的完整性和系统的安全性有很高的要求，一般企业应用的同时并发用户也不会特别多；而最近二十年兴起的互联网应用则不同，互联网应用中经常有不太复杂的操作，但是并发惊人，比如很多人同时登录，这个操作不复杂（检查用户名密码是否匹配，然后记一下日志），但可能同时有几十万甚至更多人一起发起这个操作；再比如抽奖，这个操作就是生成一个随机数，然后看在不在获奖的范围里，但是可能无数人同时点击抽奖按钮，应用需要记录每个人生成的随机数并写到数据库里备查，这些操作对传统 *RDBMS* 来说真是不堪重负（或者说实现起来性价比很低）。

互联网应用的开发者为了解决这些问题，找到的办法是砍掉 *RDBMS* 中一些在这类场景中不必要的东西，比如数据一致性检查，甚至关系模型本身也可以简化为内存里保存的海量 *key-value* 对。基于这些应用场景和解决方案，有一系列新的技术被开发出来，我们以后会逐步了解这些技术的差异和各自擅长，这里先不展开。

## Python 中的实例

下面我们通过一系列实例来了解下 Python 中使用各种数据文件和数据库的方法。

### CSV 数据文件

Python 中使用 CSV 数据文件，因为有 [pandas](https://pandas.pydata.org/) 这个第三方库而变得非常轻松和愉快。

在运行下面的代码之前记得先在命令行界面安装 `pip install pandas`。

互联网上有很多开放数据是以 CSV 格式提供的，这里我们用于示范的是来自纽约现代艺术博物馆（*Museum of Modern Art, MoMA*）的[开放数据](https://github.com/MuseumofModernArt/collection)，包括馆藏的所有作品及其艺术家。因为作品数据集太大，我们只用艺术家数据集（即附件中的 `moma-artists.csv` 文件）。


```python
import pandas as pd

data = pd.read_csv('assets/moma-artists.csv')
data.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ConstituentID</th>
      <th>DisplayName</th>
      <th>ArtistBio</th>
      <th>Nationality</th>
      <th>Gender</th>
      <th>BeginDate</th>
      <th>EndDate</th>
      <th>Wiki QID</th>
      <th>ULAN</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>Robert Arneson</td>
      <td>American, 1930–1992</td>
      <td>American</td>
      <td>Male</td>
      <td>1930</td>
      <td>1992</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>Doroteo Arnaiz</td>
      <td>Spanish, born 1936</td>
      <td>Spanish</td>
      <td>Male</td>
      <td>1936</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Bill Arnold</td>
      <td>American, born 1941</td>
      <td>American</td>
      <td>Male</td>
      <td>1941</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>Charles Arnoldi</td>
      <td>American, born 1946</td>
      <td>American</td>
      <td>Male</td>
      <td>1946</td>
      <td>0</td>
      <td>Q1063584</td>
      <td>500027998.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>Per Arnoldi</td>
      <td>Danish, born 1941</td>
      <td>Danish</td>
      <td>Male</td>
      <td>1941</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>5</th>
      <td>6</td>
      <td>Danilo Aroldi</td>
      <td>Italian, born 1925</td>
      <td>Italian</td>
      <td>Male</td>
      <td>1925</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>6</th>
      <td>7</td>
      <td>Bill Aron</td>
      <td>American, born 1941</td>
      <td>American</td>
      <td>Male</td>
      <td>1941</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>7</th>
      <td>9</td>
      <td>David Aronson</td>
      <td>American, born Lithuania 1923</td>
      <td>American</td>
      <td>Male</td>
      <td>1923</td>
      <td>0</td>
      <td>Q5230870</td>
      <td>500003363.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>10</td>
      <td>Irene Aronson</td>
      <td>American, born Germany 1918</td>
      <td>American</td>
      <td>Female</td>
      <td>1918</td>
      <td>0</td>
      <td>Q19748568</td>
      <td>500042413.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>11</td>
      <td>Jean (Hans) Arp</td>
      <td>French, born Germany (Alsace). 1886–1966</td>
      <td>French</td>
      <td>Male</td>
      <td>1886</td>
      <td>1966</td>
      <td>Q153739</td>
      <td>500031000.0</td>
    </tr>
  </tbody>
</table>
</div>



pandas 提供的 `read_csv` 函数读取一个 *CSV* 文件，并将其结果装载到数据集对象 `data` 中，`data` 的数据类型是 `pandas` 包引入的 `DataFrame`，准确的是说 `pandas.core.frame.DataFrame`，可以将其看做一个二维表（实际上底下就是个 [NumPy](https://numpy.org/) 数组），是关系模型在 Python 中的一种实现。

我们可以看到，Jupyter Notebook 对 `DataFrame` 类型的对象提供了专门的显示格式，会自动显示出漂亮的表格。如果我们在 Python 解释器命令行里运行上面的代码，输出没有这么漂亮，但也很清晰。

现在我们仔细审视下上面的表：
* 这个表有 10 行（因为我们调用 `data.head()` 时指定了输出前 10 行），不算最左边 `DataFrame` 内置的序号一共 9 列；
* 这是一张艺术家的数据表，每一行数据记录代表一位艺术家，上面一共列出了 10 位艺术家；每位艺术家的数据里有 9 个字段（列），分别是艺术家 ID、姓名、简介、国籍、性别、生卒年、Wiki QID 和 [ULAN 网络](https://www.getty.edu/research/tools/vocabularies/ulan/) ID。

`DataFrame` 提供了一个强大的查询方法 `query` 帮助我们用灵活的条件组合来查询数据，下面是一些例子：


```python
# 查询 1990 年（出生或开始活跃）的艺术家
data.query('BeginDate >= 1990 and BeginDate < 1991')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ConstituentID</th>
      <th>DisplayName</th>
      <th>ArtistBio</th>
      <th>Nationality</th>
      <th>Gender</th>
      <th>BeginDate</th>
      <th>EndDate</th>
      <th>Wiki QID</th>
      <th>ULAN</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>8518</th>
      <td>11159</td>
      <td>Lexon, France</td>
      <td>est. 1990</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>10523</th>
      <td>29566</td>
      <td>OFFECCT</td>
      <td>Sweden, est. 1990</td>
      <td>Swedish</td>
      <td>NaN</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>11545</th>
      <td>34168</td>
      <td>Kengo Kuma &amp; Associates</td>
      <td>Japan, est. 1990</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>12469</th>
      <td>37888</td>
      <td>Werkplaats Martin van Oel</td>
      <td>Dutch, est. 1990</td>
      <td>Dutch</td>
      <td>NaN</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>14254</th>
      <td>46529</td>
      <td>Joseph Pleass</td>
      <td>British, born 1990</td>
      <td>British</td>
      <td>Male</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>14703</th>
      <td>48039</td>
      <td>Jacqueline Yuan Quinn</td>
      <td>NaN</td>
      <td>American</td>
      <td>Female</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>15514</th>
      <td>69266</td>
      <td>Richard Malone</td>
      <td>Irish, born 1990</td>
      <td>Irish</td>
      <td>Male</td>
      <td>1990</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
# 查询国籍为中国、70 年代以后活跃的艺术家
data.query('Nationality == "Chinese" and BeginDate >= 1990')
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ConstituentID</th>
      <th>DisplayName</th>
      <th>ArtistBio</th>
      <th>Nationality</th>
      <th>Gender</th>
      <th>BeginDate</th>
      <th>EndDate</th>
      <th>Wiki QID</th>
      <th>ULAN</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>13386</th>
      <td>42126</td>
      <td>Birdhead</td>
      <td>Chinese, active 2004</td>
      <td>Chinese</td>
      <td>NaN</td>
      <td>2004</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>15645</th>
      <td>70012</td>
      <td>Archi-Union Architects</td>
      <td>Chinese, est. 2003</td>
      <td>Chinese</td>
      <td>NaN</td>
      <td>2003</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



`DataFrame` 还有很多好用的属性和方法，比如 `columns` 返回所有列的显示名称，`apply()` 可以对行或列执行一个指定的函数，等等。[官方文档](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html)详细的介绍了所有这些属性和方法。

> pandas 提供了强大能力来对数据进行展示和分析，我们一般将这类工具称为数据展示和分析工具，它可以加载各种数据源，数据源可以是数据文件也可以是数据库。本节展示的例子是加载一个 *CSV* 数据文件，后面几节我们会看到其他数据源的例子。
> 
> 数据的存储及管理，与数据的展示及分析可以完全解耦，各做各的互不影响，还能完美配合，就是因为大家都基于相容的关系模型（行和列）。

### JSON 数据

[JSON](https://www.json.org/) 的全称是 JavaScript Object Notation，是另一种非常流行的数据格式，它是 JavaScript 的原生数据格式，所有主流语言都有读写 *JSON* 格式的标准库，所以它可以看作是 Web 上交换数据的事实标准。

其实我们都见过这个格式，Python 的 *list* 和 *dict* 类型值的语法差不多就是 *JSON* 的语法格式，在 *JSON* 中一个对象是这么表示的：

```JSON
{ 
    'uid': 1,
    'name': 'Neo',
    'password': '2733bbf0e6a7b9a919d7cca6a7c0f74a',
    'email': 'neo@matrix.net',
    'mobile': '101-0101'
}
```

这就是 *dict* 嘛！如果是一组对象呢？那就用方括号括起来，和 *list* 一样：

```JSON
[
    { 
        'uid': 1,
        'name': 'Neo',
        'password': '2733bbf0e6a7b9a919d7cca6a7c0f74a',
        'email': 'neo@matrix.net',
        'mobile': '101-0101'
    },
    { 
        'uid': 2,
        'name': 'Trinity',
        'password': '2733bbf0e6a7b9a919d7cca6a7c0f74a',
        'email': 'trinity@matrix.net',
        'mobile': '101-0101'
    },
    { 
        'uid': 3,
        'name': 'Morpheus',
        'password': '2733bbf0e6a7b9a919d7cca6a7c0f74a',
        'email': 'morpheus@matrix.net',
        'mobile': '101-0101'
    }
]
```

我们可以对比一下 *CSV* 和 *JSON* 两种格式：
* *CSV* 更接近关系模型，而 *JSON* 更接近程序中的对象模型；
* *CSV* 表达的是一种二维表结构，而 *JSON* 可以表达多重嵌套的非常复杂的对象结构；
* *CSV* 格式上更加紧凑，几乎没有冗余，而 *JSON* 有很多冗余信息，同样的数据 *JSON* 会大很多（当然有些压缩技术可以一定程度减小这个差距）；
* 因为格式更加简单，*CSV* 在处理上性能要好很多。

所以如果我们要存储和传输的数据是比较整齐的多行数据（关系模型），*CSV* 有不少优势；但如果我们面对的是 *list*、*tuple*、*dict* 及其嵌套大杂烩，只有 *JSON* 能做到。

Quora 上[有个答案](https://www.quora.com/As-a-developer-which-format-do-you-prefer-to-transmit-data-XML-JSON-or-CSV/answer/John-Random-3)比较了常用的 *CSV*、*JSON* 还有 *XML*（这个格式是 HTML 的超集，我们这个课程中不做介绍）数据格式在数据交换中的优劣，有兴趣可以一读。

我们学习 *JSON* 格式最主要的原因是：绝大部分 Web 服务的输入和输出都是 *JSON* 格式。举个例子，我们看下[某在线教育服务商](https://www.genshuixue.com/)网站，它的课程是组织在一系列分类下面的，如果要获得某个特定分类，比如面向[初一](https://www.genshuixue.com/pc/courseCenter?categoryId=10&gradeId=200)的课程，系统后台是通过这个 Web 服务接口提供的：

`https://www.genshuixue.com/sapi/viewLogic/selectCourse/courses?course=all&size=12&subjectId=0&gradeId=200&categoryId=10`

在浏览器中打开这个地址，就可以看到系统返回的数据，就是一个 *JSON* 格式的字符串，我们可以用 Python 来访问这个服务地址、获得数据并进行处理：


```python
import urllib3
import json

url = 'https://www.genshuixue.com/sapi/viewLogic/selectCourse/courses?course=all&size=12&subjectId=0&gradeId=200&categoryId=10'

# 使用 urllib3 通过 HTTP 协议访问上面的 Web API，获得 HTTP 协议定义的一个 Response 对象
http = urllib3.PoolManager()
r = http.request('GET', url) # r 是一个 HTTPResponse 类型的对象
if r.status == 200: # 如果 Response 的状态码是 200 表示调用成功
    data = json.loads(r.data.decode('utf-8')) # 见下说明
else: # 
    print(f'访问服务失败，错误码：{r.status}')
```

上面的代码中，`urllib3` 是一个 HTTP 客户库（注意这是第三方库，需要先安装 `pip install urllib3`），通过这个库我们可以在程序中模拟一个“浏览器”去访问各种网址，通过这个库提供的功能得到一个 `HTTPResponse` 类型的对象 `r`，然后检查 `r` 携带的状态码 `r.status` 是不是 `200`，如是则表示 HTTP 访问成功，取得了我们想要的数据，否则打印出状态码退出。

如果访问成功，关键的一行代码是这样的：`data = json.loads(r.data.decode('utf-8'))`，这行代码可以分解成几层：
* `r.data`：HTTP 协议中定义的返回主数据，也就是我们通过协议实际申请的资源，是一个 JSON 格式的字符串；
* `r.data.decode('utf-8')`：上述字符串是用 UTF-8 编码的，因为数据中很可能有中文等非 ASCII 字符，编码是为了在传输过程中兼容所有硬软件，这里我们用 Python 字符串的 `decode()` 方法来解码，得到的是一个解码之后的 JSON 格式字符串，里面所有字符都是原样了；
* `json.loads(...)` 使用 `json` 库提供的方法加载一个 JSON 格式的字符串，输出一个 *dict* 类型的对象，是的，Python 中就是用 *dict* 对象来表示 JSON 格式数据的。

如果一开始搞不清楚，我们可以在代码中插入 `print()` 来查看每一小步输出的是什么，试试看，会对整个过程了解得更加清楚。

接下去我们就可以把 `data` 作为一个普通的 *dict* 对象来使用了，里面可能嵌套了好多层的 *dict* 和 *list* 对象：


```python
# 获取第一个课程的课程名（思考：为什么是这样的访问语法？）
data['data']['items'][0]['clazzName']
```




    '初一语数110分高分训练营'



除了本地数据文件和数据库管理系统，在线 Web 服务提供的数据接口也是一种非常重要的数据源，在互联网行业很多程序员不一定会与 *RDBMS* 打交道，但一定会和 Web 服务返回的 *JSON* 数据打交道。

我们在第六部分介绍 Web 应用时会继续学习使用 *JSON* 数据。

### SQLite 数据库

#### 数据连接和游标

前面介绍了两种文本型的数据格式，下面我们来看下数据库。首先看最轻量级的 *RDBMS*，[SQLite](https://www.sqlite.org/index.html)，SQLite 的数据库就是一个数据文件，下面例子使用的数据库文件 `flights.db` 已经放在 `assets` 自目录下。

Python 提供了访问 SQLite 数据库的支持，在下面的例子之前我们需要先了解两个重要概念：

第一个是“**数据连接**（*connection*）”。

一个连接表示一个使用数据库的通道；创建 *connection* 之后可以通过这个通道来访问数据库，执行 SQL 命令，对数据做增删改查（*CRUD*），直到我们关闭它；`sqlite3` 库提供的 `connect` 方法可以建立一个数据库连接，指定 SQLite 数据库文件即可。

第二个是“**数据游标**（*cursor*）”。

我们知道 *cursor* 是鼠标指针，但在数据库领域，*cursor* 代表指向数据库里某一行数据的游标指针。我们告诉数据库要访问某个数据集（可能是一张表，也可能是一个数据查询结果集），这个数据集可能很大很大，数据库管理系统接受我们的指令后并不会把整个数据集全扔给我们，而是建立一个 *cursor* 指向这个数据集的第一行，然后把这个 *cursor* 交给我们，这样我们就可以用这个 *cursor* 来一行一行的获取和处理数据，而不需要担心一次取回太多数据。

数据库管理系统通过 *connection* 和 *cursor* 与我们的程序来通信和协作，这个处理模式是为了确保效率和性能，不会被大数据量压垮。


```python
import sqlite3

try:
    # 创建指向我们的数据文件的 SQLite 数据连接
    conn = sqlite3.connect('assets/flights.db')
    # 在这个连接上初始化一个 cursor
    cursor = conn.cursor()
    # 有几种办法可以把 cursor 定位到我们需要的数据集上，下面是最通用的办法，运行一条 SQL 语句
    cursor.execute('SELECT id, name, city, code FROM airports WHERE country="China";')
    # cursor 已经定位到上述 SELECT 语句定义的数据集的首行，我们可以用 cursor.fetchone() 函数一次取回一行
    # 每次取回的数据是一个 tuple，里面是 cursor 指向的行里我们指定的各列的值，fetch 之后 cursor 会把自己向后移动一行
    print(cursor.fetchone())
    print(cursor.fetchone())
    print(cursor.fetchone())
finally:
    # 打扫战场，关闭用过的 cursor 和 connection
    cursor.close()
    conn.close()
```

    ('3364', 'Capital Intl', 'Beijing', 'PEK')
    ('6817', 'Hongyuan Airfield', 'Hongyuan', None)
    ('3366', 'Dongshan', 'Hailar', 'HLD')


#### SQL 初见

我们终于碰到了 SQL，知道了它大概长什么样。上面的 SQL 语句从机场表中取出了所有中国的机场，然后输出了几个关键字段：

```SQL
SELECT id, name, city, code FROM airports WHERE country="China";
```

* `SELECT` 后面是要输出的列，可以有多个，用逗号 `,` 隔开，还可以用 `*` 代表所有列；
* `FROM` 后面是要访问的表；
* `WHERE` 后面是查询条件，符合条件的数据行才会被选出。

所以上面的 SQL 意思是：从 airports 表中取出 country 列为 “China” 的所有行，输出这些行的 id, name, city, code 这几列。

这句 *SQL* 语句只涉及到一张表，是最简单的情况，我们还可以在一条 *SQL* 中访问多张彼此关联的表，这时情况就复杂多了，这里不打算太深入讲 *SQL*，但可以举一个例子让你体会一下。

**问题：查询所有从上海浦东飞出的航班，了解分别有哪些国家的哪些航空公司有从浦东飞出的航班。**

我们的 `flights.db` 数据库里有三个表：
* `airports` 表对应“机场”这个实体类型，里面有 8000 多个机场的数据，包括机场所在的国家、城市，机场的名称、代码和唯一 id 等信息；
* `airlines` 表对应“航公公司”这个实体类型，里面有 6000 多个航空公司的数据，包括航公公司所属国家，公司名称和唯一 id 等信息；
* `routes` 表对应“航线”这个实体类型，里面有 67000 多条航线的数据，里面有航线起始机场代码，还有运营这条航线的航空公司的 id 等信息。

现在我们面临的问题中：
* **已知条件**是起始机场的城市（“Shanghai”）和机场名称（“Pudong”），这些信息在 `airports` 表中；
* **目标输出**是航线的起始机场，以及运营航线的航空公司所属国家及公司名称，这些信息分别在 `routes` 表和 `airlines` 表中。

经过思考，我们可以确认：主要要查询的是“航线”这个实体类型，但输入条件是“机场”这个实体类型的属性，而输出包含了“航空公司”这个实体类型的一些属性。所以我们主要从 `routes` 这张表中 `SELECT`，但需要把另外两张表“关联起来”。

表的关联，在关系模型中称为 `JOIN`，两张表发生关联需要指定一个条件，比如针对 `routes` 表中的这一行数据：

`(index=867,airline=3U,airline_id=4608,source=PVG,...)`

这一行里记录了航线的运营公司 airline_id=4608，这是一个外键，我们只要拿这个 `4608` 去 `airlines` 表中找，就能找到对应的航空公司信息。于是这两张表的关联条件可以写作：

`airlines.id = routes.airline_id`

下面就是我们完成任务的 SQL 语句：

```SQL
SELECT
	airlines.country,
	airlines.name,
	routes.source,
	routes.dest
FROM
	routes
	JOIN airports ON routes.source = airports.code
	JOIN airlines ON airlines.id = routes.airline_id
WHERE
	airports.city = "Shanghai"
	AND airports.name = "Pudong"
ORDER BY
	airlines.country,
	airlines.name
```

主框架还是 `SELECT` `FROM`，`FROM` 后面是主表 `routes`，差别在于增加了两个 `JOIN...ON` 子句，从而把 `airports`、`airlines` 与主表关联起来，关联条件写在 `ON` 后面。

关联之后，`SELECT` 后面的选取列就不限于 `routes` 表了，而是关联进来的表的列也可以选取输出，写法要指定 `表名.列名` 这样的格式，我们选择输出了 `airlines` 和 `routes` 表中的一共四个列。

`WHERE` 子句中同样也不限于指定主表的条件，我们在这里指定了 `airports` 表的两个条件。

最后，我们可以指定输出结果的排序方式，这里用航空公司的所属国家和公司名来排序。下一节我们会用这个 SQL 来做例子，就能看到输出的结果了。

SQL 语言不仅可以用 `SELECT` 语句查询数据返回结果数据集，还可以增加删除和修改数据，用的是 `INSERT`、`DELETE`、`UPDATE` 这些语句，下面会讲。

#### 用 pandas 加载数据

我们也可以用 pandas 来加载 SQLite 数据库，从而利用 pandas 强大的 `DataFrame` 对象。

诀窍在 `DataFrame` 提供的 `read_sql_query` 方法，这个方法接受两个参数，第一个是要执行的 SQL 语句，第二个是用于数据访问的数据连接对象。

下面我们用前一节的那个比较复杂的 SQL 作为例子，查询所有从上海浦东出发的航线，分别是哪些国家的哪些航空公司运营的。


```python
import sqlite3
import pandas as pd

sql = 'SELECT airlines.country,airlines.name,routes.source,routes.dest \
    FROM routes JOIN airports ON routes.source=airports.code JOIN airlines ON airlines.id=routes.airline_id \
    WHERE airports.city="Shanghai" AND airports.name="Pudong" ORDER BY airlines.country,airlines.name'
conn = sqlite3.connect('assets/flights.db')
df = pd.read_sql_query(sql, conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>country</th>
      <th>name</th>
      <th>source</th>
      <th>dest</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Albania</td>
      <td>Albanian Airlines</td>
      <td>PVG</td>
      <td>MLE</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Australia</td>
      <td>Qantas</td>
      <td>PVG</td>
      <td>MEL</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Australia</td>
      <td>Qantas</td>
      <td>PVG</td>
      <td>SYD</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Brunei</td>
      <td>Royal Brunei Airlines</td>
      <td>PVG</td>
      <td>BWN</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Cambodia</td>
      <td>Cambodia Angkor Air (K6)</td>
      <td>PVG</td>
      <td>PNH</td>
    </tr>
  </tbody>
</table>
</div>




```python
print(f'总计 {len(set(df["country"]))} 个国家的 {len(set(df["name"]))} 家航空公司有从浦东飞出的航线')
```

    总计 41 个国家的 65 家航空公司有从浦东飞出的航线


这里 `df` 是一个数据集对象（即 `DataFrame` 类的对象），`df["contry"]` 取出数据集中 `country` 这一整列（是一个 *iterable* 对象），然后用 `set()` 函数将其转换为集合，从而去掉了所有重复的国家，最后用 `len()` 函数就得到集合元素个数，也就是不重复的国家数目，这是使用 pandas 数据集时常用的方法，后面统计航空公司数目也是一样的方法。

最后，用完不要忘记关闭数据库：


```python
conn.close()
```

顺便说一句，pandas 可以方便的帮助我们把数据集转存为 CSV 或者 JSON 格式的文件，分别使用 `DataFrame` 的 `to_csv()` 和 `to_json()` 方法即可，具体可以参考[官方文档](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html)。

### MySQL 数据库

MySQL 是目前世界上应用最多的数据库，根据 [Datanyze 的统计](https://www.datanyze.com/market-share/databases/mysql-market-share)，MySQL 目前占超过 46% 的市场份额，第二和第三名分别是 PostgreSQL 和 Microsoft SQL Server，份额分别为 10.11% 和 9.28%，可见优势之巨大。

MySQL 从 RDBMS 的角度来说，历史悠久功能强大而完备，远非轻量级的 SQLite 能比，但是基本的使用套路和 SQLite 没有太大的区别，也是 *connection* 和 *cursor* 那一套。不过 MySQL 本身比 SQLite 复杂很多，不是一个数据文件加一个库就能搞定，必须安装 MySQL 服务和专属的客户端库才能使用，关于 MySQL 服务环境的准备可以参考[这篇指南](x5-mysql-setup.ipynb)。

如果配置环境不顺利，也不要紧，目前阶段从下面的例子中获得一些感性认识就够了。

在继续之前我们还需要安装最新的 MySQL Connector/Python：`pip install mysql-connector-python`，这是我们在 Python 程序中连接 MySQL 的客户端支持库，提供了连接 MySQL 服务访问数据的能力，相当于我们前面使用的 `sqlite3` 库。

#### 数据连接和访问

下面的部分假定本机已有安装好的 MySQL 服务，已准备好 `fifa19` 数据库，里面有我们从 [Kagger](https://www.kaggle.com/karangadiya/fifa19) 下载并导入的数据，游戏 FIFA19 里所有球员的详细数据，存放于名为 `players` 的表中；MySQL 中已创建用户 `learn` 拥有访问此 `fifa19` 数据库的权限。

> 如果按照[这篇指南](x5-mysql-setup.ipynb)安装和配置好了本地的 MySQL 数据库，那么可能在你的系统里这个数据库名叫 `demo`，下面代码中 `database = 'fifa19'` 需要改为 `database = 'demo'`。
> 
> 总之配置 MySQL 连接时，`host` `user` `passwd` `database` 这四个参数对应 MySQL 服务的地址、用户名、密码和数据库名，根据你的环境设置即可。


```python
import mysql.connector as mysql
import pandas as pd

conn = mysql.connect(
    host = 'localhost',
    user = 'learn',
    passwd = 'demo',
    database = 'fifa19'
)

sql = 'SELECT Name, Age, Nationality FROM players WHERE Club LIKE "%Barcelona%";'
```

创建好数据连接之后，和前面 SQLite 一样，既可以使用基本的 `cursor` 方法，也可以使用 pandas 提供的 `DataFrame` 对象来访问数据。


```python
cursor = conn.cursor(buffered=True) # 这是 MySQL Connector 的一个怪癖，必须在创建 cursor 时指定 buffered=True
cursor.execute(sql)
print(cursor.fetchmany(3))
cursor.close() # 用完就关是好习惯
```

    [('L. Messi', 31, 'Argentina'), ('L. Suárez', 31, 'Uruguay'), ('M. ter Stegen', 26, 'Germany')]





    True




```python
df = pd.read_sql(sql, conn)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Name</th>
      <th>Age</th>
      <th>Nationality</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>L. Messi</td>
      <td>31</td>
      <td>Argentina</td>
    </tr>
    <tr>
      <th>1</th>
      <td>L. Suárez</td>
      <td>31</td>
      <td>Uruguay</td>
    </tr>
    <tr>
      <th>2</th>
      <td>M. ter Stegen</td>
      <td>26</td>
      <td>Germany</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Sergio Busquets</td>
      <td>29</td>
      <td>Spain</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Coutinho</td>
      <td>26</td>
      <td>Brazil</td>
    </tr>
  </tbody>
</table>
</div>



从上面的代码可以看出，和前面 SQLite 的区别只在于导入了不同的库，创建 *connection* 的方法有些差别，后面都是一样的。

用完之后不要忘记关闭数据连接，和 SQLite 这样单用户的文件型数据库不一样，MySQL 是面向多用户使用的服务，打开一个数据连接，获取数据可能会某种程度的“锁住”数据资源，让别的操作无法进行，比如我们刚才的操作对 `players` 全表执行读操作，这会导致对这个表的数据结构的调整无法进行。所以在 MySQL 这样的多用户数据库环境下， 一定要做个好公民，只使用必要的数据资源，用完立刻释放：


```python
conn.close()
```

#### 数据事务和锁

前面我们的例子全都是读数据的例子，应用程序对数据只读不写是不可能的， 写数据又是什么样子呢？

类似 MySQL 这样的多用户数据库系统，通常都会有许多数据连接同时访问数据库（可能来自同一个应用也可能来自不同的应用），要处理好写数据是个非常有挑战的事情，设想几个场景：
1. 如果一个数据连接正在读取一行数据，另一个连接请求修改甚至删除这行数据，怎么办？
2. 如果一个数据连接正在修改一行数据，另一个连接也要修改这个数据怎么办？
3. 如果两个数据连接同时请求同一行数据，一个要读一个要写，先满足哪个？
4. 如果一个数据连接正在访问某行数据，另一个连接请求修改这行数据所在数据表的结构（比如删除一个列或者增加一个索引）怎么办？
5. 如果一个数据连接读取一个数，对其加一然后写回去，这个操作进行到一半另一个连接修改了这个数怎么办？

想想就头大对不对？

*RDBMS* 发展的几十年里，这些问题都有了确定的答案。*RDBMS* 的设计倾向是牺牲一些并发能力（同时访问数据的请求数量受到一定限制），但确保数据的完整性和一致性。目前主流的 *RDBMS* 系统都能满足绝大部分应用的并发需要，同时在几乎所有情况下都能保证数据完整和一致；我们之前也提到过，特别高并发的业务场景，经过业务设计和应用设计的优化仍然无法降低对数据库的并发读写需要的，就需要使用别的数据存储方案了，比如下面会提到的 Redis 这样的内存 *key-value* 存储。

在这个设计原则之上，*RDBMS* 的设计和实现者们设计了一系列机制来解决前面的五个问题（以及其他类似的问题），其中针对前面几个问题的主要机制是“锁”，针对最后一个的是“事务”。

所谓“**锁（*lock*）**”就是给一块数据临时打上一个标记，禁止其他请求访问，用完了再解锁。加锁和解锁都是由 *RDBMS* 自动进行的，不需要应用端做什么。也就是说，虽然整个数据库是多用户的，可以被很多应用共用，但是具体到某块数据，一次只能一个用户来，排队使用，这差不多就是锁的基本概念了。然后为了提高数据库性能，让尽可能多的事情可以同时做，而不是什么都需要排队，*RDBMS* 对锁的机制进行了很多的优化，各个厂商的数据库产品会有差别，下面讲的是一些一般性思路：
* 根据数据读写的特点区分出两种锁：共享锁（读锁）和排它锁（写锁）。当读一块数据的时候加共享锁，这时候其他请求依然可以读这块数据，但不能写；当写一块数据时加排他锁，其他请求不可以读写这块数据。这是非常合理的，而且带来了很大的性能提升，因为在数据库操作中，读操作远远多于写，共享锁确保了所有的读操作都可以并发进行。
* 根据数据访问的性质区分出两种不同粒度的锁：行级锁和表级锁，行锁一次锁一行数据，表里其他行不受限制，表锁一次锁整表，所有行都受限制。显然行锁能有更好的并发表现，但是行锁在数据库这边有更高的开销，加锁解锁都更慢，所以 *RDBMS* 对行锁和表锁都是有选择的使用在最适合的场合。

主流的 *RDBMS* 产品还有很多更加细致的锁设计，我们就不展开了，绝大部分问题 *RDBMS* 都处理的很好，少数情况会有专职的 DBA（数据库管理员，对 *DBMS* 的实现细节更加了解，专门负责维护和优化 DBMS 的技术人员）来处理。对于编写应用程序的程序员来说，最重要的是时刻牢记**数据库是高度共享的资源，在访问数据库时努力做个好公民，通过应用本身的设计优化保证每次数据库读写请求都足够快，并及时释放占用的资源**。

而“**事务**（*transaction*）”则是为了保证一组操作的完整性而引入的。

所谓事务就是一组操作，这组操作要么全部成功，要么一点也不做；如果做了一半失败了，*RDBMS* 会自动回滚（*rollback*）到事务开始前的状态。

和锁不一样，事务是应用开发者需要直接操作的概念，哪些操作属于同一个事务，是由具体业务决定的，所以事务的开始和结束必须由应用端指定。主流的编程语言和数据库访问工具都提供完备的手段来帮助我们管理事务——当然，是与 *RDBMS* 配合完成的。

需要使用事务的场合很多，比如甲借钱给乙，从数据库视角来说，就是甲的资金余额减去 X，然后乙的资金余额加上 X，这两个操作分属两个不同的数据行，这两个操作要么都成功（交易完成），要么都不做（交易失败，状态回滚）。再比如计数器，假设我们想统计用户的点击次数，我们在数据库里放一个计数器，每有用户点击就加一，那么用户点击时我们要做两件事，先取出计数器当前的值 N，然后把 N+1 写回去，这两个操作也必须连续完整执行，如果分开做，万一读出 N 之后别人对计数器做了加一操作，计数器已经变成 N+1 甚至更大的数了，我们再把 N+1  写回去，这个数就错了。

当我们分析完业务，确认某几件事需要打包处理，那么我们就可以在程序要开始一个事务，按照顺序执行我们需要的数据库操作，最后提交整个事务，*RDBMS* 就会完成剩下的工作，最后告诉我们事务成功还是回滚了，然后我们可以相应的做善后处理。只要我们标记了事务的开始和结束，*RDBMS* 会保证事务中的操作要么全部成功，要么全部回滚。

注意事务的维持是需要代价的，一个事务持续的时间越长，数据库就有越多的状态处在不确定中（成功还是回滚），也意味着越多数据被锁定无法访问，大大影响其他人使用这些数据。所以我们设计应用时应该尽可能减少对事务的依赖，只用在不得不用的场合，用的时候也努力让事务更简单、可以更快完成。

#### 用事务操作数据

下面我们简单示范在 Python 中操作数据的方法，以及如何使用 MySQL 的事务。继续使用 `demo` 数据库。


```python
import mysql.connector as mysql

conn = mysql.connect(
    host = 'localhost',
    user = 'learn',
    passwd = 'demo',
    database = 'demo'
)
```


```python
cursor = conn.cursor()
cursor.execute('CREATE TABLE users (id INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY, username VARCHAR(255), firstname VARCHAR(255), lastname VARCHAR(255))')
```

前面都是我们熟悉的套路，创建数据连接，获得一个 *cursor*，用这个 *cursor* 来执行 SQL 语句，上面的 `CREATE TABLE` 语句是创建表的 SQL，后面跟着表名，和括号括起来的表的列，可以看到我们要创建的表叫 `users`，包含四个列：整数（`INT`）类型的 `id`，变长字符串（`VARCHAR`）类型的 `username`、`firstname` 和 `lastname`；其中 `id` 这个列比较特殊，后面有很多附加属性，分别是：
* `NOT NULL`：不可为空；
* `AUTO_INCREMENT`：自增，也就是说这个列的值我们不用管，MySQL 会自己给它 1、2、3 这样的自动增加的正整数值，确保它的唯一性；
* `PRIMARY KEY`：该列是 `users` 表的主键。

上面的代码运行成功后 `demo` 数据库中就有了一个空的表 `users`，里面有四个我们定义好的列：`id`，`username`，`firstname` 和 `lastname`。

我们可以在 mysql 命令行中执行命令 `USE demo` 然后 `DESC users;` 来查看建好的表 `users` 的列定义，mysql 会返回：

```
mysql> USE demo
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> DESC users;
+-----------+--------------+------+-----+---------+----------------+
| Field     | Type         | Null | Key | Default | Extra          |
+-----------+--------------+------+-----+---------+----------------+
| id        | int(11)      | NO   | PRI | NULL    | auto_increment |
| username  | varchar(255) | YES  |     | NULL    |                |
| firstname | varchar(255) | YES  |     | NULL    |                |
| lastname  | varchar(255) | YES  |     | NULL    |                |
+-----------+--------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
```

下面我们开始往表中插入数据，使用的是 `INSERT` 这个 SQL 命令（语法并不难懂，很容易理解）：


```python
cursor.execute('INSERT INTO users (username, firstname, lastname) VALUES ("neo", "Neo", "Lee")')
```

这个时候如果我们在 `mysql` 命令行中输入 `SELECT * FROM users;` 会发现并没有任何数据行，插入数据的操作未被执行，是的，因为我们需要告诉数据库，“以上操作告一段落，请执行”，否则 `RDBMS` 会等待我们提交更多请求一起执行，这中间所有请求都会被视为一个完整的事务。要提交执行的命令给数据库，使用数据连接的 `commit()` 方法：


```python
conn.commit()
```

运行完上面的代码再确认，就有新数据了：

```
mysql> SELECT * FROM users;
+----+----------+-----------+----------+
| id | username | firstname | lastname |
+----+----------+-----------+----------+
|  1 | neo      | Neo       | Lee      |
+----+----------+-----------+----------+
1 row in set (0.00 sec)
```

注意我们插入的 `INSERT` SQL 语句中并没有指定 `id` 这个字段的值，但 MySQL 自动给了 `1` 这个值，这就是前面指定 `id` 为 `AUTO_INCREMENT` 的效果。

如果我们要一次插入多条记录呢？数据连接对象提供了叫做 `executemany()` 的方法：


```python
# SQL 语句中的 %s 是占位符，表示这三个地方将替换为三个字符串类型的值（会被自动加上引号）
sql = 'INSERT INTO users (username, firstname, lastname) VALUES (%s, %s, %s)'
# 下面的列表中每个元素是一个 tuple，tuple 内的值将用来替换上面 SQL 语句中的占位符，按照先后顺序
# 列表中有几个 tuple，上面的 SQL 就会运行几次
values = [
    ('trinity', 'Carrie-Anne', 'Moss'),
    ('morpheus', 'Laurence', 'Fishburne'),
    ('smith', 'Hugo', 'Weaving')
]

cursor.executemany(sql, values)
conn.commit()
```

再去数据库里看看，三条新的记录都加入了（自增字段 `id` 工作良好）：

```
mysql> SELECT * FROM users;
+----+----------+-------------+-----------+
| id | username | firstname   | lastname  |
+----+----------+-------------+-----------+
|  1 | neo      | Neo         | Lee       |
|  2 | trinity  | Carrie-Anne | Moss      |
|  3 | morpheus | Laurence    | Fishburne |
|  4 | smith    | Hugo        | Weaving   |
+----+----------+-------------+-----------+
4 rows in set (0.00 sec)
```

如果要修改已有的数据行，可以使用 SQL 命令 `UPDATE`：


```python
cursor.execute('UPDATE users SET firstname="Keanu", lastname="Reeves" WHERE username="neo"')
conn.commit()
```

`UPDATE` 命令的语法也很好懂，`UPDATE` 后面是我们要修改的表，`SET` 后面是我们要修改的列和对应的新值，`WHERE` 的条件决定修改哪些行，所以上面这一句的意思是：找出 `users` 表中 `username` 值为 “neo” 的数据行，将这些行（在我们的例子里只有一行，但可以有多行）中的 `firstname` 字段值改为 “Keanu”，`lastname` 字段值改为 “Reeves”。检查数据库表会发现第一行记录已被更新：

```
mysql> SELECT * FROM users;
+----+----------+-------------+-----------+
| id | username | firstname   | lastname  |
+----+----------+-------------+-----------+
|  1 | neo      | Keanu       | Reeves    |
|  2 | trinity  | Carrie-Anne | Moss      |
|  3 | morpheus | Laurence    | Fishburne |
|  4 | smith    | Hugo        | Weaving   |
+----+----------+-------------+-----------+
4 rows in set (0.00 sec)
```

通常系统的用户名都不能重复，但在我们定义的 `users` 表中，除了 `id` 列是主键不能重复，其他列并没有限制，我们可以现在增加对 `username` 列的限制。注意，目前表中 `username` 列确实没有重复数据，这很重要，如果不是这样，我们增加限制会失败。


```python
cursor.execute('ALTER TABLE users ADD CONSTRAINT constraint_username UNIQUE (username)')
conn.commit()
```

上面执行成功的话，我们再向 `users` 表中添加数据将受到限制，`username` 字段值不可与已有的值重复。下面两个操作中的第一个应该成功，但第二个会由于这个限制而失败，因为我们把两个打包一起提交，这个事务将会整体回滚：


```python
try:
    cursor.execute('INSERT INTO users (username, firstname, lastname) VALUES ("oracle", "Gloria", "Foster")')
    cursor.execute('INSERT INTO users (username, firstname, lastname) VALUES ("smith", "Robert", "Taylor")')
    conn.commit()
except:
    conn.rollback()
```

如果我们不使用 `try...except` 异常处理，第二个 `cursor.execute()` 调用会抛出一个 `IntegrityError` 异常：

`IntegrityError: 1062 (23000): Duplicate entry 'smith' for key 'constraint_username'`

这里我们用了一个相当长的章节初步了解了 MySQL 这样的工业级 *RDBMS*，以及我们可以怎么使用它。很多时候我们其实不需要这样自己通过 SQL 来操作数据库，使用类似 [SQLAlchemy](https://www.sqlalchemy.org/) 这样的工具是更安全和舒适的选择，SQLAlchemy 把所有数据库操作变成完全 Python 对象化的操作，我们建立好数据连接之后可以像操作 Python 的 `list`、`tuple`、`dict` 和自定义类型一样操作数据库里的关系模型，而 SQLAlchemy 会帮我们把这些操作“翻译”成 `RDBMS` 的标准语言——SQL。

但无论如何目前我们对底层真相有所了解，会对我们解决问题很有帮助，是终生收益的。甚至于很有可能若干年后你再也不碰 Python 了，但 SQL 却是你每天会碰到的。

### Redis

最后一节我们要学习和前面讲的都不同的一种东西，[Redis](https://redis.io/) 是一个开源的内存 *key-value* 数据存储，这句话要这么理解：
* Redis 和 MySQL 一样都是开源软件的伟大产物，因为底子优秀加上开源，它在很短的时间里得到了广泛的应用，反过来也加速了它的成长和成熟；
* Redis 是一个 *key-value* 数据存储，存在 Redis 中的数据并不是关系模型（表、行、列），而是类似 *dictionary* 那样的 *key-value* 对；
* Redis 是内存数据库，数据是存放在内存中的，所以有着闪电般的速度（数百倍于最快的 SSD 硬盘）。

当然 Redis 也提供了持久化方案来把数据写入硬盘以免断电后丢失，也可以在需要时用持久化数据在内存中重建整个数据库，但只在需要时才会这么做。这一特点决定了 Redis 和 MySQL 这样的传统 RDBMS 不一样，Redis 适合用在对性能有极端要求的场合，但可以接受对 *RDBMS* 一些高级功能的牺牲，比如事务、数据一致性约束等。

通常 Redis 不会用作一个应用的主数据存储，而会和 MySQL 这样的 *RDBMS* 配合使用。所有业务数据的主存储仍然是 *RDBMS*，里面的数据是可靠、一致的，然后把一些对性能影响最大的数据装载到 Redis 的内存数据库中，提供给应用最好的访问性能，应用可以在 Redis 中做很多高速计算和处理，而只在需要的时候才把数据写回 *RDBMS*。

现实世界的大问题通常需要架构师（*architect*）设计精细的方案来解决数据同步和并发性能之间的平衡。我们目前还无法理解大多数这里面的设计模式，所以这一节只简单示范一下在 Python 中如何使用 Redis 这样的新型内存存储。

> **架构师**（*architect*）是复杂软件系统中做出系统级方案设计和决策的人，绝大部分情况下架构师是软件开发者中经验和水平最高的，需要对业务和软件技术都有非常深刻全面的理解才能担当这一角色。

Redis 和 MySQL 一样，也是多用户的服务，不过安装和配置要简单一些，具体指引可以参考[这篇指南](x6-redis-setup.ipynb)。

Redis 用起来非常简单，基本上就是 Python 的 *dict* 的概念，事实上 Redis 的意思就是 *Remote Dictionary Service*。简单来说：
* 一个 Redis 服务可以有多个数据库，分别用 0、1、2 这样的数字编号，缺省是 0 号数据库，如果要切换可以用 `SELECT index` 命令来切换；
* 每个数据库里就是许多 *key-value* 对，其中的 *key* 永远是字符串 `string`；
* Redis 里的 *value* 可以是几种类型，最常用的是 `string`、`list`、`hash` 和 `set` 这几种类型，基本上对应 Python 里的 `str`、`list`、`dict` 和 `set` 类型，所以 Python 开发者用起来会非常顺手；
* 对这些 *key-value* 对的操作，基本就是 `GET`、`SET`、`DEL` 这些，所有 Redis 支持的命令可以参考 [官方文档](https://redis.io/commands)。

#### 使用 Redis 客户端

我们先来看看怎么用 Redis 自己的客户端程序来使用它的服务。下面我们都假定 Redis 服务已经在本地成功运行（缺省端口 6379）。

在命令行运行 `redis-cli` 就会进入到 Redis 的 *REPL* 提示符：

`127.0.0.1:6379> `

在提示符后面输入 Redis 命令就可以看到结果，随时可以用 `QUIT` 或者 `EXIT` 来退出 *REPL*。下面来看一些例子。

1. 切换到 1 号数据库，然后切回缺省 0 号数据库（留意提示符的变化）：

```redis
127.0.0.1:6379> select 1
OK
127.0.0.1:6379[1]> select 0
OK
127.0.0.1:6379> 
```

2. 用 `SET key value` 命令插入一些 *key-value* 数据，用 `KEYS *` 命令来列出所有 *keys*，用 `GET key` 命令来查询对应的 `value`：

```redis
127.0.0.1:6379> SET Bahamas Nassau
OK
127.0.0.1:6379> SET Croatia Zagreb
OK
127.0.0.1:6379> SET China Beijing
OK
127.0.0.1:6379> KEYS *
1) "China"
2) "Croatia"
3) "Bahamas"
127.0.0.1:6379> KEYS C*
1) "China"
2) "Croatia"
127.0.0.1:6379> GET Croatia
"Zagreb"
```

3. 用 `MSET` 和 `MGET` 来做批量操作，`MSET` 后面是一串 `key1 value1 key2 value2 key3 value3...` 这样依次排列的 `key-value` 们，而 `MGET` 后面是一串想取得 *value* 的 *keys*：

```redis
127.0.0.1:6379> MSET Lebanon Beirut Norway Oslo France Paris
OK
127.0.0.1:6379> MGET Lebanon Norway Bahamas
1) "Beirut"
2) "Oslo"
3) "Nassau"
```

4. 用 `EXISTS` 命令来判断某个 *key* 是否存在，存在返回 `1`，否则返回 `0`：

```redis
127.0.0.1:6379> EXISTS China
(integer) 1
127.0.0.1:6379> EXISTS Japan
(integer) 0
```

上面展示的都是 *key* 和 *value* 都是字符串的最简单模式，Redis 里的 *value* 还可以是 `list`、`hash` 和 `set` 这些类型，各有自己的 `SET`、`GET` 命令。下面以 `hash` 为例。

5. 用 `HSET` 和 `HGET` 来读写 `hash` 类型的 *value*：

```redis
127.0.0.1:6379> HSET redis url "https://redis.io/"
(integer) 1
127.0.0.1:6379> HSET redis name "Redis"
(integer) 1
127.0.0.1:6379> HSET redis github "https://github.com/antirez/redis"
(integer) 1
127.0.0.1:6379> HGET redis github
"https://github.com/antirez/redis"
127.0.0.1:6379> HGETALL redis
1) "url"
2) "https://redis.io/"
3) "name"
4) "Redis"
5) "github"
6) "https://github.com/antirez/redis"
```

上面这个例子处理的数据基本上和下面的 Python 代码等价：


```python
data = {
    'redis': {
        'name': 'Redis',
        'url': 'https://redis.io/',
        'github': 'https://github.com/antirez/redis'
    }
}

print(data['redis']['github'])
print(data)
```

    https://github.com/antirez/redis
    {'redis': {'name': 'Redis', 'url': 'https://redis.io/', 'github': 'https://github.com/antirez/redis'}}


`list` 和 `set` 类型的例子我们就不写了，有兴趣可以自行尝试。下面我们看看在 Python 里怎么使用 Redis。

#### 使用 Python 访问 

首先我们需要安装 Redis 的 Python 支持库：`pip install redis`。然后就可以在 Python 代码中使用 Redis 了。

首先创建 Redis 数据连接，注意下面的输入参数其实都可以省略，因为我们的环境里都是缺省值，写出来只是为了展示


```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)
```

然后我们就可以用这个 `r` 来做所有的事情了，就和我们在 REPL 中输入命令效果相当。


```python
r.mset({ '中国': '北京', '日本': '东京' })
```




    True



上面的代码就和在 REPL 中运行 `MSET ...` 一样。注意我们这里用了中文的 *key* 和 *value*，这是完全合法的，但是要注意，Redis 保存非 ASCII 字符会采用 UTF-8 编码，所以在使用时需要进行解码：


```python
r.get("Bahamas")
```




    b'Nassau'




```python
r.get("中国")
```




    b'\xe5\x8c\x97\xe4\xba\xac'




```python
r.get("中国").decode("utf-8")
```




    '北京'




```python
r.hgetall('redis')
```




    {b'url': b'https://redis.io/',
     b'name': b'Redis',
     b'github': b'https://github.com/antirez/redis'}



关于 Redis 目前了解到这样就差不多可以了，以后我们会在实际项目中继续学习。

## 小结

本章是关于数据库及相关概念的入门，算是打开了编程世界里一块大陆的传送门，至于这扇门后面到底有多少奇妙，还要在以后不断学习和体验。
* 理解持久化的基本概念；
* 理解关系模型的基本概念；
* 理解关系型数据库的基本概念；
* 掌握 Python 中使用 CSV 和 JSON 数据的基本方法；
* 掌握 Python 中使用 SQLite 和 MySQL 这样的关系型数据库的基本方法；
* 初步了解 Redis。
