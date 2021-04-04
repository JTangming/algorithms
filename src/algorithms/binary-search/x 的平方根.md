### x 的平方根
原题：https://leetcode-cn.com/problems/sqrtx/

#### 二分查找

找到中间值，这里假设 l = 0, r = x，mid = l + Math.floor((r - l) / 2).

- 如果中间元素的平方小于等于目标值 x，我们认为是目标值；
- 如果目标值较小，继续在左侧搜索；
- 如果目标值较大，则继续在右侧搜索；
- 直到 l > r 为止。

```js
/**
 * @param {number} x
 * @return {number}
 */
var mySqrt = function(x) {
    let l = 0, r = x;
    let ans = 0;
    while (l <= r) {
        let mid = (l + r) >> 1;
        if (mid * mid <= x) {
            ans = mid;
            l = mid + 1
        } else {
            r = mid - 1;
        }
    }
    return ans;
};
```