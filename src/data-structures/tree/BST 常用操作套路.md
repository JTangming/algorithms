### 二叉查找树常用操作套路

根据 https://github.com/JTangming/blog/issues/32 中的一些基本知识用伪代码来加深理解。

#### 核心算法
BST 的核心算法操作就是递归，通过递归来找到临界点从而完成增删改查。简单的遍历框架代码如下：
```ts
let traverse = (root: TreeNode) => {
  // 这里判断临界条件，做相应的操作
  traverse(root.left);
  traverse(root.right);
};
```