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
 * @return {number[][]}
 */
var levelOrder = function(root) {
    const ret = [];
    if (!root) return ret;

    let queue = [root];
    while(queue.length > 0) {
        let len = queue.length;
        ret.push([]);
        for (let i = 0; i < len; i++) {
            let temp = queue.shift();
            ret[ret.length - 1].push(temp.val);
            if (temp.left) queue.push(temp.left);
            if (temp.right) queue.push(temp.right);
        }
    }

    return ret;
};