---
title: 服务树的数据结构设计及实现方案
date: 2017-12-01 09:53:27
toc: true
tags:
- 数据结构
- 服务树
---
在项目实际开发中经常会遇到目录树或者服务树的开发需求，当公司业务达到一定级别，为了方便管理服务集群，利用树形结构建立起了的一种服务组织关系，方便集群服务治理，以服务集群节点为管理单元，而不是某台机器，这就是服务树或者目录树的由来。而在尽量少的占用资源前提下，如何高效、灵活、简单的管理好这棵树，这便是本次设计所想要解决的主要问题。
<!-- more -->
这里是针对本人所在公司的实际业务需求而设计，尽可能的抽象出通用的数据结构供大家参考使用，希望能帮助到需要的人。
## 一、这棵树的需求
### 1. 需求说明  
+ 能够目录层级的展示整棵服务树；
+ 能够自由的添加/修改/删除除根节点之外的节点；
+ 每个节点能够挂载子节点与机器；
+ 每个机器可以挂载在多个子节点下；
+ 能够快速的查找出某节点下(包括其全部子节点或不包括其全部子节点)挂载的所有机器；
+ 能够快速的查找出某机器挂载在哪个/哪些节点下；  

### 2. 需求分析
+ 由于相同机器可以挂在同一个节点上，所以实际上是一个有向图结构，但为了实现方便，可以假设这种情况是复制了一个资产信息，从而将数据结构关系变为一颗M叉树；
+ 这棵树是自上而下生成的，初始化只有一个Root节点；
+ 为保持统一，目录下无资产信息的，假设其下挂有一个虚拟资产叶子节点使其成为非叶子节点
+ 假设层级节点是非叶子节点，所有资产信息是叶子节点，则非叶子节点有唯一父节点；

