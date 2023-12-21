---
title: 关系数据库设计-函数依赖
summary: 函数依赖设计方法
date: 2019-12-10
tags: ['Data Base']
mathjax: true
---

一般而言，关系数据库设计的目标是生成一组关系模式，是我们存储信息时避免不必要的冗余，并方便地获取信息。这是通过设计满足**适当范式(Normal Form or in French "Forme Normale")**来实现的。本文简短地通过**函数依赖**介绍设计方法。

# 概念

1. **实体**：现实世界中客观存在并可以被区别的事物。比如“一个学生”、“一本书”、“一门课”等等。值得强调的是这里所说的“事物”不仅仅是看得见摸得着的“东西”，它也可以是虚拟的，不如说“老师与学校的关系”。
2. **属性**：教科书上解释为：“实体所具有的某一特性”，由此可见，属性一开始是个逻辑概念，比如说，“性别”是“人”的一个属性。在关系数据库中，属性又是个物理概念，属性可以看作是“表的一列”。
3. **元组**：表中的一行就是一个元组。
4. **分量\*\***：元组的某个属性值。在一个关系数据库中，它是一个操作原子，即关系数据库在做任何操作的时候，属性是“不可分的”。否则就不是关系数据库了。
5. **码**：表中可以唯一确定一个元组的某个属性（或者属性组），如果这样的码有不止一个，那么大家都叫候选码，我们从候选码中挑一个出来做老大，它就叫主码。
6. **全码**：如果一个码包含了所有的属性，这个码就是全码。
7. **主属性**：一个属性只要在任何一个候选码中出现过，这个属性就是主属性。
8. **非主属性**：与上面相反，没有在任何候选码中出现过，这个属性就是非主属性。
9. **外码**：一个属性（或属性组），它不是码，但是它别的表的码，它就是外码。

# 标识

- 属性：A，B，C

- 属性的集合：X, Y, Z 其中 XY = X $\cup$ Y

- 全体属性：U

- 属性集：R

- 关系：r

- 关系模式 r(R)
- 元组：t, t' 元组中的属性值：t.A, t.B
- 函数依赖：f，g，h
- 函数依赖的集合：F，G，H

# 函数依赖

定义：设 X,Y 是关系 R 的两个属性集合，当任何时刻 R 中的任意两个元组中的 X 属性值相同时，则它们的 Y 属性值也相同，则称 X 函数决定 Y，或 Y 函数依赖于 X。

函数依赖：$f: X \to  Y$ 需满足以下两个条件

1. $XY \subseteq Z$

2. $\forall t,t’ \in r: t.X = t'.X \Rightarrow t.Y = t'.Y$

**理解**：若在一张表中，属性或属性组 X 确定，必定能确定属性 Y 的值，则称 Y 函数依赖于 X,记做 X->Y.如下表，在学生表 ST 中通过学号能唯一确定一个姓名，则姓名函数依赖于学号

学生表 ST

| 学号 | 姓名 | 性别 |
| :--: | :--: | :--: |
|  1   | 张三 |  男  |

教师表 T

| TID (教师编号) | 姓名 | 性别 |
| :------------: | :--: | :--: |
|       1        | 李四 |  男  |

课程表 SC

| CID (课程号) | 课程名称 | TID(授课老师 ID) | 学分 |
| :----------: | :------: | :--------------: | :--: |
|      1       |   数学   |        1         |  2   |

成绩表 SG

| 学号 | CID(课程号) | 成绩 |
| :--: | :---------: | :--: |
|  1   |      1      |  80  |

## 平凡函数依赖&非平凡函数依赖

**定义**：设一个关系为 R(U),X 和 Y 为属性集 U 上的子集,若 X->Y 且 X 不包含 Y,则称 X->Y 为非平凡函数依赖,否则若 X 包含 Y 则必有 X->Y,称此 X->Y 为平凡函数依赖.

**举例**：在学生表 ST 中,学号总能函数决定它本身,记作“学号->学号”,对于任一个给定的学号,都有它本身的学号值唯一对应,此为平凡函数依赖.（学号，性别）->性别与（学号，性别）->学号也是平凡函数依赖

通常，我们主要讨论的是非平凡函数依赖：如学生表 ST 中学号函数决定的其他属性都是非平凡函数依赖

## 完全函数依赖

**定义**：设 X,Y 是关系 R 的两个属性集合，X’是 X 的真子集，存在 X->Y，但对每一个 X’都有 X’!->Y，则称 Y 完全函数依赖于 X。

**理解**：在一张表中，若 X -> Y，且对于 X 的任何一个真子集（假如属性组 X 包含超过一个属性的话），X ' -> Y 不成立，那么我们称 Y 对于 X 完全函数依赖。

**举例**：在成绩表 SG 中，成绩完全函数依赖于（学号，CID（课程号））

## 部分函数依赖

**定义**：设 X,Y 是关系 R 的两个属性集合，存在 X->Y，若 X’是 X 的真子集，存在 X’->Y，则称 Y 部分函数依赖于 X。

**说明**：假如 Y 函数依赖于 X，但同时 Y 并不完全函数依赖于 X，那么我们就称 Y 部分函数依赖于 X

