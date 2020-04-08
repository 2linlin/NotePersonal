---
title: leetcode题解
date: 2020-04-08 00:00:00
categories:
- 算法
tags:
- 算法
top:
---



### 数据结构

#### [1. 两数之和](https://leetcode-cn.com/problems/two-sum/)

给定一个整数数组 `nums` 和一个目标值 `target`，请你在该数组中找出和为目标值的那 **两个** 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

**示例:**

```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

**题解：**

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        for (int i = 0; i < nums.length; i++) {
            for (int j = i + 1; j < nums.length; j++) {
                if (nums[i] + nums[j] == target) {
                    return new int[] {i,j};
                }
            }
        }
        throw new Illegal

    }
}
```

#### [724. 寻找数组的中心索引](https://leetcode-cn.com/problems/find-pivot-index/)

给定一个整数类型的数组 `nums`，请编写一个能够返回数组**“中心索引”**的方法。

我们是这样定义数组**中心索引**的：数组中心索引的左侧所有元素相加的和等于右侧所有元素相加的和。

如果数组不存在中心索引，那么我们应该返回 -1。如果数组有多个中心索引，那么我们应该返回最靠近左边的那一个。

**示例 1:**

```
输入: 
nums = [1, 7, 3, 6, 5, 6]
输出: 3
解释: 
索引3 (nums[3] = 6) 的左侧数之和(1 + 7 + 3 = 11)，与右侧数之和(5 + 6 = 11)相等。
同时, 3 也是第一个符合要求的中心索引。
```

**示例 2:**

```
输入: 
nums = [1, 2, 3]
输出: -1
解释: 
数组中不存在满足此条件的中心索引。
```

**说明:**

- `nums` 的长度范围为 `[0, 10000]`。
- 任何一个 `nums[i]` 将会是一个范围在 `[-1000, 1000]`的整数。

**题解：**

```java
class Solution {
    public int pivotIndex(int[] nums) {
        // 直接使用暴力法求解
        // 边界考虑：最大的和为 5000 * 1000 = 5000000。用int存储即可。
        for (int i = 0; i < nums.length; i++) {
            // 1.i左边所有数相加
            int leftAdd = 0;
            for (int j = 0; j < i; j++) {
                leftAdd = leftAdd + nums[j];
            }
            // 2.i右边所有数相加
            int rightAdd = 0;
            for (int k = i + 1; k < nums.length; k++) {
                rightAdd = rightAdd + nums[k];
            }
            // 3.判断是否相等，相等则返回nums[i]的值
            if (leftAdd == rightAdd) {
                return i;
            }
        }
        // 没有则返回-1
        return -1;
    }
}
```

#### [747. 至少是其他数字两倍的最大数](https://leetcode-cn.com/problems/largest-number-at-least-twice-of-others/)

在一个给定的数组`nums`中，总是存在一个最大元素 。

查找数组中的最大元素是否至少是数组中每个其他数字的两倍。

如果是，则返回最大元素的索引，否则返回-1。

**示例 1:**

```
输入: nums = [3, 6, 1, 0]
输出: 1
解释: 6是最大的整数, 对于数组中的其他整数,
6大于数组中其他元素的两倍。6的索引是1, 所以我们返回1.
```

 

**示例 2:**

```
输入: nums = [1, 2, 3, 4]
输出: -1
解释: 4没有超过3的两倍大, 所以我们返回 -1.
```

 

**提示:**

1. `nums` 的长度范围在`[1, 50]`.
2. 每个 `nums[i]` 的整数范围在 `[0, 100]`

**题解：**

```java
class Solution {
    public int dominantIndex(int[] nums) {
        // 0.长度小于2的话，直接返回。根据题意，数组只有一个元素也是合法输入。
        if (nums.length < 2) {
            return 0;
        }
        // 1.先备份原始数组，以便后续用于匹配索引
        HashMap<Integer, Integer> orgMap = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            orgMap.put(nums[i], i);
        }
        // 2.先将数组进行排序
        Arrays.sort(nums);
        // 3.查看最大数是否是第二大数的两倍，是则用该数到原始数组中查看它的原始索引，并返回索引
        if (nums[nums.length-1] >=  nums[nums.length-2]*2) {
            return orgMap.get(nums[nums.length-1]);
        }
        // 4.否则返回-1
        return -1;
    }
}
```

#### [66. 加一](https://leetcode-cn.com/problems/plus-one/)

难度简单454收藏分享切换为英文关注反馈

给定一个由**整数**组成的**非空**数组所表示的非负整数，在该数的基础上加一。

最高位数字存放在数组的首位， 数组中每个元素只存储**单个**数字。

你可以假设除了整数 0 之外，这个整数不会以零开头。

**示例 1:**

```
输入: [1,2,3]
输出: [1,2,4]
解释: 输入数组表示数字 123。
```

**示例 2:**

```
输入: [4,3,2,1]
输出: [4,3,2,2]
解释: 输入数组表示数字 4321。
```

**题解：**

一开始想到的是，把数字从数据拿出来，加一，再放回去。但是这种解法，数字过大时会有整形溢出的问题：

```java
// 错误的解法
class Solution {
    public int[] plusOne(int[] digits) {
        // 1.把数字从数组里拿出来
        int num = 0;
            
        for (int i = digits.length; i > 0 ; i--) {
            num = num + (int)(digits[i-1] * Math.pow(10, digits.length - i));
        }
        
        // 2.加1
        num++;
        
        // 3.把数据放回数组
        List<Integer> arrayList = new ArrayList<>();
        
        // 十位数及以上
        while(num > 0) {
            arrayList.add(num % 10);
            num = num / 10;
        }
        
        
        // 转为数组
        int[] array = new int[arrayList.size()];
        for (int i = 0; i < arrayList.size(); i++) {
            array[arrayList.size() - i - 1] = arrayList.get(i);
        }
        
        return array;
    }
}
```

后来改为直接用数组实现，只需判断进位到哪里结束就行了。这种解法反而简单点。最终解法：

```java
class Solution {
    public int[] plusOne(int[] digits) {
        // 1.加1
        digits[digits.length - 1]++;

        // 2.进位：数字不全为9的情况
        for (int i = digits.length - 1; i > 0; i--) {
            if (digits[i] == 10) {
                digits[i] = 0;
                digits[i - 1]++;
            }
        }

        // 3.进位：数字全为9的情况
        if (digits[0] == 10) {
            int[] result = new int[digits.length + 1];
            result[0] = 1;
            return result;
        }

        return digits;
    }
}
```

