本文记录代码随想录中遇到的KMP算法和它的解法、

KMP的经典思想就是:当出现字符串不匹配时，可以记录一部分之前已经匹配的文本内容，利用这些信息避免从头再去做匹配。

介绍：KMP算法
```
  KMP算法是一种改进的字符串匹配算法，由D.E.Knuth，J.H.Morris和V.R.Pratt提出的，因此人们称它为克努特—莫里斯—普拉特操作（简称KMP算法）。
  KMP算法的核心是利用匹配失败后的信息，尽量减少模式串与主串的匹配次数以达到快速匹配的目的。
  具体实现就是通过一个next()函数实现，函数本身包含了模式串的局部匹配信息。KMP算法的时间复杂度O(m+n)，相比起暴力匹配字符串的O(m*n)来说有较优的性能。
```

KMP算法解决了什么问题 - 字符串匹配问题
```
  例如给出一个文本串：aabaabaaf, 一个模式串：aabaaf， 要验证文本串中是否出现过模式串，就是KMP解决的经典问题
```

KMP算法的具体实现 - 先求出模式串的最长相等前后缀数组， 即next数组
例如：
```
为什么使用前缀表：前缀表记录了字符串下标i之前的，有多大长度的相同前缀和后缀。
模式串 aabaaf, 对应的前缀表是：[0, 1, 0, 1, 2, 0], 数组中的元素的含义是 - 模式串中，到当前下标的子串， 它目前的最大相等前后缀的长度。 
而这个长度也用于后续我们的“回退”操作，来返回到模式串的对应下标重新匹配。（回到这个下标就表示了，这个位置前后有最大相等的前后缀，即有对称的特性可减少遍历次数）
```

然后结合next数组，遍历文本串，逐个匹配模式串；
若匹配成功 - 累加当前前缀位置，直到前缀下标等于模式串长度，则判断匹配成功，返回模式串的起点；
若匹配失败 - 当前前缀结合next数组进行回退，尽可能回退到前缀表对应下标减少额外比对

**题目1，实现 strStr()**：
实现 strStr() 函数。
给你两个字符串 haystack 和 needle ，请你在 haystack 字符串中找出 needle 字符串出现的第一个位置（下标从 0 开始）。如果不存在，则返回  -1 。

示例 1：

输入：haystack = "aabaabaaf", needle = "aabaaf"
输出：3

**代码：**

```typescript
function strStr(haystack, needle) {
    // kmp算法 - 思路，先求出模式串的前缀表next数组， 再遍历文本串逐个匹配模式串
    // 若匹配上，则累加匹配长度，直到当前前缀匹配长度等于模式串，则时候输出模式串的起始下标
    // 若未匹配上，则将当前前缀结合next数组进行回退，减少额外遍历

    // 先获取模式串的next数组 - 即前缀表，这里采用的方式是下标不减1也不右移位，原地操作，而对应的处理就是：前后缀不匹配的时候，回退前缀到next数组中的上一个下标。
    const getNext = (str) => {
        const next = [0] // 初始化为0， 因为长度为1的字符串，最长相等前后缀为0
        let j = 0 // j 代表当前前缀的下标， i 代表当前后缀的下标

        for (let i = 1; i < str.length; i++) { // 后缀从第二个字符开始   aabaaf
          while (j > 0 && str[j] !== str[i]) { // 若前缀非起始位置，且当前不匹配。则回退前缀
            j = next[j - 1] // 若一直不匹配，当前的前缀指针会一直回退到起点：下标0的位置
          }
          if (str[i] === str[j]) {
            j++ // 若当前前后缀匹配，则右移动前缀指针，继续匹配
          }
          next.push(j) // next数组收集当前前缀下标（前缀表构造）
        }
        return next
    }

    const next = getNext(needle) // 模式串构造前缀表

    // 下面遍历文本串，逐个匹配模式串
    let j = 0
    for (let i = 0; i < haystack.length; i++) { // i 遍历文本串的后缀位置
        while(j > 0 && haystack[i] !== needle[j]) { // 逐个匹配
            j = next[j - 1] // 模式串不匹配时候的回退操作， 在不匹配的时候，尽可能回退到前缀表对应下标减少额外比对
        }
        if (haystack[i] === needle[j]) {
            j++
        }
        if (j === needle.length) { // 若持续成功匹配，一直到当前模式串的长度和比对前缀下标一样，则判断匹配成功
            return i - needle.length + 1
        }
    }

    return -1
};
```

**题目2，重复的子字符串**：
给定一个非空的字符串 s ，检查是否可以通过由它的一个子串重复多次构成。

示例 1：
输入: s = "abab"
输出: true
解释: 可由子串 "ab" 重复两次构成。

**代码：**

```typescript
function repeatedSubstringPattern(s: string): boolean {
  // 公式 ： 若字符串长度可以整除（字符串长度 - 最长相等前后缀的长度）， 则表示这个字符串是这个周期（next数组）的循环
  const getNext = (str: string) => {
    const next = [0]
    let j = 0
    for (let i = 1; i < str.length; i++) {
      while (j > 0 && str[i] !== str[j]) {
        j = next[j - 1]
      }
      if (str[i] === str[j]) {
        j++
      }
      next.push(j)
    }
    return next
  }

  const next = getNext(s)

  if (next[next.length - 1] !== 0 && s.length % (s.length - next[next.length - 1]) === 0) { // 判断字符串是否整除一个周期（周期：字符串长度 - 最长相同前后缀的长度）
    return true
  }
  return false
};
```

