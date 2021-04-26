> 二分查找：时间复杂度O(log2n), 空间复杂度O(1)

实现
```javascript
function binarySearch(arr, target) {
  let low = 0, high = arr.length - 1, mid
  while (low <= high) {
    mid = Math.floor(low + high) / 2
    if (arr[mid] === target) {
      return mid
    } else if (arr[mid] > target) {
      high = mid - 1
    } else {
      low = mid + 1
    }
  }
}
```