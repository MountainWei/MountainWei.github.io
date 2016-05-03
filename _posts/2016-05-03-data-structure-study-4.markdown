---
layout:     post
title:      "数据结构之队列"
subtitle:   " Lesson 1 C实现队列"
date:       2016-05-02 23:35:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - Queue
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
复习完链表后，接下来就是队列了。其实队列也是一种特殊的线性表，对插入和删除作了限制，只允许从一端插入，另一端删除，从而
具有了先入先出的特点。队列的应用广泛，最典型的就是消息队列机制，消息按照FIFO方式进行处理。好了，话不多说，看C语言的实现吧。

## 0x02 头文件——queue.h
这部分代码是我在C Primer Plus(第五版)上摘抄下来的，收获颇多。首先来看队列的头文件queue.h，里面定义了基本的队列结构体和操作接口。

{% highlight c %}
/* queue.h --队列接口 */
#include<stdbool.h>

#pragma C9x on

#ifndef _QUEUE_H_

#define _QUEUE_H_

typedef int Item;

#define MAXQUEUE 10
typedef struct node{
    Item item;
    struct node *next;
}Node;

typedef struct queue{
    Node *front;
    Node *rear;
    int items;
}Queue;

void InitializeQueue(Queue *pq);

bool QueueIsFull(const Queue *pq);

bool QueueIsEmpty(const Queue *pq);

int QueueItemCount(const Queue *pq);

bool EnQueue(Item item, Queue *pq);

bool DeQueue(Item *pitem, Queue *pq);

void EmptyTheQueue(Queue *pq);

#endif // _QUEUE_H_
{% endhighlight %}

## 0x03 逻辑实现——queue.c
头文件queue.h中定义的接口都会在queue.c中实现，代码如下：

{% highlight c %}
/*queue.c --队列类型的实现文件 */
#include<stdio.h>

#include<stdlib.h>

#include"queue.h"

static void CopyToNode(Item item, Node *pn);
static void CopyToItem(Node *pn, Item *pi);

void InitializeQueue(Queue *pq)
{
    pq->front = pq->rear = NULL;
    pq->items = 0;
}

bool QueueIsFull(const Queue *pq)
{
    return pq->items == MAXQUEUE;
}

bool QueueIsEmpty(const Queue *pq)
{
    return pq->items == 0;
}

int QueueItemCount(const Queue *pq)
{
    return pq->items;
}

bool EnQueue(Item item, Queue *pq)
{
    Node *pnew;
    if(QueueIsFull(pq))
        return false;
    pnew = (Node *)malloc(sizeof(Node));
    if(pnew == NULL)
    {
        fprintf(stderr, "unable to allocate memory\n");
        exit(1);
    }
    CopyToNode(item, pnew);
    pnew->next = NULL;

    if(QueueIsEmpty(pq))
        pq->front = pnew;
    else
        pq->rear->next = pnew;
    pq->rear = pnew;
    pq->items++;

    return true;
}

bool DeQueue(Item *pitem, Queue *pq)
{
    Node *pt;
    if(QueueIsEmpty(pq))
        return false;
    CopyToItem(pq->front, pitem);
    pt = pq->front;
    pq->front = pq->front->next;
    free(pt);
    pq->items--;
    if(pq->items==0)
        pq->rear = NULL;
    return true;
}

void EmptyTheQueue(Queue *pq)
{
    Item dummy;
    while(!QueueIsEmpty(pq))
        DeQueue(&dummy, pq);
}

static void CopyToNode(Item item, Node *pn)
{
    pn->item = item;
}

static void CopyToItem(Node * pn, Item *pi)
{
    *pi = pn->item;
}
{% endhighlight %}

## 0x04 测试Queue的驱动程序——use_q.c
最后，我们简单写一个测试Queue各个操作的驱动程序use_q.c，代码如下：

{% highlight c %}
/* use_q.c --测试Queue接口的驱动程序 */
#include<stdio.h>

#include "queue.c"

int main(void)
{
    Queue line;
    Item temp;
    char ch;

    InitializeQueue(&line);
    puts("testing the Queue interface.Type a to add a value,");
    puts("type d to delete a value, and type q to quit.");

    while((ch=getchar())!='q')
    {
        if(ch!='a'&&ch!='d')
            continue;
        if(ch=='a')
        {
            printf("Integer to add:");
            scanf("%d", &temp);
            if(!QueueIsFull(&line))
            {
                printf("Putting %d into queue\n", temp);
                EnQueue(temp, &line);
            }
            else
                puts("Queue is full!");
        }
        else
        {
            if(QueueIsEmpty(&line))
                puts("Nothing to delete!");
            else
            {
                DeQueue(&temp, &line);
                printf("Removing %d from queue\n", temp);
            }
        }
        printf("%d items is in queue\n", QueueItemCount(&line));
        puts("Type a to add,d to delete ,q to quie:");
    }
    EmptyTheQueue(&line);
    puts("Bye!");

    return 0;
}
{% endhighlight %}
