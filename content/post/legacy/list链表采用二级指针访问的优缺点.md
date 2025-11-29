+++
date = '2016-03-01T11:00:34+08:00'
draft = false
title = '采用二级指针实现单链表操作 单链表翻转 删除单链表结点'

+++


* register_filesystem中的二级指针，刚开始没看懂。怒了，如果这个都没看懂，还搞什么C语言编程
* leetcode中的反转链表，二级指针操作巨好用
* 先上代码吧


```cpp
#include<stdio.h>
#include<stdlib.h>

typedef struct ListNode{
    int val;
    struct ListNode *next;
} ListNode;

void printList(ListNode *head){
    if(head){
        printf("%d\n", head->val);
        printList(head->next);
    }
}

void printList_r(ListNode *head){
    if(!head)
        return;
    if(head -> next)
        printList_r(head->next);
    printf("%d\n", head->val);
}

ListNode ** addToTail(ListNode **list){
    ListNode **p ;
    for( p = list; *p; p = & (*p)->next);
    return p;
}

void reverseList(ListNode **list){
    ListNode *head = NULL;
    ListNode **p = list;
    while(*p){
        ListNode *next = (*p)->next;
        (*p)->next = head;
        head = *p;
        *p = next;
    }
    *list = head;
}

/* 
 * delete first item whose val equals to val
 *
*/
void deleteNodeAll(ListNode **l, int val){

    while(*l){
        if((*l)->val == val){
            ListNode *tmp = (*l)->next;
            free(*l);
            *l = tmp;
            break;
        }
        l = &(*l)->next;
    }
}

//delete all the items whose val equals to val
void deleteNodeFirst(ListNode **l, int val){

    while(*l){
        if((*l)->val == val){
            ListNode *tmp = (*l)->next;
            free(*l);
            *l = tmp;
        }
        else
            l = &(*l)->next;
    }
}

int main()
{
    int a[] = {1, 2, 3, 2, 3 ,4};
    int len = sizeof(a) / sizeof(int);
    int i = 0;

    static ListNode *list;

    for( i = 0; i < len; i++)
    {
        ListNode *node = malloc(sizeof(struct ListNode));
        node->val = a[i];
        ListNode **p = addToTail(&list);
        *p = node;
    }

    printf("--------print list in sequence------------\n");
    printList(list);

    printf("--------print list after reversed------------------\n");
    reverseList(&list);
    printList(list);

    printf("--------delete first 1----------\n");
    deleteNodeFirst(&list, 1);
    printList(list);

    printf("--------delete first 3----------------\n");
    deleteNodeFirst(&list, 3);
    printList(list);

    printf("--------delete first 4-----------------\n");
    deleteNodeFirst(&list, 4);
    printList(list);

    printf("--------delete non-existed item-------------\n");
    deleteNodeFirst(&list, 4);
    printList(list);

    printf("--------delete the last item----------------\n");
    deleteNodeFirst(&list, 2);
    printList(list);

    printf("--------reverse an empty list---------------\n");
    reverseList(&list);
    printList(list);


}
```
