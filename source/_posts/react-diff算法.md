---
title: react-diff算法
date: 2019-03-02 10:19:17
tags: [react, diff]
---

## react v15的diff算法
React diff 会帮助我们计算出**Virtual DOM**中真正变化的部分，并只针对该部分进行实际 DOM 操作，而非重新渲染整个页面，从而保证了每次操作更新后页面的高效渲染，因此 Virtual DOM 与 diff 是保证 React 性能口碑的幕后推手。

### React diff 前提
- Web UI中DOM节点跨层级的移动操作特别少，可以忽略不计。-> 所以，只比较同一层级的。tree diff
- 拥有相同类的两个组件会生成相似的树结构，反之生成不同的树结构。-> component diff
- 对于同一层级的子节点，添加唯一key值区分。-> element diff
React基于上述三大策略，将O(n^3) 复杂度的问题转换成 O(n) 。

### tree diff
通过updateDepth对Virtual DOM 树进行层级控制，只会对**同一层级**下的同一父节点的子节点进行比较。
当出现跨层级的DOM移动操作时，会新建节点，不会去移动，所以React建议不进行DOM节点跨层级的操作。

### component diff
- 如果是同一类型的组件，按照原策略继续比较 virtual DOM tree。
- 如果不是，则将该组件判断为 dirty component，从而替换整个组件下的所有子节点。
- 对于同一类型的组件，允许用户通过shouldComponentUpdate()来判断组件是否diff，以及PureComponent的shouldComponentUpdate的shallowEqual

### element diff
对于同一层级的节点，React diff提供了三种节点操作：INSERT_MARKUP（插入）、MOVE_EXISTING（移动）和 REMOVE_NODE（删除）。
使用**设置唯一key**的方式进行优化，首先对新集合的点进行遍历，对于已经存在的点，判断当前节点在老集合中的位置child._mountIndex是否小于lastIndex(表示访问过的节点在老集合中最右的位置)，若小于，说明该节点在新集合中的位置靠后，则移动该节点；反之，则不动。
对于不存在的点，则创建该节点。
新集合的点diff遍历完成后，需要再遍历老集合，删去老集合中存在但新集合中不存在的点。
PS: 在遍历之前，会先更新节点的子元素和创建新元素，最终返回的就是待遍历的新集合

##REFs: 
[React 源码剖析系列 － 不可思议的 react diff](https://zhuanlan.zhihu.com/p/20346379)

### Diffing 算法（reconciliation协调）
1. 首先比较两棵树的根节点。不同类型的根节点元素会有不同的形态
2. 对比不同类型的元素：当根节点为不同类型的元素时，React会拆卸原有的树并且建立起新的树。当拆卸一棵树时，React 会销毁 Counter 组件并且重新装载一个新的组件。
3. 对比同一类型的DOM元素：React 会保留 DOM 节点，仅比对及更新有改变的属性
4. 对比同类型的组件元素：当一个组件更新时，组件实例保持不变，这样 state 在跨越不同的渲染时保持一致
5. 对子节点进行递归：当递归 DOM 节点的子元素时，React 会同时遍历两个子元素的列表；当产生差异时，生成一个 mutation。给子节点添加key值
由于 React 依赖探索的算法，因此当以下假设没有得到满足，性能会有所损耗。
1. 该算法不会尝试匹配不同组件类型的子树。如果你发现你在两种不同类型的组件中切换，但输出非常相似的内容，建议把它们改成同一类型。
2. Key 应该具有稳定，可预测，以及列表内唯一的特质。不稳定的 key（比如通过 Math.random() 生成的）会导致许多组件实例和 DOM 节点被不必要地重新创建，这可能导致性能下降和子组件中的状态丢失。
