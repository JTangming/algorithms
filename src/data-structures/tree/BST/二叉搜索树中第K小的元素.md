### 二叉搜索树中第K小的元素

原题：https://leetcode-cn.com/problems/kth-smallest-element-in-a-bst/

BST，除了它的定义，还有一个重要的性质：BST 的中序遍历结果是有序的（升序）。

```js
/**
 * Definition for a binary tree node.
 * function TreeNode(val, left, right) {
 *     this.val = (val===undefined ? 0 : val)
 *     this.left = (left===undefined ? null : left)
 *     this.right = (right===undefined ? null : right)
 * }
 */
/**
 * @param {TreeNode} root
 * @param {number} k
 * @return {number}
 */
var kthSmallest = function(root, k) {
    /*** 递归 */
    // let arr = [];

    // var traverse = (node) => {
    //     if (node === null) return;
    //     traverse(node.left);
    //     arr.push(node.val);
    //     traverse(node.right);
    // }

    // traverse(root);

    // return arr[k-1];

    /*** 中序遍历迭代 */
    let stack = new Array();
    stack.push(root);
    let cur = root;
    let index = 0;
    let res = null;
    while (stack.length > 0 || cur !== null) {
        while (cur !== null) {
            if (cur.left) stack.push(cur.left);
            cur = cur.left;
        }
        let node = stack.pop();
        index++;
        if (index === k) {
            res = node.val;
            break;
        }
        if (node.right !== null) {
            stack.push(node.right);
            cur = node.right;
        }
    } 

    return res;
};
```