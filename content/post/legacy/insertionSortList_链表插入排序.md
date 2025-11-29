+++
date = '2016-06-05T22:27:39+08:00'
draft = true
title = 'insertionSortList_链表插入排序'

+++

写go并发代码有点疲劳，那就A个题吧。我觉得LintCode上的题目都很不错，有难有易，循序渐进的，随手选一道题来做吧。

选择了[链表插入排序](http://www.lintcode.com/zh-cn/problem/insertion-sort-list/)，写起来不那么容易，但是也有调试的快感，其实对于难题，我们需要做到的是心思缜密、抽丝剥茧一步步实现；对于中等题目，我们需要抓住本质，一击制敌；在编码过程中，心里有时刻有代码执行时间复杂度和空间占用的底，逐渐养成这个习惯；编码过程中同样重要的还有编码风格，如果能够简洁明了的表达程序逻辑，就追求简洁，如果变量较多，需要在变量命名的时候注意甄别，对于写出的代码，尽量追求逻辑清晰风格良好，这是一个程序员的基本素质。

> 用插入排序对链表排序

思路：插入排序的特点是，待排序的节点前面所有节点都是已排序的，同时记录已排序部分的末尾，也就是待排序节点的prev节点，所以每次拿带排序节点依次从头比较，直到比较到自己头上；在比较过程中，如果没有合适的位置，最终比较到自己头上，说明待排序节点可以直接补在已排序节点的后面；如果有合适的位置，就直接break，然后待排序节点从链表取出，prev节点的next指向待排序节点的next，保存当前break处的节点，待排序节点占据该节点位置，并将待排序节点的next指向break处节点，完成插入。

[完整代码地址](https://github.com/BG2BKK/daily-programming/blob/master/cpp/insertionSortList.cpp)

* codelist

```cpp
/**
 * Definition of ListNode
 * class ListNode {
 * public:
 *     int val;
 *     ListNode *next;
 *     ListNode(int val) {
 *         this->val = val;
 *         this->next = NULL;
 *     }
 * }
 */
class Solution { 
	public:
		ListNode *insertionSortList(ListNode *head) {
			// write your code here
			if(!head || !head->next)
				return head;
			ListNode *sortedHead = head;
			ListNode *prev = head;
			ListNode *node = prev->next;
			while(node){
				ListNode **p = &sortedHead;
				while(*p!=node){
					if((*p)->val > node->val){
						break;
					}
					p = &((*p)->next);
				}
				if(*p == node){
					prev = node;
					node = node->next;
				} else {
					ListNode *next = *p;
					*p = node;
					prev->next = node->next;
					(*p)->next = next;
					node = prev->next;
				}
			}
			return sortedHead;
		}
};
```
