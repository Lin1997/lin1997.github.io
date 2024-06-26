---
title: JPS寻路算法
tags: 
    - 算法
---

> **参考**
>
> - [A Visual Explanation of JPS](https://zerowidth.com/2013/a-visual-explanation-of-jump-point-search/)

## 前言

2D 网格上最短路径的代价统一搜索算法有很多种。A*算法是广度优先（Dijkstra）和深度优先搜索的一种常见且直接的优化。对此算法有许多扩展，包括D*、HPA*和矩形对称性剪枝，它们都旨在减少寻找最佳路径所需的节点数量。

[Jump Point Search](https://harablog.wordpress.com/2011/09/07/jump-point-search/)算法，是使矩形网格上的路径查找更加高效的众多方法之一。

本文假设你假定您已熟悉A*搜索算法以及DijkstrA*算法在寻路问题上的应用，参见[A*算法的介绍](https://theory.stanford.edu/~amitp/GameProgramming/AStarComparison.html)。

下文假设在规则的正方形网格上进行路径查找，其中水平和垂直移动的成本为1，对角线移动的成本为√2。

> 可以在[该链接](https://zerowidth.com/2013/a-visual-explanation-of-jump-point-search/)可视化执行JPS算法。


## 路径扩展与对称性剪枝

在每次执行A*算法迭代时，我们都会朝着当前已知的最佳方向扩展搜索范围。然而，在某些情况下可能会出现效率低下的情况，例如在大空地中的搜索就是一个例子。

![对称路径](/assets/posts/jps-symmetric-paths.svg)

存在很多等效的最佳路径。Dijkstra的广度优先搜索证明了这一点，你可以看到这些路径的成本都是一样的。唯一的区别在于我们选择以何种顺序进行对角线或水平移动。
在JPS论文中称这些为“对称路径”，因为它们实际上是相同的。理想情况下，我们可以识别这种情况并忽略除一个之外的其它路径。

## 智能地扩展

A*算法通过最简单的方式来扩展其搜索：将一个节点的直接邻居添加到 接下来要检查的集合(open set)中。如果我们能够稍微向前看一点，并跳过一些我们直觉上认为不值得直接查看的节点，那会怎样？我们可以尝试识别路径对称性存在的情况，并在扩展搜索时忽略某些节点。

## 向前看：水平和垂直方向

![水平移动](/assets/posts/jps-horizontal-expand.svg)
我们先看一下水平运动，让我们考虑从绿色节点向右移动。我们可以对这个节点的直接邻居做出几个假设。

![忽略我们来自的父节点](/assets/posts/jps-horizontal-parent.svg)
首先，我们可以忽略我们来自的父节点（已经访问过），用灰色中标记。

![忽略身后对角的节点](/assets/posts/jps-diagonally-behind.svg)
其次，我们可以假设我们身后对角线的节点是通过我们的父节点到达的，因为那些路径比通过我们到达要短。

![忽略上下的节点](/assets/posts/jps-above-below.svg)
上面和下面的节点也可以从我们的父节点更优化地到达。如果通过我们，成本是 2 而不是 √2̅，所以我们也可以忽略它们。

![忽略对角前方的节点](/assets/posts/jps-diagonally-front.svg)
我们前方对角线的邻居可以通过我们上方和下方的邻居到达。与通过我们的路径成本相同，所以为了简单起见，我们假设前者方式更好，并忽略这些节点。

![只关注前方节点](/assets/posts/jps-front.svg)
因此只留下了一个节点需要检查：我们正前方的那一个。我们已经假设了我们所有其他的邻居都通过其他路径到达，所以我们可以只关注这一个邻居。

![继续跳过节点](/assets/posts/jps-jump-ahead.svg)
这就是诀窍：只要道路畅通，我们继续跳过节点到右边，并重复我们的检查，而不必正式地将节点添加到open set中。

但是“道路畅通”是什么意思？我们的假设何时不正确，何时需要停下来仔细看看？

![上方有障碍物,强制邻居](/assets/posts/jps-forced-neighbor.svg)
上文对右对角线的节点做出了一个假设，即任何等效的路径都会通过我们上方和下方的邻居经过。但是有一种情况不成立：如果上方或下方的障碍物阻挡了道路。

在这里，我们必须停下来重新检查。我们不仅要看我们右边的节点，我们还被迫看它对角线向上的那一个。在JPS论文中，这被称为**强制邻居**，因为我们被迫考虑它。

当我们到达一个有强制邻居的节点时，我们停止向右跳跃，并将当前节点添加到open set中，以便后续进一步检查和扩展。

![前方有障碍物](/assets/posts/jps-blocked-ahead.svg)
假设：如果我们在向右跳跃时道路被阻挡，我们可以安全地忽略整个跳跃。我们已经假设了我们上方和下方的路径是通过其他节点处理的，且我们还没有因为强制对角线邻居而停下来。因为我们只关心我们正右方的内容，那里的障碍物意味着没有其他地方可去。

## 向前看：对角线方向

![对角线移动](/assets/posts/jps-diagonally-move.svg)
我们可以应用类似的简化假设来移动对角线。在这个例子中，我们向上和向右移动。

![忽略左和下节点](/assets/posts/jps-left-below.svg)
我们可以做出的第一个假设是，我们左边和下方的邻居可以从我们的父节点通过直接垂直或水平移动最优化地到达。

![忽略左边向上以及下边向右节点](/assets/posts/jps-left-below-expand.svg)
我们也可以对左边向上以及下边向右的节点做出同样的假设。这些也可以通过我们左方和下方的邻居更高效地到达。

![考虑对角线及两个分量方向](/assets/posts/jps-diagonal-and-components.svg)
因此我们只剩下三个邻居需要考虑：一个在我们原始移动方向的对角线上，两个分量方向: 上方和右方。

![左或下有障碍物,强制邻居](/assets/posts/jps-forced-neighbor-2.svg)
与水平移动时的强制邻居类似，当左边或下方有障碍物存在时，对角线左上方或右下方邻居不能通过其他方式到达，只能通过我们。这些是对角线移动的强制邻居。

当我们对角线移动时，有三种邻居需要考虑，我们如何向前跳跃呢？

我们的两个邻居需要垂直或水平移动。通过前文已经知道如何在这些方向上跳过，我们可以先看这些方向，如果没有发现值得查看的节点，我们再对角线移动一步，然后再次尝试。

例如，以下是对角线跳跃的几个步骤。在沿对角线移动之前，先考虑水平和垂直路径，直到其中一个路径发现了值得进一步考虑的节点。在本例中，它是障碍的边缘，它被检测到是因为当我们向右跳跃时，它有一个强制邻居。

![对角线扩展步骤1](/assets/posts/jps-diagonally-move-step-1.svg)
首先，我们水平和垂直扩展。两者都遇到了障碍（或地图的边缘），所以我们可以对角线移动。

![对角线扩展步骤2](/assets/posts/jps-diagonally-move-step-2.svg)
再次，垂直和水平扩展都遇到了障碍，所以我们对角线移动。

![对角线扩展步骤3](/assets/posts/jps-diagonally-move-step-3.svg)
第三次...

![对角线扩展步骤4](/assets/posts/jps-diagonally-move-step-4.svg)
最后，垂直扩展还是到达了地图的边缘，但向右的跳跃找到了一个强制邻居。

![对角线扩展步骤5](/assets/posts/jps-diagonally-move-step-5.svg)
此时，我们将当前节点添加到open set中，并继续进行类似A*算法的下一次迭代。

## 小结：算法步骤

现在我们已经考虑了在特定方向移动时跳过大多数邻居节点的方法，并确定了一些我们可以跳过的规则。

为了将这些与A*算法联系起来，我们将在每次检查open set中的节点时应用这些“向前跳跃”步骤。我们将使用其父节点来确定移动方向，并尽可能远地向前跳过。如果我们发现了一个感兴趣的节点，我们将忽略所有中间步骤（因为我们已经使用我们的简化规则跳过了它们），并将那个新节点添加到open set中。

open set中的每个节点都是基于其父节点的方向进行扩展的，遵循之前的跳跃点扩展：首先水平和垂直方向，然后是对角线方向。
下面是路径扩展的一个完整示例步骤。

![JPS步骤0](/assets/posts/jps-step-0.svg)
与之前的例子相同，但目标节点被标记出来了。

![JPS步骤1](/assets/posts/jps-step-1.svg)
从已确定的节点（open set中唯一的一个）开始，我们垂直扩展但一无所获。

![JPS步骤2](/assets/posts/jps-step-2.svg)
水平跳跃找到一个有强制邻居的节点（用紫色高亮显示）。我们将这个节点添加到open set中。

![JPS步骤3](/assets/posts/jps-step-3.svg)
最后，我们继续对角线扩展，因为我们碰到了地图的边缘，所以没有发现任何东西。

![JPS步骤4](/assets/posts/jps-step-4.svg)
接下来我们检查下一个最佳的（在此例中，唯一的）开放节点。因为我们在到达这个节点时是水平移动的，所以我们继续水平跳跃。

![JPS步骤5](/assets/posts/jps-step-5.svg)
因为我们也有一个强制邻居，我们也在这个方向上扩展。遵循对角线跳跃的规则，我们先对角线扩展，然后水平和垂直方向都看看。

![JPS步骤6](/assets/posts/jps-step-6.svg)
一无所获，我们再次对角线扩展。

![JPS步骤7](/assets/posts/jps-step-7.svg)
这次，当我们水平扩展时（无处可去），垂直扩展时，我们看到目标节点。这和发现一个有强制邻居的节点一样有趣，所以我们将这个节点添加到open set中。

![JPS步骤8](/assets/posts/jps-step-8.svg)
最后，我们扩展最后一个开放节点，到达目标。

![JPS步骤9](/assets/posts/jps-step-9.svg)
跳过算法的最后一次迭代——将目标本身添加到open set中只是为了识别它: 我们找到了（一个）最佳路径！

