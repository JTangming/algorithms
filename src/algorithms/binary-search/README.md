### 二分查找
二分查找是一种基于比较目标值和数组中间元素的算法，即：

- 如果目标值等于中间元素，则找到目标值。
- 如果目标值较小，继续在左侧搜索。
- 如果目标值较大，则继续在右侧搜索。

它是一种效率较高的查找方法，前提是数据结构必须先排好序，可以在数据规模的对数时间复杂度内完成查找。

#### 简单的例子
原题： https://leetcode-cn.com/problems/binary-search/solution/er-fen-cha-zhao-by-leetcode/
给定一个 n 个元素有序的（升序）整型数组 nums 和一个目标值 target  ，写一个函数搜索 nums 中的 target，如果目标值存在返回下标，否则返回 -1。

例子：
输入: nums = [-1,0,3,5,9,12], target = 9
输出: 4
解释: 9 出现在 nums 中并且下标为 4

```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number}
 */
var search = function(nums, target) {
    let l = 0, r = nums.length - 1;
    let ret = -1;
    while (l <= r) {
        let mid = (l + r) >> 1;
        if (nums[mid] === target) {
            ret = mid; 
            break;
        }    
        if (nums[mid] > target) {
            r = mid - 1;
        } else {
            l = mid + 1;
        }
    }
    return ret;
};
```

时间复杂度：O(log N)，即为二分查找需要的次数。
空间复杂度：O(1).