**举例**：学生表 ST 中（学号，姓名）-> 性别，但是存在学号 -> 性别，所以称性别部分依赖于（学号，姓名）

## 传递函数依赖

**定义**：设 X,Y,Z 是关系 R 中互不相同的属性集合，存在 X->Y(Y !->X),Y->Z，则称 Z 传递函数依赖于 X。

**说明**：假如 Z 函数依赖于 Y，且 Y 函数依赖于 X （严格来说还有一个 X 不包含于 Y，且 Y 不函数依赖于 Z 的前提条件），那么我们就称 Z 传递函数依赖于 X

**举例**：关系 S1（学号，系名，系主任），学号 -> 系名，系名 -> 系主任，并且系名 ！-> 学号，所以学号 -> 系主任为传递函数依赖。

## 函数依赖集合

推论：设 F 为一个在集合 U 上的函数依赖的集合，若关系 r(U)中的所有属性满足 F 中的所有函数依赖，则它们也同样满足 $X \to Y$

> 如果在 r(R)上的每个合法实例都满足 $X \to Y$, 那我们说该函数依赖在模式 r(R)上**成立(hold)**.

即 $F \models X \to Y$

若 $F = {A -> B, B -> C}$, 则$ F \models A \to C$

# 超码(superkey) & 码/候选码(key)

超码：设有 R(Z)为关系模式, F 为 Z 上的函数依赖集合，X 是属性集 R 的子集，$X \subseteq Z$, 若$F \models X \to Z$，则 X 是 R 的超码(surclé 或 superkey)。

若**只存在一个**这样的超码，则 X 是 R 上的码:

或者是说：设 K 为关系模式 R\<U, F\>的属性(组)，若可推导出函数依赖 K->U，则称 K 为 R 的**码/候选码**

> une surclé X est appelé une clé de R, s'il n’existe pas de sous-ensemble stricte Y⊂ X de X qui est aussi une surclé.
>
> Une clé de R est un ensemble minimal (-au sens de l’inclusion) qui détermine tous les autres
>
> Surclé d’une relation : ensemble d’attributs dont la connaissance des valeurs permet d’identifier un n-upplet unique de la relation considérée. Il peut exister plusieurs surclé. Une relation étant un ensemble : l’ensemble des attributs de la relation forme toujours une surclé.
>
> Clé d’une relation : est une surclé minimale. Par conséquent si on retire un attribut de la clé alors le reste des attributs ne forme plus une surclé.
> Lorsqu’il existe plusieurs clés, on en choisit une que l’on nomme clé primaire, elle est représentée dans le schéma de relation en soulignant le nom des attributs concernés par la clé.

1. 问题：若 F = \{A -> B, B -> C\}, 那么 A 是 R(ABC)的码吗？

   需要证明：$F\models A \to ABC$

   证明：当且仅当 $∀t,t’ ∈ r: t.A = t'.A \Rightarrow t.B = t'.B$, 且 $t.B = t'.B \Rightarrow t.C = t'.C$, r 满足 F

   我们可以得到 $∀t,t’ ∈ r: t.A = t’.A \Rightarrow t.A = t'.A \land t.B = t'.B \land t.C = t'.C$

2. 若 F = \{A -> B, B -> C\}, 那么 A 是 R(AC)的码吗？

   不足以证明。因为 R 中不包含 B，所以我们可推断出 A 不是一个码。

# 传递闭包

函数依赖的闭包定义：$F^+ = \lbrace X \to Y  \vert F \models X \to Y \rbrace$

**属性值的闭包定义**: $[X]^{+}_{F} = \lbrace A \vert F\models X \to A \rbrace$

给定属性值集合 X，函数依赖 F，计算 X 的闭包：

1. 先将$X^{+}$:=$X$

2. 当存在$Y \to Z$且 $Y \subseteq X^{+}, Z \nsubseteq X^{+} $ （此处可以将 Z 分解，使得箭头右侧只有一个属性值）

   $X^{+} := X^{+} \cup Z$

3. 将$Y \to Z$从 F 中去掉，避免重复

   重复第二，三步，直到结束，此时便得到了 X 的闭包$X^{+}$

## 应用

1. 问题：给定函数依赖 A -> B, AB->CF, BC->D，求 A 的闭包

   1. 初始 A 的闭包集为\{A\}

   2. A->B: 因为 A 在 A 的闭包集中，且 B 不在 A 的闭包集中，那么向 A 的闭包集中加入 B

   3. AB->CF：将其分解为 AB->C 和 AB->F，AB 在闭包集中，且 C 不在，加入 C
   4. 同样的加入 F
   5. BC->D：同样
   6. 最终 A 的闭包集为\{A,B,C,D,F\}

