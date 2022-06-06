---
title: SQL顺序索引-B+树
summary: 数据库系统索引
date:       2019-10-14
tags: ['Data Base']
---

本文介绍B+树索引的特点及遍历，插入，删除等基本操作

# B+树索引

B+树索引结构是在数据插入和删除情况下仍能保持其执行效率的最广泛的索引结构之一。B+树采用平衡树结构，其中树根到树叶的每条路径长度相同。  
B+树结构会增加文件插入及删除的性能开销，同时会增加空间开销。考虑到对更新频率较高的文件来说，这种开销也是可以接受的，因为这样会减少文件重组的代价。  

## B+树的特点

B+树是一种多级索引，但是其结构不同于多级索引结构文件，它最多包含$n-1$ 个搜索码值$K_1，K_2，...，K_{n-1}$，以及$n$个指针$P_1，P_2，...，P_n$

![nodeB_.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856501350_9215.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

简单概括下 B+ 树的三个特点：

![ds_bplus_tree1 _2_.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856502416_4982.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

1. 关键字数和子树相同  
2. 非叶子节点仅用作索引，它的关键字和子节点有重复元素
3. 叶子节点用指针连在一起  

首先第一点不用特别介绍了，在 B 树中，节点的关键字用于在查询时确定查询区间，因此关键字数比子树数少一；而在 B+ 树中，节点的**关键字代表子树的最大值**，因此**关键字数等于子树数**。

第二点，除叶子节点外的所有节点的关键字，都在它的下一级子树中同样存在，最后所有数据都存储在叶子节点中。

根节点的最大关键字其实就表示整个 B+ 树的最大元素。

第三点，叶子节点包含了全部的数据，并且按顺序排列，B+ 树使用一个**链表**将它们排列起来，这样在查询时效率更快。

由于 B+ 树的中间节点不含有实际数据，只有子树的最大数据和子树指针，因此磁盘页中可以容纳更多节点元素，也就是说同样数据情况下，B+ 树会 B 树更加“矮胖”，因此查询效率更快。

B+ 树的查找必会查到叶子节点，更加稳定。

有时候需要查询某个范围内的数据，由于 B+ 树的叶子节点是一个有序链表，只需在叶子节点上遍历即可，不用像 B 树那样挨个中序遍历比较大小。

B+ 树的三个优点：

1. 层级更低，IO 次数更少
2. 每次都需要查询到叶子节点，查询性能稳定
3. 叶子节点形成有序链表，范围查询方便

## B树的阶及高度

(英语对应order)定义是不统一的：
> Unfortunately, the literature on B-trees is not uniform in its terminology (Folk & Zoellick 1992, p. 362).  
>- Bayer & McCreight (1972), Comer (1979), and others define the **order of B-tree as the minimum number of keys** in a non-root node.  
> - Folk & Zoellick (1992) points out that terminology is *ambiguous* because the maximum number of keys is not clear. An order 3 B-tree might hold a maximum of 6 keys or a maximum of 7 keys.  
> - Knuth (1998, p. 483) avoids the problem by defining the **order to be maximum number of children** (which is one more than the maximum number of keys).

国内的数据结构教材一般是按照Knuth定义,即“阶”定义为一个节点的子节点数目的最大值。
> 树中每个非叶节点有$\lceil{n/2} \rceil $ ~ $n$  个子女，其中n对于特定的树是固定的。 -- [《数据库系统概念》:作者: （美）Abraham Silberschatz / （美）Henry F.Korth / （美）S.Sudarshan](https://book.douban.com/subject/10548379/)  

**我们按照Bayer & McCreight定义：**
> Soit n le nombre de clés contenues dans un nœud  假设节点中含n个指针  
>Un arbre-B+ est d’ordre d si  一棵d阶B+树为:  
>- Pour un nœud intermédiaire et une feuille : d ≤ n ≤ 2d   每个内部节点/叶子节点含d ~ 2d个指针 
>- Pour la racine: 1 ≤ n ≤ 2d  对于根节点，含1 ~ 2d 个指针   

> Hauteur minimum quand l’arbre est plein : 当树满时的最大高度  
>h = $\log_{2d+1}(N) +1 $
>- 2d clés par nœud  每个节点2d个指针
>- N feuilles (il y a donc $N \times 2d $ clés dans la table correspondante) 
N个叶子节点，因此共有 $N \times 2d $个指针

> Hauteur maximum quand les nœuds sont le plus vides
possible : 当树为最不满时的最大高度   
>h = $\log_{d+1}(N) +2 $  
>- d clés par nœud, 1 pour la racine 每个节点d个指针，根节点仅有1个
>- 2N feuilles ($2N \times d $ clés dans la table correspondante) 共2N个叶子节点，因此共有$2N \times d $个指针
>- En pratique d est assez grand, donc h reste raisonnable (4 pour une relation « normale ») 实际情况下d阶会很大，而h通常为4

![arbre ordre 2.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856503683_7025.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

## B+树遍历

![parcours-arbre-b.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856504731_3112.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

搜索数字43：
1. $43<50$,指针来到Bloc2
2. $40<43<45$,指针来到Bloc1
3. 43∈Bloc1

## B+树插入

1. 若为空树，创建一个叶子结点，然后将记录插入其中，此时这个叶子结点也是根结点，插入操作结束。

2. 针对叶子类型结点：根据key值找到叶子结点，向这个叶子结点插入记录。插入后，若当前结点key的个数小于等于m-1，则插入结束。否则将这个叶子结点分裂成左右两个叶子结点，左叶子结点包含前m/2个记录，右结点包含剩下的记录，将第m/2+1个记录的key进位到父结点中（父结点一定是索引类型结点），进位到父结点的key左孩子指针向左结点,右孩子指针向右结点。将当前结点的指针指向父结点，然后执行第3步。

3. 针对索引类型结点：若当前结点key的个数小于等于2d，则插入结束。否则，将这个索引类型结点分裂成两个索引结点，左索引结点包含前d个key，右结点包含d+1个key，将第d+1个key进位到父结点中，进位到父结点的key左孩子指向左结点, 进位到父结点的key右孩子指向右结点。将当前结点的指针指向父结点，然后重复第3步。

下面是一颗d=2 B树的插入过程，2阶B树的结点最少2个key，最多4个key。

空树中插入5  

![insertion1.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856505304_6326.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

依次插入8，10，15  

![insertion2.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856506341_5902.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

插入16  

![insertion3.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856507020_4108.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

插入16后超过了关键字的个数限制，所以要进行分裂。在叶子结点分裂时，分裂出来的左结点2个记录，右边3个记录，中间key成为索引结点中的key，分裂后当前结点指向了父结点（根结点）。结果如下图所示。  

![insertion4.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856508563_2088.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

插入17    

![insertion5.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856509492_4560.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

插入18，插入后如下图所示  

![insertion6.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856510437_3902.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

当前结点的关键字个数大于5，进行分裂。分裂成两个结点，左结点2个记录，右结点3个记录，关键字16进位到父结点（索引类型）中，将当前结点的指针指向父结点。  

![insertion7.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856511116_8186.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

插入若干数据后  

![insertion8.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856512133_7097.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

在上图中插入7，结果如下图所示  

![insertion11.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856512894_1092.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

当前结点的关键字个数超过4，需要分裂。左结点2个记录，右结点3个记录。分裂后关键字7进入到父结点中，将当前结点的指针指向父结点，结果如下图所示。  

![insertion9.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856513717_5691.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

当前结点的关键字个数超过4，需要继续分裂。左结点2个关键字，右结点2个关键字，关键字16进入到父结点中，将当前结点指向父结点，结果如下图所示。

![insertion10.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856514561_4515.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

## B+树删除

如果叶子结点中没有相应的key，则删除失败。否则执行下面的步骤

1. 删除叶子结点中对应的key。删除后若结点的key的个数大于等于d，删除操作结束,否则执行第2步。

2. 若兄弟结点key有富余（>=d+1），向兄弟结点借一个记录，同时用借到的key替换父结（指当前结点和兄弟结点共同的父结点）点中的key，删除结束。否则执行第3步。

4. 若兄弟结点中没有富余的key,则当前结点和兄弟结点**合并**成一个新的叶子结点，并删除父结点中的key（父结点中的这个key两边的孩子指针就变成了一个指针，正好指向这个新的叶子结点），将当前结点指向父结点（必为索引结点），执行第4步（第4步以后的操作和B树就完全一样了，主要是为了更新索引结点）。

4. 若索引结点的key的个数大于等于d，则删除操作结束。否则执行第5步

5. 若兄弟结点有富余，父结点key下移，兄弟结点key上移，删除结束。否则执行第6步

6. 当前结点和兄弟结点及父结点下移key合并成一个新的结点。将当前结点指向父结点，重复第4步。

注意，通过B+树的删除操作后，索引结点中存在的key，不一定在叶子结点中存在对应的记录。

下面是一颗 d=2 2阶B树的删除过程，2阶B数的结点最少2个key，最多4个key。

初始状态  

![suppression1.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856515478_7307.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

删除22,删除后结果如下图  

![suppression2.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856516302_204.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

删除后叶子结点中key的个数大于等于2，删除结束

---

删除15，删除后的结果如下图所示  

![suppression3.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856517094_1466.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

删除后当前结点只有一个key,不满足条件，而兄弟结点有三个key，可以从兄弟结点借一个关键字为9的记录,同时更新将父结点中的关键字由10也变为9，删除结束。

![suppression4.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856518106_6029.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

删除7，删除后的结果如下图所示  

![suppression5.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856518982_9397.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

当前结点关键字个数小于2，（左）兄弟结点中的也没有富余的关键字（当前结点还有个右兄弟，不过选择任意一个进行分析就可以了，这里我们选择了左边的），所以当前结点和兄弟结点合并，并删除父结点中的key，当前结点指向父结点。  

![suppression6.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856519896_7417.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

此时当前结点的关键字个数小于2，兄弟结点的关键字也没有富余，所以父结点中的关键字下移，和两个孩子结点合并，结果如下图所示。

![suppression7.png](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856520889_8357.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

---

**参考资料**  

[B树和B+树的插入、删除图文详解](https://www.cnblogs.com/nullzx/p/8729425.html)  

[《数据库系统概念》:作者: （美）Abraham Silberschatz / （美）Henry F.Korth / （美）S.Sudarshan](https://book.douban.com/subject/10548379/)