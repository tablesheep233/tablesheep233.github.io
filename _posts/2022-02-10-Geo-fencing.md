---
layout:     post
title:      "Geo-fencing"
subtitle:   " \"地理围栏算法笔记\""
author:     "tablesheep"
date:       2022-02-10 20:00:00
header-style: text
catalog: true
tags:
    - Java
    - 算法
---


最近项目遇到了一个需求，判断一个水质监测点是否在某个集水区（不规则多边形）内，对此进行了很多学习和研究，发现其实很多功能都跟这个类似，比如外卖、快递的服务区域，共享单车的停放区域等。这种需求的核心问题就是判断某个点是否在某个多边形的内部（**point-in-polygon**），根据wiki定义就是**地理围栏**（Geo-fencing）。



## 如何判断点在多边形内

### 射线法

从给定点开始，沿某一方向做射线（为了方便一般选择沿x轴方向），统计射线与每一条边的交点个数，若为奇数则在多边形内部，若为偶数则在外部，射线法时间复杂度为O(n)，n为多边形边数。

![射线法](/img/post/point-in-polygon-ray.png)

射线法情况说明：

1. A、G：点A、G的情况由于边跟射线平行，所以要判断点是否在边上（即x坐标的范围）

2. B、C：点B、C属于正常情况，不过多赘述

3. D、F：D点射线交点为3个，是奇数，按前面讲的它应该在多边形内部，但肉眼可见的不是；类似的还有F点，它射线交点只有2个，却是在内部。这两种情况应该怎么处理呢？

   射线与边相交，则点的y坐标必定处于边两个端点[ymin，ymax]范围内，而多边形的一个角必定关联着两条边。

   看一下点D的情况，记点C纵坐标yD，O、P、Q纵坐标yO、yP、yQ，则有yC属于[min(yO，yP)，max(yO，yP)]以及属于[min(yP，yQ)，max(yP，yQ)]，这里我们取了双闭区间，那么就会导致计算交点时点P的位置会计算两次，最终交点数量为3，不影响结果。如果我们取双开区间，则会忽略点P，那么最终交点数量为1，也符合预期，类似的点F也一样。

   不过这样取双开双闭是不是就可以了呢，并不是，看一下点E，对于双开双闭分别是0、2，不符合结果。必须要让点E只计算一次，而点D、F要计算两次或者不计算才可以，这其实就是一个临界值问题。

   其实只要取单开区间就好了，对于纵坐标y，跟边的关系，应该有y属于(ymin，ymax]（或者属于[ymin，ymax)）。这种情况下点D、E、F交点数分别是0、1、3，符合我们的预期

代码如下：

```java
/**
 * 返回一个点是否在一个多边形区域内（包括多边形边界上）
 * 采用单边射线法，向右做水平射线，若交点为奇数则在多边形内
 */
public boolean isPolygonContainsPoint(double[] polyX, double[] polyY, int size, double x, double y) {
    int intersectCount = 0;
    for (int i = 0; i < size; i++) {
        int j = (i + 1) % size;
        double x1 = polyX[i];
        double y1 = polyY[i];
        double x2 = polyX[j];
        double y2 = polyY[j];
        // 取多边形任意一个边,做点的水平延长线,求解与当前边的交点个数
        if (y1 == y2) { // 边是水平线段,要么没有交点,要么有无限个交点
            if ((y == y1) && (x >= Math.min(x1, x2) && x <= Math.max(x1, x2))) {
                // 点在边上
                return true;
            }
        }
        // 没有交点(点在边下 或 点在边上 )
        if (y < Math.min(y1, y2) || y >= Math.max(y1, y2)) {
            continue;
        }
        // 求解交点（相似三角形）
        double intersect = (y - y1) * (x2 - x1) / (y2 - y1) + x1;
        if (intersect == x) {
            //点在边上
            return true;
        }
        if (intersect > x) {
            // 右相交
            intersectCount++;
        }
    }
    return intersectCount % 2 == 1;
}
```





## 优化

### 最小外包矩形初筛

对于需要判断的点特别多的情况，可以使用最小外包矩形快速筛选一遍，过滤出可能符合的点。

最小外包矩形 **MBR(Minimal Bounding Rectangle)**顾名思义，就是一个最小的可以把多边形包起来的矩形。我们可以算出多边形右上角以及左下角的坐标，这样就形成外包矩形了。

代码：

```java
private void initOuterRectangle(double[] polyX, double[] polyY, int size) {
    maxX = polyX[0];
    maxY = polyY[0];
    minX = maxX;
    minY = maxY;
    for (int i = 1; i < size; i++) {
        double x = polyX[i];
        double y = polyY[i];
        if (maxX < x) {
            maxX = x;
        }
        if (maxY < y) {
            maxY = y;
        }
        if (minX > x) {
            minX = x;
        }
        if (minY > y) {
            minY = y;
        }
    }
}


private boolean isInOuterRectangle(double x, double y) {
    return x >= minX && x <= maxX && y >= minY && y <= maxY;
}
```



### RTree

**RTree**定义（来自wiki）：

R树的核心思想是聚合距离相近的节点并在树结构的上一层将其表示为这些节点的[最小外接矩形](https://zh.wikipedia.org/wiki/最小外接矩形)，这个最小外接矩形就成为上一层的一个节点。R树的“R”代表“Rectangle（矩形）”。因为所有节点都在它们的最小外接矩形中，所以跟某个矩形不相交的查询就一定跟这个矩形中的所有节点都不相交。叶子节点上的每个矩形都代表一个对象，节点都是对象的聚合，并且越往上层聚合的对象就越多。也可以把每一层看做是对数据集的近似，叶子节点层是最细粒度的近似，与数据集相似度100%，越往上层越粗糙。

个人理解：将所有的多边形都用 **MBR**包住，每个多边形的**MBR**是叶子节点，相近的几个**MBR**构成叶子节点的父节点，最终所有的**MBR**会构成根节点，这就是**RTree**。

对于多边形数量较多的判断，我们可以使用**RTree**优化，类似的对于多边形的边很多的情况也可以使用**RTree**优化，将多边形的边都构成**MBR**子节点，相近的边用**MBR**包起构成父节点，最终也构成一棵树。

代码：
// todo φ(゜▽゜*)♪ 





## References

- <https://www.geeksforgeeks.org/how-to-check-if-a-given-point-lies-inside-a-polygon/>
- <https://www.cnblogs.com/lbser/p/4471742.html>
- <http://alienryderflex.com/polygon/>
- <https://www.jianshu.com/p/3e3465ad1641>
- <https://zhuanlan.zhihu.com/p/362045520>
- <https://blog.csdn.net/enweitech/article/details/80654420>
- <https://en.wikipedia.org/wiki/R-tree>
- <https://blog.csdn.net/MongChia1993/article/details/69941783>