2. 问题：有如下关系 R(学生，老师，课程，教室，时间)，如下函数依赖，

   - 学生，时间 -> 教室
   - 教室，时间 -> 课程
   - 课程 -> 老师，教室
   - 学生，教室 -> 时间

   求出这个关系的码。

   ***

   对应 R\< U、F\>（U 是属性集，F 是函数依赖集）

   - 如果有属性不在 F 中出现，那么它必须包含在候选码中

   - 如果有属性在所有函数依赖中一直存在于左边，则它必包含在候选码中；同理只在右边出现过的属性一定不属于候选码

   - 如果有属性或属性组能唯一标识元组，则它就是候选码

   ***

   1. 哪个属性从未在右侧出现？学生
   2. 学生是一个码吗？不是。学生+=学生
   3. 我们尝试先试一个，从时间/教室开始
   4. (学生，时间)是一个码吗？是。(学生，时间) -> (学生，老师，课程，教室，时间) 可以通过闭包得到
   5. (学生，教室)是一个码吗？是。(学生，教室) -> (学生，老师，课程，教室，时间)

# 基本集，最小基本集

给定一个函数依赖集 F；

**F 的基本集**：和 A 等价的函数依赖集；

F=\{AB -> D, D ->C, C ->D, AB ->C\}

G=\{AB -> C, D ->C, C ->D\}

两个函数依赖集合是等价的，但 G 更加**紧密(compact)**。对于两个相等的函数依赖集合，可以说 F ≡ G

**F 的最小化基本集**：满足

1. 函数依赖的右边只有单个属性
2. 删除任意一个函数依赖都不将是基本集，也就是说不重复
3. 从函数依赖的左边删除任意一个属性，则不将是基本集；也就是说不包含左侧过多/重复的属性

**F 是 G 的最小覆盖**: 若 F ≡ G 并且 F 满足最小化基本集

## 最小覆盖

给定 F=\{AB->CD, ACE->B, D->C, C->D, CD->BE\}

将其分解，使箭头右侧只有一个属性

$F_2$=\{AB->C, AB->D, ACE->B, D->C, C->D, CD->B, CD->E\}

除去重复的函数依赖：

- ACE->B, 因为 $C->D, CD->B \models C-> Β$
- AB->C **或** AB->D
  - 因为$AB->D, D->C \models AB->C$
  - 或者因为$ AB->C, C->D \models AB->D$

现在得到新的$F_2$=\{AB->C, D->C, C->D, CD->B, CD->E\} (除去了 AB->D，当然也可以除去 AB->C，那么就有了两种可能性)

现在除去箭头左侧重复的属性值, 可以看到 CD->B, CD->E 的箭头左侧有两个属性，C 和 D

计算它们的闭包：

$[C]^+ _ {F_2} = {C, D, B, E}$ 以及 $[D]^+_{F_2} = {C, D, B, E}$

我们可以在 CD->B 或 CD->E 除去 C 或 D，这样在这一步就有了 4 种可能性

因此一共可以有 2\*4 = 8 种可能性

现在得到

1. F2.1=\{AB->C, D->C, C->D, C->B, C->E\}
2. F2.2=\{AB->C, D->C, C->D, C->B, D->E\}
3. F2.3=\{AB->C, D->C, C->D, D->B, C->E\}
4. F2.4=\{AB->C, D->C, C->D, D->B, D->E\}
5. F3.1=\{AB->D, D->C, C->D, C->B, C->E\}
6. F3.2=\{AB->D, D->C, C->D, C->B, D->E\}
7. F3.3=\{AB->D, D->C, C->D, D->B, C->E\}
8. F3.4=\{AB->D, D->C, C->D, D->B, D->E\}

# Armstrong 公理

假定 X, Y 和 Z 是关系 R 的属性集

- 增广:${X ->Y} \Rightarrow {XZ -> YZ}$
- 传递: ${X ->Y, Y -> Z} \Rightarrow {X -> Z}$
- 自反: $W ⊆ X \Rightarrow {X -> W}$
- 附加 :
  - 合并: ${X->Y, X->Z} \Rightarrow (X->YZ)$
  - 分解: ${X->YZ} \Rightarrow {X->Y, X->Z}$
  - 伪传递: ${X->Y, WY->Z} \Rightarrow {XW->Z}$

## 例子

假定 F = \{A -> C, B -> D\}，证明 AB 是 R=ABCD 的码

1.  AB -> ABC (A -> C + 增广)
2.  ABC ->ABCD (B -> D + 增广)
3.  AB ->ABCD (1.+2. + 传递)

---

参考资料

[数据库学习](https://www.jianshu.com/p/abf48dfec989)

[关系数据库设计](https://josonle.github.io/blog/2018-11-12-%E5%85%B3%E7%B3%BB%E6%95%B0%E6%8D%AE%E5%BA%93%E8%AE%BE%E8%AE%A1%EF%BC%88F-%E9%97%AD%E5%8C%85%E3%80%81%E5%80%99%E9%80%89%E7%A0%81%E6%B1%82%E8%A7%A3%E3%80%81%E8%8C%83%E5%BC%8F%E5%88%A4%E6%96%AD%E5%8F%8ABCNF%E5%88%86%E8%A7%A3%EF%BC%89.html/)

[Système de Gestion de Base de Données](http://www-bd.lip6.fr/wiki/site/enseignement/licence/3i009/start)

[《数据库系统概念》:作者: （美）Abraham Silberschatz / （美）Henry F.Korth / （美）S.Sudarshan](https://book.douban.com/subject/10548379/)

---
