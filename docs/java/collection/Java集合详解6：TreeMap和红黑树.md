# Table of Contents

  * [什么是红黑树](#什么是红黑树)
    * [定义](#定义)
    * [实践](#实践)
      * [红黑树操作](#红黑树操作)
        * [插入操作](#插入操作)
        * [删除操作](#删除操作)
      * [红黑树实现](#红黑树实现)
        * [插入](#插入)
        * [删除节点](#删除节点)
    * [3.总结](#3总结)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)

本文参考多篇优质技术博客，参考文章请在文末查看

《Java集合详解系列》是我在完成夯实Java基础篇的系列博客后准备开始整理的新系列文章。
为了更好地诠释知识点，形成体系文章，本系列文章整理了很多优质的博客内容，如有侵权请联系我，一定删除。

这些文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

本系列文章将整理于我的个人博客：

> www.how2playlife.com

## 什么是红黑树

首先，什么是红黑树呢？ 红黑树是一种“平衡的”二叉查找树，它是一种经典高效的算法，能够保证在最坏的情况下动态集合操作的时间为O（lgn）。红黑树每个节点包含5个域，分别为color,key,left,right和p。 color是在每个节点上增加的一个存储位表示节点的颜色，可以是RED或者BLACK。key为结点中的value值，left,right为该结点的左右孩子指针，没有的话为NIL，p是一个指针，是指向该节的父节点。如下图（来自维基百科）表示就是一颗红黑树，NIL为指向外结点的指针。（外结点视为没有key的结点）



       红黑树有什么性质呢？一般称为红黑性质，有以下五点：

     1）每个结点或者是红的或者是黑的；

     2）根结点是黑的；

     3）每个叶结点（NIL）是黑的；

     4）如果一个结点是红的，则它的两个孩子都是黑的；

     5）对每个结点，从该结点到其他其子孙结点的所有路径上包含相同数目的黑结点。

       为了后面的分析，我们还得知道以下知识点。

    （1）黑高度：从某个结点x出发（不包括该结点）到达一个叶结点的任意一条路径上，黑色结点的个数称为该结点x的黑高度。

    （2）一颗有n个内结点的红黑树的高度至多为2lg(n+1)。   （内结点视为红黑树中带关键字的结点）

    （3）包含n个内部节点的红黑树的高度是 O(log(n))。

### 定义

红黑树是特殊的二叉查找树，又名R-B树(RED-BLACK-TREE)，由于红黑树是特殊的二叉查找树，即红黑树具有了二叉查找树的特性，而且红黑树还具有以下特性：

*   **1.每个节点要么是黑色要么是红色**

*   **2.根节点是黑色**

*   **3.每个叶子节点是黑色，并且为空节点(还有另外一种说法就是，每个叶子结点都带有两个空的黑色结点（被称为黑哨兵），如果一个结点n的只有一个左孩子，那么n的右孩子是一个黑哨兵；如果结点n只有一个右孩子，那么n的左孩子是一个黑哨兵。)**

*   **4.如果一个节点是红色，则它的子节点必须是黑色**

*   **5.从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。**

有几点需要注意的是：

1.特性3中指定红黑树的每个叶子节点都是空节点，但是在Java实现中红黑树将使用null代表空节点，因此遍历红黑树时看不到黑色的叶子节点，反而见到的叶子节点是红色的

2.特性4保证了从根节点到叶子节点的最长路径的长度不会超过任何其他路径的两倍，例如黑色高度为3的红黑树，其最短路径(路径指的是根节点到叶子节点)是2(黑节点-黑节点-黑节点)，其最长路径为4(黑节点-红节点-黑节点-红节点-黑节点)。

### 实践

#### 红黑树操作

##### 插入操作

首先红黑树在插入节点的时，我们设定插入节点的颜色为**红色**,如果插入的是黑色节点，必然会违背特性5，即改变了红黑树的黑高度，如下插入红色结点又存在着几种情况：

1.**黑父**

如图所示，这种情况不会破坏红黑树的特性，即不需要任何处理





![](https://upload-images.jianshu.io/upload_images/4761309-5c5b2c2111526b40.png?imageMogr2/auto-orient/strip|imageView2/2/w/317/format/webp)






2.**红父**

当其父亲为红色时又会存在以下的情况

*   **红叔**

红叔的情况，其实相对来说比较简单的，如下图所示，只需要通过修改父、叔的颜色为黑色，祖的颜色为红色，而且回去递归的检查祖节点即可





![](https://upload-images.jianshu.io/upload_images/4761309-d03b6cc68cd297e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/582/format/webp)






*   **黑叔**

黑叔的情况有如下几种，这几种情况下是不能够通过修改颜色达到平衡的效果，因此会通过旋转的操作，红黑树种有两种旋转操作，左旋和右旋(现在存在的疑问，什么时候使用到左旋，什么时候使用到右旋)

*   Case 1:[先右旋，在改变颜色(根节点必须为黑色，其两个子节点为红色，叔节点不用改变)],如下图所示，注意省略黑哨兵节点

    

    

![](https://upload-images.jianshu.io/upload_images/4761309-be4bb2dee4bffd10.png?imageMogr2/auto-orient/strip|imageView2/2/w/870/format/webp)

    

    

*   Case 2:[先左旋变成Case1中的情况，再右旋，最后改变颜色(根节点必须为黑色，其两个子节点为红色，叔节点不用改变)],如下图所示，注意省略黑哨兵节点

    

    

![](https://upload-images.jianshu.io/upload_images/4761309-7eed01cd63266976.png?imageMogr2/auto-orient/strip|imageView2/2/w/751/format/webp)


    

*   Case 3:[先左旋，最后改变颜色(根节点必须为黑色，其两个子节点为红色，叔节点不用改变)],如下图所示，注意省略黑哨兵节点

    

    

![](https://upload-images.jianshu.io/upload_images/4761309-2885dd45740eb113.png?imageMogr2/auto-orient/strip|imageView2/2/w/831/format/webp)

    


    

*   Case 4:[先右旋变成Case 3的情况，再左旋，最后改变颜色(根节点必须为黑色，其两个子节点为红色，叔节点不用改变)],如下图所示，注意省略黑哨兵节点

    

    

![](https://upload-images.jianshu.io/upload_images/4761309-db86cb65433a632e.png?imageMogr2/auto-orient/strip|imageView2/2/w/754/format/webp)

    


    

以上就是红黑树新增节点所有可能的操作，下面会介绍红黑树中的删除操作

##### 删除操作

删除操作相比于插入操作情况更加复杂，删除一个节点可以大致分为三种情况：

*   1.删除的节点没有孩子节点，即当前节点为叶子节点，这种可以直接删除

*   2.删除的节点有一个孩子节点，这种需要删除当前节点，并使用其孩子节点顶替上来

*   3.删除的节点有两个孩子节点，这种需要先找到其后继节点(树中大于节点的最小的元素);然后将其后继节点的内容复制到该节点上，其后继节点就相当于该节点的替身， 需要注意的是其后继节点一定不会有两个孩子节点(这点应该很好理解，如果后继节点有左孩子节点，那么当前的后继节点肯定不是最小的，说明后继节点只能存在没有孩子节点或者只有一个右孩子节点)，即这样就将问题转换成为1,2中的方式。

在讲述修复操作之前，首先需要明白几点，

1.对于红黑树而言，单支节点的情况只有如下图所示的一种情况，即为当前节点为黑色，其孩子节点为红色,(1.假设当前节点为红色，其两个孩子节点必须为黑色，2.若有孙子节点，则必为黑色，导致黑子数量不等，而红黑树不平衡)





![](https://upload-images.jianshu.io/upload_images/4761309-f8b873df1b880922.png?imageMogr2/auto-orient/strip|imageView2/2/w/318/format/webp)






2.由于红黑树是特殊的二叉查找树，它的删除和二叉查找树类型，真正的删除点即为删除点A的中序遍历的后继(前继也可以)，通过红黑树的特性可知这个后继必然最多只能有一个孩子，其这个孩子节点必然是右孩子节点，从而为单支情况(即这个后继节点只能有一个红色孩子或没有孩子)

下面将详细介绍，在执行删除节点操作之后，将通过修复操作使得红黑树达到平衡的情况。

*   Case 1:被删除的节点为红色，则这节点必定为叶子节点(首先这里的被删除的节点指的是真正删除的节点，通过上文得知的真正删除的节点要么是节点本身，要么是其后继节点，若是节点本身则必须为叶子节点，不为叶子节点的话其会有左右孩子，则真正删除的是其右孩子树上的最小值，若是后继节点，也必须为叶子节点，若不是则其也会有左右孩子，从而和2中相违背)，这种情况下删除红色叶节点就可以了，不用进行其他的操作了。





![](https://upload-images.jianshu.io/upload_images/4761309-ff82fc5e72f98af8.png?imageMogr2/auto-orient/strip|imageView2/2/w/431/format/webp)




*   Case 2:被删除的节点是黑色，其子节点是红色，将其子节点顶替上来并改变其颜色为黑色，如下图所示





![](https://upload-images.jianshu.io/upload_images/4761309-66968f353c49fe50.png?imageMogr2/auto-orient/strip|imageView2/2/w/434/format/webp)



*   Case 3:被删除的节点是黑色，其子节点也是黑色，将其子节点顶替上来，变成了双黑的问题，此时有以下情况

    *   Case 1:新节点的兄弟节点为**红色**，此时若新节点在左边则做左旋操作，否则做右旋操作，之后再将其父节点颜色改变为红色，兄弟节点

        



![](https://upload-images.jianshu.io/upload_images/4761309-55eb7590905dbe38.png?imageMogr2/auto-orient/strip|imageView2/2/w/490/format/webp)







![](https://upload-images.jianshu.io/upload_images/4761309-75984ffc3773a040.png?imageMogr2/auto-orient/strip|imageView2/2/w/562/format/webp)

    



从图中可以看出，操作之后红黑树并未达到平衡状态，而是变成的**黑兄**的情况

*   Case 2:新节点的兄弟节点为**黑色**,此时可能有如下情况

    *   红父二黑侄：将父节点变成黑色，兄弟节点变成红色，新节点变成黑色即可,如下图所示

        

        

![](https://upload-images.jianshu.io/upload_images/4761309-19ad02906c67ff6b.png?imageMogr2/auto-orient/strip|imageView2/2/w/589/format/webp)



![](https://upload-images.jianshu.io/upload_images/4761309-ed648a6bf4224a10.png?imageMogr2/auto-orient/strip|imageView2/2/w/564/format/webp)



*   黑父二黑侄：将父节点变成新节点的颜色，新节点变成黑色，兄弟节点染成红色，还需要继续以父节点为判定点继续判断,如下图所示

    



![](https://upload-images.jianshu.io/upload_images/4761309-85819fe812e784f1.png?imageMogr2/auto-orient/strip|imageView2/2/w/603/format/webp)


![](https://upload-images.jianshu.io/upload_images/4761309-cc00764b1900a74e.png?imageMogr2/auto-orient/strip|imageView2/2/w/549/format/webp)



*   红侄：

情况一:新节点在右子树，红侄在兄弟节点左子树，此时的操作为右旋，并将兄弟节点变为父亲的颜色，父亲节点变为黑色，侄节点变为黑色，如下图所示





![](https://upload-images.jianshu.io/upload_images/4761309-62ecd431cc7a3e64.png?imageMogr2/auto-orient/strip|imageView2/2/w/895/format/webp)




情况二:新节点在右子树，红侄在兄弟节点右子树，此时的操作为先左旋，后右旋并将侄节点变为父亲的颜色，父节点变为黑色，如下图所示





![](https://upload-images.jianshu.io/upload_images/4761309-b5e0ada3d870dcf6.png?imageMogr2/auto-orient/strip|imageView2/2/w/879/format/webp)



情况三：新节点在左子树，红侄在兄弟节点左子树,此时的操作为先右旋在左旋并将侄节点变为父亲的颜色，父亲节点变为黑色，如下图所示





![](https://upload-images.jianshu.io/upload_images/4761309-bd3e2c1efdc7a147.png?imageMogr2/auto-orient/strip|imageView2/2/w/885/format/webp)




情况四：新节点在右子树，红侄在兄弟节点右子树,此时的操作为左旋，并将兄弟节点变为父节点的颜色，父亲节点变为黑色，侄节点变为黑色，如下图所示





![](https://upload-images.jianshu.io/upload_images/4761309-9f34c34f7d02da29.png?imageMogr2/auto-orient/strip|imageView2/2/w/879/format/webp)



#### 红黑树实现

如下是使用JAVA代码实现红黑树的过程，主要包括了插入、删除、左旋、右旋、遍历等操作

##### 插入

```
/* 插入一个节点
 * @param node
 */
private void insert(RBTreeNode<T> node){
    int cmp;
    RBTreeNode<T> root = this.rootNode;
    RBTreeNode<T> parent = null;

    //定位节点添加到哪个父节点下
    while(null != root){
        parent = root;
        cmp = node.key.compareTo(root.key);
        if (cmp < 0){
            root = root.left;
        } else {
            root = root.right;
        }
    }

    node.parent = parent;
    //表示当前没一个节点，那么就当新增的节点为根节点
    if (null == parent){
        this.rootNode = node;
    } else {
        //找出在当前父节点下新增节点的位置
        cmp = node.key.compareTo(parent.key);
        if (cmp < 0){
            parent.left = node;
        } else {
            parent.right = node;
        }
    }

    //设置插入节点的颜色为红色
    node.color = COLOR_RED;

    //修正为红黑树
    insertFixUp(node);
}

/**
 * 红黑树插入修正
 * @param node
 */
private void insertFixUp(RBTreeNode<T> node){
    RBTreeNode<T> parent,gparent;
    //节点的父节点存在并且为红色
    while( ((parent = getParent(node)) != null) && isRed(parent)){
        gparent = getParent(parent);

        //如果其祖父节点是空怎么处理
        // 若父节点是祖父节点的左孩子
        if(parent == gparent.left){
            RBTreeNode<T> uncle = gparent.right;
            if ((null != uncle) && isRed(uncle)){
                setColorBlack(uncle);
                setColorBlack(parent);
                setColorRed(gparent);
                node = gparent;
                continue;
            }

            if (parent.right == node){
                RBTreeNode<T> tmp;
                leftRotate(parent);
                tmp = parent;
                parent = node;
                node = tmp;
            }

            setColorBlack(parent);
            setColorRed(gparent);
            rightRotate(gparent);
        } else {
            RBTreeNode<T> uncle = gparent.left;
            if ((null != uncle) && isRed(uncle)){
                setColorBlack(uncle);
                setColorBlack(parent);
                setColorRed(gparent);
                node = gparent;
                continue;
            }

            if (parent.left == node){
                RBTreeNode<T> tmp;
                rightRotate(parent);
                tmp = parent;
                parent = node;
                node = tmp;
            }

            setColorBlack(parent);
            setColorRed(gparent);
            leftRotate(gparent);
        }
    }
    setColorBlack(this.rootNode);
}

```

插入节点的操作主要分为以下几步：

*   1.定位：即遍历整理红黑树，确定添加的位置，如上代码中insert方法中就是在找到添加的位置

*   2.修复：这也就是前面介绍的，添加元素后可能会使得红黑树不在满足其特性，这时候需要通过变色、旋转来调整红黑树，也就是如上代码中insertFixUp方法

##### 删除节点

如下为删除节点的代码

```
private void remove(RBTreeNode<T> node){
    RBTreeNode<T> child,parent;
    boolean color;
    //被删除节点左右孩子都不为空的情况
    if ((null != node.left) && (null != node.right)){

        //获取到被删除节点的后继节点
        RBTreeNode<T> replace = node;

        replace = replace.right;
        while(null != replace.left){
            replace = replace.left;
        }

        //node节点不是根节点
        if (null != getParent(node)){
            //node是左节点
            if (getParent(node).left == node){
                getParent(node).left = replace;
            } else {
                getParent(node).right = replace;
            }
        } else {
            this.rootNode = replace;
        }

        child = replace.right;
        parent = getParent(replace);
        color = getColor(replace);

        if (parent == node){
            parent = replace;
        } else {
            if (null != child){
                setParent(child,parent);
            }
            parent.left = child;

            replace.right = node.right;
            setParent(node.right, replace);
        }

        replace.parent = node.parent;
        replace.color = node.color;
        replace.left = node.left;
        node.left.parent = replace;
        if (color == COLOR_BLACK){
            removeFixUp(child,parent);
        }

        node = null;
        return;
    }

    if (null != node.left){
        child = node.left;
    } else {
        child = node.right;
    }

    parent = node.parent;
    color = node.color;
    if (null != child){
        child.parent = parent;
    }

    if (null != parent){
        if (parent.left == node){
            parent.left = child;
        } else {
            parent.right = child;
        }
    } else {
        this.rootNode = child;
    }

    if (color == COLOR_BLACK){
        removeFixUp(child, parent);
    }
    node = null;
}

```

```
/**
 * 删除修复
 * @param node
 * @param parent
 */
private void removeFixUp(RBTreeNode<T> node, RBTreeNode<T> parent){
    RBTreeNode<T> other;
    //node不为空且为黑色，并且不为根节点
    while ((null == node || isBlack(node)) && (node != this.rootNode) ){
        //node是父节点的左孩子
        if (node == parent.left){
            //获取到其右孩子
            other = parent.right;
            //node节点的兄弟节点是红色
            if (isRed(other)){
                setColorBlack(other);
                setColorRed(parent);
                leftRotate(parent);
                other = parent.right;
            }

            //node节点的兄弟节点是黑色，且兄弟节点的两个孩子节点也是黑色
            if ((other.left == null || isBlack(other.left)) &&
                    (other.right == null || isBlack(other.right))){
                setColorRed(other);
                node = parent;
                parent = getParent(node);
            } else {
                //node节点的兄弟节点是黑色，且兄弟节点的右孩子是红色
                if (null == other.right || isBlack(other.right)){
                    setColorBlack(other.left);
                    setColorRed(other);
                    rightRotate(other);
                    other = parent.right;
                }
                //node节点的兄弟节点是黑色，且兄弟节点的右孩子是红色，左孩子是任意颜色
                setColor(other, getColor(parent));
                setColorBlack(parent);
                setColorBlack(other.right);
                leftRotate(parent);
                node = this.rootNode;
                break;
            }
        } else {
            other = parent.left;
            if (isRed(other)){
                setColorBlack(other);
                setColorRed(parent);
                rightRotate(parent);
                other = parent.left;
            }

            if ((null == other.left || isBlack(other.left)) &&
                    (null == other.right || isBlack(other.right))){
                setColorRed(other);
                node = parent;
                parent = getParent(node);
            } else {
                if (null == other.left || isBlack(other.left)){
                    setColorBlack(other.right);
                    setColorRed(other);
                    leftRotate(other);
                    other = parent.left;
                }

                setColor(other,getColor(parent));
                setColorBlack(parent);
                setColorBlack(other.left);
                rightRotate(parent);
                node = this.rootNode;
                break;
            }
        }
    }
    if (node!=null)
        setColorBlack(node);
}

```

删除节点主要分为几种情况去做对应的处理：

*   1.删除节点,按照如下三种情况去删除节点
    *   1.真正删除的节点没有子节点
    *   2.真正删除的节点有一个子节点
    *   3.正在删除的节点有两个子节点
*   2.修复红黑树的特性，如代码中调用removeFixUp方法修复红黑树的特性。

### 3.总结

以上主要介绍了红黑树的一些特性，包括一些操作详细的解析了里面的过程，写的时间比较长，感觉确实比较难理清楚。后面会持续的理解更深入，若有存在问题的地方，请指正。



## 参考文章



[红黑树(五)之 Java的实现](https://link.jianshu.com/?t=http://www.cnblogs.com/skywang12345/p/3624343.html/)

[通过分析 JDK 源代码研究 TreeMap 红黑树算法实现](https://link.jianshu.com/?t=https://www.ibm.com/developerworks/cn/java/j-lo-tree/index.html?ca=drs-)

[红黑树](https://link.jianshu.com/?t=http://blog.csdn.net/eric491179912/article/details/6179908)

[（图解）红黑树的插入和删除](https://link.jianshu.com/?t=http://www.cnblogs.com/deliver/p/5392768.html)

[红黑树深入剖析及Java实现](https://link.jianshu.com/?t=https://zhuanlan.zhihu.com/p/24367771)

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)



​                     
