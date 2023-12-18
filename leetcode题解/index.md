# Leetcode 1，21题 题解


<!--more-->

# leetcode 1，21
### leetcode 第一题：求两数之和
##### 解法一：

暴力枚举：
~~~go
func twoSum(nums []int, target int) []int {
  for i:=0;i<len(nums)-1;i++{
    for n:=i+1;n<len(nums);n++{
      if nums[i]+nums[n]==target{
        return []int{i,n}
      }
    }
  }
  return nil
}
~~~

##### 解法二：
哈希表：

~~~go
func twoSum(nums []int, target int) []int {
  hashTable := make(map[int]int)
  for i , v := range nums{
    if j,ok :=hashTable[target-v];ok{
      return []int{i,j}
    }
    hashTable[v]=i;
  }
  return nil;
}
~~~~~

哈希表使用O(1)的时间查询到了O(n)的信息
暴力枚举使用O(1)的时间只查询到了O(1)的信息
所以哈希表的时间复杂度是O(n),暴力枚举的时间复杂度是O(n^2)

### leetcode 第二十一题：合并两个有序链表

~~~go
/**
 \* Definition for singly-linked list.
 \* type ListNode struct {
 \*   Val int
 \*   Next *ListNode
 \* }
 */
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
  HeadNode := &ListNode{
    Val :-1,
    Next :nil,
  }
  p := HeadNode
  for list1 != nil && list2 != nil{
    if list1.Val < list2.Val{
      p.Next = list1
      list1 = list1.Next
    }else {
      p.Next = list2
      list2 = list2.Next
    }
    p = p.Next
  }
  if list1 != nil{
    p.Next = list1
  }
  if list2 != nil{
    p.Next = list2
  }
  return HeadNode.Next
}

~~~

先声明了一个虚拟头结点，方便后续操作，也可以不用额外处理指针为空的情况
然后不断比较两个链表，将小的插入到新的链表，并不断让新链表向前移动
###### Tip(取自labuladong的算法小抄)：
什么时候需要虚拟头结点？
当创建新的链表时，可以使用虚拟头结点来简化边界情况的处理。

