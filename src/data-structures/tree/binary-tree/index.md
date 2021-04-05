### Reference
- [二叉树](https://github.com/JTangming/blog/issues/32)

#### DFS（深度优先搜索）和 BFS（广度优先搜索）
二叉树节点定义如下：
```js
/**
 * Definition for a binary tree node.
 * function TreeNode(val, left, right) {
 *     this.val = (val===undefined ? 0 : val)
 *     this.left = (left===undefined ? null : left)
 *     this.right = (right===undefined ? null : right)
 * }
 */
```
##### DFS 遍历使用递归
```js
function dfs(root) {
    if (root === null) {
        return;
    }
    dfs(root.left);
    dfs(root.right);
}
```

##### BFS 遍历使用队列数据结构
```js
function bfs(root) {
    let queue = new Array();
    queue.push(root);
    while (queue.length > 0) {
        let node = queue.pop();
        if (node.left) {
            queue.unshift(node.left);
        }
        if (node.right) {
            queue.unshift(node.right);
        }
    }
}
```