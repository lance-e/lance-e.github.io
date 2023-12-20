# Leetcode 86,23题 题解


<!--more-->

# leetcode 86，23

### leetcode 第86题：分隔链表

~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func partition(head *ListNode, x int) *ListNode {
    headNode1 := &ListNode{Val:-1,Next:nil}
    headNode2 := &ListNode{Val:-1,Next:nil}
    p1 := headNode1 // 小
    p2 := headNode2 // 大
    p := head
    for p != nil {
        if p.Val < x {
            p1.Next = p
            p1 = p1.Next
        }else {
            p2.Next = p 
            p2 = p2.Next
        }
        temp := p.Next
        p.Next = nil //把该结点断开
        p = temp

    }
    p1.Next = headNode2.Next
    return headNode1.Next
}
~~~

### leetcode 第23题：合并k个升序链表

最初的做法(最蠢的做法)：
先将所有的链表合并到一个链表中，再对该链表进行冒泡排序.
当然肯定有更优解法，但是！我就只有这点知识😭，所以只能想到这里，看到很多题解是使用优先级队列，所以等我补充完这里的知识就再重刷一遍

~~~go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeKLists(lists []*ListNode) *ListNode {
    if lists == nil{
        return nil
    }
    first := &ListNode{Val: -10000, Next: nil}
	p := first
	for _, link := range lists {
		p.Next = link
		for p.Next != nil {
			p = p.Next

		}
	}
	if first.Next == nil{
        return first.Next
    }
	Sorted := false

	for !Sorted {
		current := first 
		Sorted = true
		for current.Next != nil {
			if current.Val > current.Next.Val {
				current.Val, current.Next.Val = current.Next.Val, current.Val
				Sorted = false
			}
			current = current.Next
		}
		if Sorted {
			return first.Next
		}

	}
	return nil
}
~~~

