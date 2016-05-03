---
layout:     post
title:      "数据结构之单链表（三）"
subtitle:   " Lesson 3 单链表常见算法"
date:       2016-05-02 23:30:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - List
    - C language
    - Algorithm
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
在[上一篇：数据结构之单链表（二）](/2016/05/02/data-structure-study-2/)中,我们在单链表上实现插入排序、选择排序和快速排序
这三种经典排序算法。本片博客中，我们将继续深入，探讨一些单链表相关的题目。主要内容如下：

- 链表倒置
- 求链表的第K个节点
- 求链表的倒数第K个节点，如果K大于链表长度则返回NULL
- 求单链表的中间结点，偶数个结点时返回中间两个节点的前一个
- 求单链表的中间结点，偶数个结点时返回中间两个节点的前一个
- 判断单链表是否有环
- 判断单链表是否有环，无环则返回NULL，有环则计算出环的开始节点以及环长


## 0x02 链表倒置
没什么好说的，直接上代码。

{% highlight c %}
/*链表倒置
*/
void reverse(LinkList *list)
{
    if(NULL == list->head)
        return;
    PNode current = list->head->next;
    PNode tmp = NULL;
    list->head->next = NULL;
    while(current)
    {
        tmp = current->next;
        current->next = list->head;
        list->head = current;
        current = tmp;
    }
}
{% endhighlight %}

## 0x03 求链表的第K个节点
同样没什么好说的，上代码。

{% highlight c %}
/*求链表的第K个节点
*/
Node* KthNode(const LinkList *list, int k)
{
    if(NULL == list->head || k<=1)
        return NULL;
    PNode scan = list->head;
    while(scan && k>1)
    {
        scan = scan->next;
        k--;
    }
    return scan;

}
{% endhighlight %}

## 0x04 求链表的倒数第K个节点
现在是求链表的倒数第K个节点，如果K大于链表长度则返回NULL。

{% highlight c %}
/*求链表的倒数第K个节点，如果K大于链表长度则返回NULL
*/
Node* backKth(const LinkList *list, int k)
{
    if(NULL == list->head || k<=0)
        return NULL;
    PNode pk = list->head;
    while(NULL == pk || k>1)
    {
        pk = pk->next;
        k--;
    }
    if(NULL == pk)
        return NULL;

    PNode p = list->head;
    while(NULL != pk->next)
    {
        pk = pk->next;
        p = p->next;
    }
    return p;
}
{% endhighlight %}

## 0x05 求单链表的中间结点
这里分两种情况，分别是偶数个结点时返回中间两个节点的前一个和后一个，主要区别在于指针前进的顺序，在代码中自行体会：

{% highlight c %}
/*
*求单链表的中间结点，偶数个结点时返回中间两个节点的前一个
*/

