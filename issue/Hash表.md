**介绍：**
Hash表是什么？
```
  1、哈希表（英文名字为Hash table，国内也有一些算法书籍翻译为散列表，大家看到这两个名称知道都是指hash table就可以了）。
  2、哈希表是根据关键码的值而直接进行访问的数据结构。
  3、那么哈希表能解决什么问题呢，一般哈希表都是用来快速判断一个元素是否出现集合里。
      例如要查询一个名字是否在这所学校里。
      要枚举的话时间复杂度是O(n)，但如果使用哈希表的话， 只需要O(1)就可以做到。
  4、Hash碰撞：
    若多个元素都映射到了相同索引下标的位置，我们可以使用拉链法或线性探索法来解决。
    拉链法：将发生冲突的元素都被存储在链表中。 这样我们就可以通过索引区分重复的元素。
    线性探测法：保证tableSize大于dataSize。 我们需要依靠哈希表中的空位来解决碰撞问题。
```

**题目1，有效的字母异位词**：
给定两个字符串 s 和 t ，编写一个函数来判断 t 是否是 s 的字母异位词。
示例 1: 输入: s = "anagram", t = "nagaram" 输出: true
示例 2: 输入: s = "rat", t = "car" 输出: false

说明: 你可以假设字符串只包含小写字母。

**代码：**

```typescript
function isAnagram(s: string, t: string): boolean {
  if(s.length !== t.length) {
    return false // 异位字符串长度一定相等
  }
  // 思路， 使用数组哈希表， 长度为26，对应26位a-z字母出现次数
  const hash = Array(26).fill(0)
  const baseASCII = 'a'.charCodeAt(0) // 'a' - 'z'的ASCII 是97到122， 所以可以利用当前字符 - 'a'的ASCII的差值作为当前字符在hash表中的映射key，即数组下标

  // 遍历字符串，修改hash
  for (let i = 0; i < s.length; i++) {
    hash[s[i].charCodeAt(0) - baseASCII]++ // 第一个字符用于入hash
    hash[t[i].charCodeAt(0) - baseASCII]-- // 第一个字符用于抵消已有hash
  }
  return hash.every((item:number) => item === 0) // 只要最后hash数组每一项都被抵消，则为异位字符
};
```


**题目1-2，字母异位词分组**：
给你一个字符串数组，请你将 字母异位词 组合在一起。可以按任意顺序返回结果列表。
字母异位词 是由重新排列源单词的字母得到的一个新单词，所有源单词中的字母通常恰好只用一次。

**代码：**
```typescript
function groupAnagrams(strs: string[]): string[][] {
  let map = new Map()
  for (let i = 0; i < strs.length; i++) {
    const sortKey = strs[i].split('').sort().join('') // ['b', 'a', 'e'].sort() 按ASCII码排序为统一的key
    map.get(sortKey) ? map.get(sortKey).push(strs[i]) : map.set(sortKey, [strs[i]])
  }
  return [...map.values()]
};
```

**题目2，快乐数**：
编写一个算法来判断一个数 n 是不是快乐数。

「快乐数」 定义为：
对于一个正整数，每一次将该数替换为它每个位置上的数字的平方和。
然后重复这个过程直到这个数变为 1，也可能是 无限循环 但始终变不到 1。
如果这个过程 结果为 1，那么这个数就是快乐数。
如果 n 是 快乐数 就返回 true ；不是，则返回 false 。

**代码：**

```typescript
function isHappy(n: number): boolean {
  const calculateN = (num: number): number  => { // 计算一个数的完全平方数
    return String(num).split('').reduce((pre, cur) => pre + Number(cur)*Number(cur), 0)
  }

  const mySet = new Set() // set来去重， 判断当前是否有计算过数字n
  let res = n

  while(res !== 1 && !mySet.has(res)) { // 每次记录当前拆分的数字到set中，如果有重复出现则退出循环
    mySet.add(res)
    res = calculateN(res)
  }
  return res === 1
};

```

**题目3，三数之和**：
给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有和为 0 且不重复的三元组。
注意：答案中不可以包含重复的三元组。

```typescript
function threeSum(nums: number[]): number[][] {
  const res = []
  nums.sort((a, b) => a -b)

  for (let i = 0; i < nums.length - 2; i++) {
    // 对i去重
    if (i > 0 && nums[i] === nums[i - 1]) {
        continue
    }
    let j = i + 1, k = nums.length - 1
    while (j < k) {
      let sum = nums[i] + nums[j] + nums[k]
      if (sum === 0) {
        res.push([nums[i], nums[j], nums[k]])
        // 对j 、 k去重
        while(nums[j] === nums[j + 1]) j++
        while(nums[k] === nums[k - 1]) k--
        j++
        k--
      } else if (sum > 0) {
        k--
      } else {
        j++
      }
    }
  }

  return res
};
```


**题目4，四数之和**：
给你四个整数数组 nums1、nums2、nums3 和 nums4 ，数组长度都是 n ，请你计算有多少个元组 (i, j, k, l) 能满足：
  nums1[i] + nums2[j] + nums3[k] + nums4[l] == 0

