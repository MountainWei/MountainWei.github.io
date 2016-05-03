---
layout:     post
title:      "数据结构之单链表（二）"
subtitle:   " Lesson 2 单链表排序"
date:       2016-05-02 23:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - List
    - C language
    - Sort Algorithm
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
在[上一篇：数据结构之单链表（一）](/2016/05/03/data-structure-study-1/)中,我们用C语言实现了单链表的基本操作，本片博客将在
单链表的基础上继续深入，实现插入排序、选择排序和快速排序这三种经典排序算法。

链表排序和数组排序的思路类似，只是链表操作起来比较麻烦，因为不能随机访问，所以只能借助于类似于前置或后置插入，添加等概念来完成。

## 0x02 单链表实现
所有的代码都在list2.c中，首先是实现一个基本的单链表，为后面的排序操作打好基础。

{% highlight c %}
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
{% endhighlight %}

## 0x02 插入排序
第一个实现的是插入排序，

插入排序，它的基本想法是从第一个节点开始，每次将一个节点放到结果链表，并保证每个节点加入到结果链表前后，
结果链表都是有序的。每个节点在被链入结果链表时有三种情况：

- 该节点值比结果链表中的所有元素值都大，则将该节点追加到结果链表的最后；

- 该节点值比结果链表中的所有元素值都小，则将该节点插入到结果链表最前面；

- 该节点值在结果链表中处于中间位置，则将该节点插入到合适位置。

{% highlight c %}
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
{% endhighlight %}

## 0x03 选择排序
接下来是选则排序。选择排序的基本思想是每次从源链表中选取并移除一个最小的元素，追加到结果链表的尾部，直到源链表变空为止。
因此本算法的关键点在于如何从源链表中选取并移除一个最小的元素。

考虑到一般情况下，我们在移除链表中的某个元素时，需要知道它的前一个节点的指针，于是我们可以按照最小元素的出现位置分为两种
情况：最小元素是链表头部节点和最小元素不是链表头部节点。我们先找出链表中除头结点外的最小值节点，然后再和头节点的值比较，
然后进行处理。寻找除头结点外的最小值节点的代码可以很简洁，这也是为什么要分成这两部分处理的原因。代码如下：

{% highlight c %}
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
{% endhighlight %}

## 0x04 快速排序
最后是快速排序，指定两个指针p和q，这两个指针均往next方向移动，移动的过程中保持p之前的key都小于选定的key，p和q之间的key都
大于选定的key，那么当q走到末尾的时候便完成了一次支点的寻找。

{% highlight c %}
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

void quickSort(Node *pBegin, Node *pEnd)
{
    if(pBegin != pEnd)
    {
        PNode separator = getSeperator(pBegin, pEnd);
        quickSort(pBegin, separator);
        quickSort(separator->next, pEnd);
    }
}
{% endhighlight %}