## 二、数据结构及数据库设计
### 1. 数据结构设计  
采用M叉树设计，每一个目录节点相当于一个非叶子节点，每个一个资产相当于一个叶子节点。  
在保存这颗M叉树时采用[静态Huffman编码](https://baike.baidu.com/item/%E5%93%88%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81/1719730?fr=aladdin#4_1)，每个节点在保存自身信息的同时，还要保存自己父节点的Huffman编码ID。  
**为保证查询数据的方便，在此定义Root节点的父节点就是它自己。**  
<div align=center>
![](/img/M_tree.jpg)  
**一棵采用前缀编码的树**
</div>

### 2. 数据库设计  
+ 树非叶子节点表  
<div align=center>
![](/img/tree_node.png)
</div>
+ 树叶子节点表  
<div align=center>
![](/img/tree_leaf.png)  
</div>

**注意表中添加的索引**

## 三、算法说明
### 1. 渲染整棵树目录  
采用树的层级遍历方式，递归实现，每次添加一层信息，递归完成后返回。  
采用层级遍历的好处还有一点，因为实际使用中资产树的深度并不会很深，一般公司目录级别都不会超过5层，所以就算不要缓存每次动态加载也会很快。
### 2. 增删改节点  
在`tree_node表`里增删改记录即可。
### 3. 增删改资产挂载信息  
在`tree_leaf表`里增删改记录即可。
### 4. 查找某个节点下所有的资产信息  
+ 包含其所有子节点的资产信息  
以查询的节点的`huffman_id`为查询条件，查询`tree_node表`里parent_id`前缀为改ID的节点及它自身，随即获取该节点下的所有子树包括它自己的所有节点，然后去`tree_leaf表`查找所有节点下挂载的资产信息ID即可。  
    **注：这里可以采用join方式连接查询语句，可以避免使用in函数而使索引不生效的情况。**
+ 不包含其所有子节点的资产信息  
直接在`tree_leaf表`里查询对应`huffman_id`的资产信息即可。

### 5. 查找某个资产信息所挂载的节点  
在`tree_leaf表`里查找对应资产挂载点的`huffman_id`后再去`tree_node表`里查找即可。或者使用同样JOIN。

## 四、实际验证
### 1. 验证环境  
+ 硬件环境：`Mysql5.6 CentOS6.5 2核4G 50G云硬盘 最大读IOPS约3000-4000`  
![](/img/hard_info.png)  
![](/img/mysql_info.png)  
+ 数据量：`tree_node表 10000条记录，tree_leaf表100000条记录，host_assets表100000条记录`
![](/img/tree_node_num.png)  ![](/img/tree_leaf_num.png)  ![](/img/host_assets_num.png)  
+ 初始化数据：  
`tree_node表`  
![](/img/tree_node_demo.png)   
`tree_leaf表`  
![](/img/tree_leaf_demo.png)  

### 2. 验证结果  
渲染目录及增删改节点和资产挂载信息就是一般的数据库增删改操作，所以在此就不在赘述，这里实际验证就只验证查的效率即可。  
+ 查找某个节点下所有的节点信息(以最大的根节点为例)   
result:
![](/img/explain_sql.png)  
explain:
![](/img/result.png)  
sql:

``` 
select * from tree_leaf   
    join tree_node 
        on tree_leaf.node_huffman_id = tree_node.huffman_id 
    where tree_node.parent_id like "1%";
```
        
+ 查找某个资产信息所挂载的节点  
result:
![](/img/result_2.png) 
explain:
![](/img/explain_2.png)   
sql:  

``` 
select * from tree_node 
    join tree_leaf 
        on tree_node.huffman_id = tree_leaf.node_huffman_id 
    where host_sn = "host_1000";
```

## 五、代码示例
``` python
#import packages

class tree_demo(object):

    #init your db session
    #db = SQLAlchemy()

    #初始化
    def __init__(self):
        pass

    #获取节点信息
    def get_nodes_by_id(self, huffman_id = 1):
        try:
            node_record = db.session.query(tree_node).filter(tree_node.huffman_id == huffman_id).first()
            return node_record
        except Exception as e:
            return e

    #获取机器挂载的节点信息
    def get_nodes_by_hostsn(self, host_sn):
        try:
            node_records = db.session.query(tree_node.name, tree_node.huffman_id, tree_node.parent_id).join(tree_leaf, 
                tree_leaf.node_huffman_id == tree_node.huffman_id).filter(tree_leaf.host_sn == host_sn).all()
            return node_records
        except Exception as e:
            return e

    #获取该节点下包括其子节点下的所有资产信息
    def get_hostsn_with_child(self, huffman_id = 1):
        try:
            key = '%s%%' % huffman_id
            hostsn_records = db.session.query(tree_leaf.host_sn).join(tree_node, 
                tree_leaf.node_huffman_id == tree_node.huffman_id).filter(
                    tree_node.parent_id.like(key) | tree_node.huffman_id == huffman_id).all()
            return hostsn_records
        except Exception as e:
            return e

    #获取该节点下但不包括其子节点下的资产信息
    def get_hostsn_without_child(self, huffman_id = 1):
        try:
            key = '%s%%' % huffman_id
            hostsn_records = db.session.query(tree_leaf.host_sn).filter(tree_leaf.node_huffman_id == huffman_id).all()
            return hostsn_records
        except Exception as e:
            return e

    #渲染整棵树(广度优先遍历)
    def get_tree(self, huffman_id = 1):
        try:
            pass
        except Exception as e:
            return e

    #新增节点
    def add_node(self, parent_id, name):
        try:
            new_node = tree_node()
            new_node.parent_id = parent_id
            new_node.name = name
            db.session.commit()
            new_node.huffman_id = parent_id + str(new_node.id)
            db.session.commit()
            return new_node
        except Exception as e:
            db.session.rollback()
            return e

    #删除节点
    def delete_node(self, huffman_id):
        try:
            num_rows_deleted = db.session.query(tree_node).filter(tree_node.huffman_id == huffman_id).delete()
            db.session.commit()
            return num_rows_deleted
        except Exception as e:
            db.session.rollback()
            return e

    #更新节点名称
    def update_node_name(self, huffman_id, name):
        try:
            num_rows_updated = db.session.query(tree_node).filter(tree_node.huffman_id == huffman_id).update({"name":name})
            db.session.commit()
            return num_rows_updated
        except Exception as e:
            db.session.rollback()
            return e

    #新增机器节点挂载信息
    def add_leaf(self, huffman_id, host_sn):
        try:
            new_leaf = tree_leaf()
            new_leaf.node_huffman_id = huffman_id
            new_leaf.host_sn = host_sn
            db.session.commit()
            return new_leaf
        except Exception as e:
            return e

    #删除机器节点挂在信息
    def delete_leaf(self,huffman_id, host_sn):
        try:
            num_rows_deleted = db.session.query(tree_leaf).filter(tree_leaf.node_huffman_id == huffman_id).filter(
                tree_leaf.host_sn == host_sn).delete()
            db.session.commit()
            return num_rows_deleted
        except Exception as e:
            db.session.rollback()
            return e

    #格式化整棵树
    def truncate_tree(self):
        try:
            db.session.query(tree_node).filter(tree_node.id > 1).delete()
            db.session.query(tree_leaf).delete()
            db.session.commit()
            return "Initialized"
        except Exception as e:
            db.session.rollback()
            return e

class tree_node(db.Model):
    #please insert (1, "ROOT", 1, 1, now()) into your tree_node table for initializtion
    __tablename__ = 'tree_node'

    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(255), nullable=False)
    huffman_id = db.Column(db.String(50), nullable=False, default = "")
    parent_id = db.Column(db.String(50), nullable=False, index=True)
    update_time = db.Column(db.TIMESTAMP, nullable=False, server_default = func.now())        

class tree_leaf(db.Model):
    __tablename__ = 'tree_leaf'

    id = db.Column(db.Integer, primary_key=True)
    node_huffman_id = db.Column(db.String(50), nullable=False, index=True)
    host_sn = db.Column(db.String(255), nullable=False)
    update_time = db.Column(db.TIMESTAMP, nullable=False, server_default = func.now()) 
```