```typescript
function fourSum(nums: number[], target: number): number[][] {
  // 思路：跟三数之和类似，只是三数只需要先遍历一次第一个数开启while循环，而四数在这个基础上需要再处理第二个数
  const res = []
  nums.sort((a, b) => a -b)

  for (let i1 = 0; i1 < nums.length - 3; i1++) {  // i1后面留三位给i2、j、k
    // 对i1去重
    if (i1 > 0 && nums[i1] === nums[i1 - 1]) {
        continue
    }
    for (let i2 = i1 + 1; i2 < nums.length - 2; i2++) {  // i2后面留两位给j、k
      // 第二位i2去重
      if (i2 - i1 > 1 && nums[i2] === nums[i2 - 1]) {
        continue
      }
      let j = i2 + 1, k = nums.length - 1
      while (j < k) {
        let sum = nums[i1] + nums[i2] + nums[j] + nums[k]
        if (sum === target) {
          res.push([nums[i1], nums[i2], nums[j], nums[k]])
          // 对j 、 k去重
          while(nums[j] === nums[j + 1]) j++
          while(nums[k] === nums[k - 1]) k--
          j++
          k--
        } else if (sum > target) {
          k--
        } else {
          j++
        }
      }
    }
 
  }

  return res
};
```

**题目5，四数想加**：
给你四个整数数组 nums1、nums2、nums3 和 nums4 ，数组长度都是 n ，请你计算有多少个元组 (i, j, k, l) 能满足：
nums1[i] + nums2[j] + nums3[k] + nums4[l] == 0

```typescript
function fourSumCount(nums1: number[], nums2: number[], nums3: number[], nums4: number[]): number {
  const myMap = new Map()
  let res = 0
  // map收集前两个数的和
  for (let i of nums1) {
    for (let j of nums2) {
      const sum = i + j
      myMap.set(sum, myMap.get(sum) ? myMap.get(sum) + 1 : 1)
    }
  }
  for (let k of nums3) {
    for (let l of nums4) {
      const sum = k + l
      const target = myMap.get(0 - sum)
      if (target) {
        // 若当前的后两个数，能找到对应的前两个数的映射， 则结果里累加当前前两个数和的情况下的元组数量
        res += target
      }
    }
  }
  return res
};
```

**题目6，赎金信**：
给你两个字符串：ransomNote 和 magazine ，判断 ransomNote 能不能由 magazine 里面的字符构成。
如果可以，返回 true ；否则返回 false 。
magazine 中的每个字符只能在 ransomNote 中使用一次。

```typescript
function canConstruct(ransomNote: string, magazine: string): boolean {
  // 和异位字符串类似，思路是使用hash数组先处理magazine字符串， 然后对ransomNote进行验证
  const hashArr = Array(26).fill(0)
  const baseASCII = 'a'.charCodeAt(0)

  for (let s of magazine) {
    hashArr[s.charCodeAt(0) - baseASCII]++ // 遍历magazine字符串，收集hash结果
  }

  for (let s of ransomNote) {
    hashArr[s.charCodeAt(0) - baseASCII]--
    if (hashArr[s.charCodeAt(0) - baseASCII] < 0) {
      return false // 若出现hash-map中不存在的字符， 则返回false
    }
  }

  return true
};
```


**题目7，数组交集**：
给定两个数组 nums1 和 nums2 ，返回 它们的交集 。输出结果中的每个元素一定是 唯一 的。我们可以 不考虑输出结果的顺序 。

```typescript
function intersection(nums1: number[], nums2: number[]): number[] {
  // 思路：用set1先处理第一个数组入set
  // 然后遍历数组2， 来将存在set1的值加入结果set，最后返回结果set的数组化
  const set1: Set<number> = new Set(nums1)
  const setRes: Set<number> = new Set()
  for (const s of nums2) {
    if (set1.has(s)) {
      setRes.add(s)
    }
  }
  return Array.from(setRes)
};
```


**题目8，找到字符串中所有字母异位词**：
给定两个字符串 s 和 p，找到 s 中所有 p 的 异位词 的子串，返回这些子串的起始索引。不考虑答案输出的顺序。
示例
```typescript
  输入: s = "cbaebabacd", p = "abc"
  输出: [0,6]
  解释:
  起始索引等于 0 的子串是 "cba", 它是 "abc" 的异位词。
  起始索引等于 6 的子串是 "bac", 它是 "abc" 的异位词。
```
```typescript
  function findAnagrams(s: string, p: string): number[] {
    // 思路，两个字符串转数组的hash table

    const res = [], base = 'a'.charCodeAt(0)
    const [n, m] = [s.length, p.length]
    const arr1 = new Array(26).fill(0)
    const arr2 = new Array(26).fill(0)

    if (n < m) {
      return res
    }

    // 先收集第一个子串的长度下标入两个hash table
    for (let i = 0; i < m; i++) {
      arr1[s[i].charCodeAt(0) - base]++
      arr2[p[i].charCodeAt(0) - base]++
    }

    if (arr1.toString() === arr2.toString()) { // 收集第一项
        res.push(0)
    }

    // 处理后续下标长度，处理第一个hash table，处理方式是：每次去除第一位旧的字符，然后加入新的字符入hash
    for (let i = m; i < n; i++) {
      arr1[s[i - m].charCodeAt(0) - base]--
      arr1[s[i].charCodeAt(0) - base]++

      if (arr1.toString() === arr2.toString()) {
        res.push(i - m + 1) // 收集可行的开始下标
      }
    }

    return res
  };
```

