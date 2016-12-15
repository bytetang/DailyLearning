[TOC]


我一直面临一个容易忽略的问题，阅读和积累的知识过于散乱。最近新总结一个阅读方法叫做主题阅读。
基本的做法比较简单，主要是要培养成习惯。

> 提炼知识点主题，随之阅读大量相关题材。不要分散阅读。

举个栗子来说，想研究下Hashmap的原理，大体知道其put，get和hash碰撞问题。涉及到hash算法、链表存取、红黑二叉树。那么就随这几条主线使用集中的时间来阅读和理解。这样的好处，能在最后的时间把知识点串联，效率高而且方便查阅。

为了承接HashMap原理的分析，先来了解下红黑二叉树。

##特点##

首先红黑二叉树是一个特殊的查找二叉树，所以包含二叉树的特征：任意一个节点所包含的键值，大于等于左孩子的键值，小于等于右孩子的键值。

- 每个节点为黑色或者红色
- 根节点为黑色
- 空的叶子节点是黑色
- 如果一个节点是红色的，那么他的子节点一定是黑色的
- 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点

###描述节点###
```java
public class RBTNode<T extends Comparable<T>> {
        boolean color;//节点颜色
        T key;
        RBTNode<T> left;//左节点
        RBTNode<T> right;//右节点
        RBTNode<T> parent;//父节点

        public RBTNode(boolean color, T key, RBTNode<T> left, RBTNode<T> right, RBTNode<T> parent) {
            this.color = color;
            this.key = key;
            this.left = left;
            this.right = right;
            this.parent = parent;
        }
    }
```
###红黑树左旋、右旋算法###

 通过示意图来看其过程（对x进行左旋）
     
       px                          px
       |                           |
       x                           y
      / \    leftRotate           / \
     lx  y   ------------>       x   ry
        / \                     / \
        ly ry                  lx ly

其过程就是把x的变为y的左节点（左旋）

代码分析：
```java
  RBTNode<T> y = x.right;
        x.right = y.left;//y的左子树设为x的右子数
        if (y.left != null){
            y.left.parent = x;//解除y左子树与y的关系,建立与x的关系
        }

        y.parent = x.parent;//解除y与parent的关系,与x的parent建立关系

        if (x.parent == null){//x是根节点
            this.mRoot = y;
        }else{//设置y在x父节点的左节点or有右节点(取代原x的在其父节点的位置)
            if (x.parent.left == x){
                x.parent.left = y;
            }else{
                x.parent.right = y;
            }
        }

        y.left = x;//x设为y的左节点
        x.parent = y;//解除x与其父节点关联,让其父节点和y关联.(前提完成上述操作x的关系已经全部给了y)
```
好吧，这个代码我写了好几遍，然后右旋的逻辑就呼之欲出了。讲y节点变成x的右节点（右旋）<br>
示意图：

         py                              py
         |                               |
         y                               x
        / \      rightRotate            / \
       x   ry  -------------->         lx  y
      / \                                 / \
     lx rx                               rx  ry

Java代码
 
```java
private void rightRotate(RBTNode<T> y){
        RBTNode<T> x = y.left;
        y.left = x.right;

        if (x.right != null){
            x.right.parent = y;
        }

        x.parent = y.parent;

        if (y.parent == null){
            this.mRoot = x;
        }else{
            if (y.parent.left == y){
                y.parent.left = x;
            }else{
                y.parent.right = x;
            }
        }

        x.right = y;
        y.parent = x;
    }
```
左旋右旋理解了，那么他有什么用呢。
> 左旋右旋一般应用在红黑树的调整。使二叉查找树变成标准的红黑树。

### 红黑树插入 ###
插入过程：

把红黑树当做二叉查找树，把新节点插入二叉查找树
对二叉查找树进行调整，使其变成标准的红黑树。


二叉查找树插入新节点的过程：<br>
1）若B树是空的，则直接将新节点作为根节点插入。<br>
2）x等于b的根节点的值则直接返回 <br>
3）若x小于b根节点值，x则插入作为b的左子树。反之插入作为b的右子树。<br>

转换成代码：
```java
public void insetTree(RBTNode<T> node){
        RBTNode<T> y = null;//记录插入的位置节点
        RBTNode<T> x = this.mRoot;//从根节点开始查找
        while (x!=null){
            y = x;
            if (node.key.compareTo(x.key)<0){//查找node节点的位置(该插入到哪个节点下面)
                x = x.left;
            }else{
                x = x.right;
            }
        }

        if (y!=null){
            if (node.key.compareTo(y.key)<0){
                y.left = node;//插入到左子树
            }else{
                y.right = node;//插入到右子树
            }
        }
        
        node.color = RED;
        ...
    }
```
调整成红黑树：
插入的节点可能会破坏掉红黑二叉树的结构，使其不能满足条件：<font color="red">如果一个节点是红色的，那么他的子节点一定是黑色的.</font>
那么就需要通过左右旋算法去完成调整。