Node* findMid(const LinkList *list)
{
    if(NULL == list->head || NULL == list->head->next)
        return list->head;
    PNode slow = list->head;
    PNode fast = list->head;
    while(NULL != fast->next && NULL != fast->next->next)
    {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;
}

/*求单链表的中间结点，偶数个结点时返回中间两个节点的前一个
*/
Node *findMid2(const LinkList *list)
{
    if(NULL == list->head || NULL == list->head->next)
        return list->head;
    PNode slow = list->head;
    PNode fast = list->head;
    while(NULL != fast->next)
    {
        slow = slow->next;
        fast = fast->next;
        if(NULL == fast->next)
            break;
        else
            fast = fast->next;
    }
    return slow;
}
{% endhighlight %}

## 0x06 单链表是否有环问题
解决该问题的核心思想是使用快慢指针，很巧妙。

算法描述：
首先，我们判断单链表是否有环，这里我们也借用了步长法，即指针p1，p2都指向链表头，然后开始遍历，p1每次移动一步，p2每次移动
两步，然后判断p2有没有遇到p1。如果遇到了p1，说明链表有环；如果遇到之前，p2就已经到达链表尾部（值为NULL），说明链表没有环。
算法正确性证明：

1. 如果链表无环，我们的算法是能得到正确的结果的；
2. 这里我们考虑链表有环的情况。按照算法的流程，当慢指针p1，到达环开始节点A时，此时快节点必定在环中的某个节点，我们假设为B。
这里我们只是随便假设B而已，B可能在环中的任意位置，也有可能就是节点A（那样的话，算法就直接得到结果了）。我们假设指针以顺时针方
向在环中移动，B距离A的长度为y个节点，可以认为B处的快指针p2落后于A处的快指针p1的距离为y（0 <= y < LengthOfCircle）个节点。
现在开始赛跑，每轮循环快指针p2向前跑2个节点，慢指针p1向前跑1个节点，两者综合起来的效果就是快指针和慢指针的距离减少1个节点，
因此经过y轮循环之后，两指针将相遇，且相遇点为A向前y个节点的M点。因此证明了上述算法是正确的。

首先判断单链表是否有环：

{% highlight c %}
/*
*判断单链表是否有环
*/
bool isHasCircle(const LinkList *list)
{
    if(NULL == list->head || NULL == list->head->next)
        return false;
    PNode slow = list->head;
    PNode fast = list->head;
    while(NULL != fast->next)
    {
        fast = fast->next;
        if(NULL == fast->next)
            return false;
        fast = fast->next;
        slow = slow->next;
        if(fast == slow)
            return true;
    }
    return false;
}
{% endhighlight %}

接下来，如果单链表有环则要求出环的开始节点和环长。在前面的证明中，我们是从慢指针到达A点开始证明的，而忽略了前面从链表头
到A点的x个距离。现在假设慢指针从链表头head到达A点时，快指针p2已经移动到了B点，设环长为s个节点，那么有关系式：2x = x + n * s + s - y。
即x = (n+1)s–y。当慢指针p1和快指针p2同时到达M点时，即到达前面证明的最后时，我们再增加下面一些操作：另起一个指针p从链表
头head开始移动，每次移动一个节点，同样慢指针p1也继续从M点向前移动。当p经过x步到达A时，p1也经过n*s + s - y到达A点，和p相遇，
这样我们就找到了A点。在此过程中，我们可以同时记下p1反复经过M点的次数n，然后利用s = (x + y)/ (n + 1)计算环的长度。注意，我
们无法直接单独得到x或y的值，但却能统计从head开始到慢指针p1和快指针p2第一次在M点相遇时经过的循环次数x + y。具体实现如下：

{% highlight c %}
/*
*求出有环单链表的环开始节点和环长
*/

Node* findCircleFirst(const LinkList *list, int nCircleLen)
{
    PNode slow = list->head;
    PNode fast = list->head;
    int xy = 0; //x+y的值

    while(NULL != fast->next)
    {
        fast = fast->next;
        if(NULL == fast->next)
        {
            return NULL;
        }
        fast = fast->next;
        slow = slow->next;
        xy++;

        if(fast == slow)
        {
            PNode p = list->head;
            PNode meet = fast;
            int n = 0;
            while(p!=slow)
            {
                p = p->next;
                slow = slow->next;
                if(slow == meet)
                    n++;
            }

            nCircleLen = xy/(n+1);
            return p;
        }

    }
    return NULL;
}
{% endhighlight %}

## 0x07 总结
最后将上一篇中的排序代码和本篇中的代码进行合并，展现给大家

{% highlight c %}
/*
* list.c --单链表排序及常见算法
*/
#include<stdio.h>

#include<stdlib.h>

#include<stdbool.h>

typedef int ElemType;
typedef struct Node
{
    ElemType val;
    struct Node *next;
}Node,*PNode;

typedef struct
{
    PNode head;
    int size;
}LinkList;

void initList(LinkList *list)
{
    list->head = NULL;
    list->size = 0;
}

LinkList createList(ElemType A[], int count)
{
    if(NULL==A)
        return;
    LinkList list;
    initList(&list);

    int i;
    PNode first = (PNode)malloc(sizeof(Node));
    first->val = A[0];
    first->next = NULL;
    list.head = first;
    PNode scan = first;
    for(i=1;i<count;i++)
    {
        scan->next = (PNode)malloc(sizeof(Node));
        scan->next->val = A[i];
        scan->next->next = NULL;
        scan = scan->next;
    }
    list.size = count;
    return list;
}

void swap(PNode a, PNode b)
{
    if(a->val == b->val)
        return;
    int tmp = a->val;
    a->val = b->val;
    b->val = tmp;
}

void showList(const LinkList *list)
{
    PNode scan = list->head;
    while(scan)
    {
        printf("%d ", scan->val);
        scan = scan->next;
    }
    printf("\n");

}

/*
插入排序
*/
void insertSort(LinkList *list)
{
    if(NULL==list->head || NULL == list->head->next)
        return;
    PNode current = list->head->next;
    PNode tmp = NULL;
    list->head->next = NULL;

    while(NULL != current)
    {
        if(current->val < list->head->val)
        {
            tmp = current->next;
            current->next = list->head;
            list->head = current;
            current = tmp;
        }
        else
        {
            PNode p = list->head;
            while(NULL != p->next)
            {
                if(current->val < p->next->val)
                {
                    tmp = current->next;
                    current->next = p->next;
                    p->next = current;
                    current = tmp;
                    break;
                }
                else
                {
                    p = p->next;
                }
            }
            if (NULL == p->next)
            {
                tmp = current->next;
                current->next = NULL;
                p->next = current;
                current = tmp;
            }
        }//end else
    }//end while(NULL != current)
}

/*
选择排序
*/
void selectSort(LinkList *list)
{
    PNode scan = NULL;
    PNode pminpre = NULL;
    Node L = {0, NULL};
    PNode Ltail = &L;
    while(NULL != list->head && NULL != list->head->next)
    {
        pminpre = list->head;
        scan = list->head->next;
        while(NULL != scan && NULL != scan->next)
        {
            if(scan->next->val < pminpre->next->val)
                pminpre = scan;
            scan = scan->next;
        }

        if(list->head->val <= pminpre->next->val)
        {
            Ltail->next = list->head;
            Ltail = Ltail->next;
            list->head = list->head->next;
        }
        else
        {
            Ltail->next = pminpre->next;
            Ltail = Ltail->next;
            pminpre->next = pminpre->next->next;
        }
    }

    Ltail = Ltail->next = list->head;
    Ltail->next = NULL;
    list->head = L.next;
}

/*
*快速排序
*/

Node* getSeperator(Node *pBegin, Node *pEnd)
{
    PNode p = pBegin;
    PNode q = pBegin->next;
    ElemType key =  pBegin->val;
    while(q != pEnd)
    {
        if(q->val < key)
        {
            p = p->next;
            swap(p,q);
        }
        q = q->next;
    }
    swap(pBegin, p);
    return p;
}

/*
快速排序
*/
void quickSort(Node *pBegin, Node *pEnd)
{
    if(pBegin != pEnd)
    {
        PNode separator = getSeperator(pBegin, pEnd);
        quickSort(pBegin, separator);
        quickSort(separator->next, pEnd);
    }
}

/*
链表倒置
*/
void reverse(LinkList *list)
{
    if(NULL == list->head)
        return;
    PNode current = list->head->next;
    PNode tmp = NULL;
    list->head->next = NULL;
    while(current)
    {
        tmp = current->next;
        current->next = list->head;
        list->head = current;
        current = tmp;
    }
}

/*
求链表的第K个节点
*/
Node* KthNode(const LinkList *list, int k)
{
    if(NULL == list->head || k<=1)
        return NULL;
    PNode scan = list->head;
    while(scan && k>1)
    {
        scan = scan->next;
        k--;
    }
    return scan;

}

/*
求链表的倒数第K个节点，如果K大于链表长度则返回NULL
*/
Node* backKth(const LinkList *list, int k)
{
    if(NULL == list->head || k<=0)
        return NULL;
    PNode pk = list->head;
    while(NULL == pk || k>1)
    {
        pk = pk->next;
        k--;
    }
    if(NULL == pk)
        return NULL;

    PNode p = list->head;
    while(NULL != pk->next)
    {
        pk = pk->next;
        p = p->next;
    }
    return p;
}

/*
求单链表的中间结点，偶数个结点时返回中间两个节点的前一个
*/

Node* findMid(const LinkList *list)
{
    if(NULL == list->head || NULL == list->head->next)
        return list->head;
    PNode slow = list->head;
    PNode fast = list->head;
    while(NULL != fast->next && NULL != fast->next->next)
    {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;
}

/*
求单链表的中间结点，偶数个结点时返回中间两个节点的前一个
*/
Node *findMid2(const LinkList *list)
{
    if(NULL == list->head || NULL == list->head->next)
        return list->head;
    PNode slow = list->head;
    PNode fast = list->head;
    while(NULL != fast->next)
    {
        slow = slow->next;
        fast = fast->next;
        if(NULL == fast->next)
            break;
        else
            fast = fast->next;
    }
    return slow;
}

/* 判断单链表是否有环 */
bool isHasCircle(const LinkList *list)
{
    if(NULL == list->head || NULL == list->head->next)
        return false;
    PNode slow = list->head;
    PNode fast = list->head;
    while(NULL != fast->next)
    {
        fast = fast->next;
        if(NULL == fast->next)
            return false;
        fast = fast->next;
        slow = slow->next;
        if(fast == slow)
            return true;
    }
    return false;
}

/*
计算出环的开始节点以及环长
*/

Node* findCircleFirst(const LinkList *list, int nCircleLen)
{
    PNode slow = list->head;
    PNode fast = list->head;
    int xy = 0; //x+y的值

    while(NULL != fast->next)
    {
        fast = fast->next;
        if(NULL == fast->next)
        {
            return NULL;
        }
        fast = fast->next;
        slow = slow->next;
        xy++;

        if(fast == slow)
        {
            PNode p = list->head;
            PNode meet = fast;
            int n = 0;
            while(p!=slow)
            {
                p = p->next;
                slow = slow->next;
                if(slow == meet)
                    n++;
            }

            nCircleLen = xy/(n+1);
            return p;
        }

    }
    return NULL;
}


int main(void)
{
    int a[6] = {7,2,1,3,5,6};
    LinkList list;
    list = createList(a, 6);
    printf("before sort:");
    showList(&list);
    PNode node = findMid(&list);
    printf("mid node is %d\n", node->val);
//   reverse(&list);
//   quickSort(list.head, NULL);
//   selectSort(&list);
//  insertSort(&list);
    printf("after insertSort:");
    showList(&list);

    return 0;

}
{% endhighlight %}