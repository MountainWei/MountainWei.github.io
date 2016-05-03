---
layout:     post
title:      "数据结构之栈"
subtitle:   " Lesson 1 C实现栈"
date:       2016-05-02 23:40:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - Stack
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
紧跟在队列后面的数据结构就是栈啦，同队列一样，栈也是线性表的一种变体，只能在同一端插入和删除，从而具有先入后出的特性。栈在计算机世界
中应用广泛，大家最熟悉的栈应用非函数调用莫属，因此，理解栈是理解函数调用过程的基础，而理解函数调用过程是掌握编程的基础。可以说，掌握栈的原理
是一个程序员的基本素质。

## 0x02 C语言实现
栈的这部分代码也是我在C Primer Plus(第五版)上摘抄下来的，收获颇多。一般而言，栈的实现有两种，一种是使用数组存储元素，实现简单，
但是栈的大小不能动态改变；另一种是使用链表存储元素，实现复杂，但能动态增加栈的大小。本片博客中使用链表来存储元素。具体代码如下：

{% highlight c %}
/*链表实现栈结构 */
#include<stdio.h>

#include<stdlib.h>

#include<stdbool.h>

typedef int Item;
typedef struct node
{
    Item data;
    struct node *next;
} NODE, *PNODE;

typedef struct stack
{
    PNODE pTop;
    PNODE pBottom;
}STACK,*PSTACK;

//函数声明

void initStack(PSTACK pStack);
void pushStack(PSTACK pStack,Item item);
bool popStack(PSTACK pstack,Item *pItem);
void traverseStack(const PSTACK pStack);
bool isEmpty(const PSTACK pStack);
void clearStack(PSTACK pStack);
int main(void)
{
    STACK stack;
    Item item;
    initStack(&stack);
    pushStack(&stack,10);
    pushStack(&stack,20);
    pushStack(&stack,30);
    pushStack(&stack,50);
    traverseStack(&stack);
    if(popStack(&stack,&item))
        printf("success pop stack,the item's value is %d\n", item);
    else
        printf("pop stack failed.\n");
    clearStack(&stack);
    traverseStack(&stack);
    system("PAUSE");
    return 0;
}

void initStack(PSTACK pStack)
{
    pStack->pTop = (PNODE)malloc(sizeof(NODE));
    if(NULL!=pStack->pTop)
    {
        pStack->pBottom = pStack->pTop;
        pStack->pTop->next = NULL;
    }
    else
    {
        printf("内存分配失败，程序退出.\n");
        exit(1);
    }
    return;
}

void pushStack(PSTACK pStack, Item item)
{
    PNODE pNew = (PNODE)malloc(sizeof(NODE));
    if(NULL == pNew)
    {
        printf("内存分配失败，程序退出.\n");
        exit(-1);
    }
    pNew->data = item;
    pNew->next = pStack->pTop;
    pStack->pTop = pNew;
    return;
}

bool popStack(PSTACK pStack, Item *pItem)
{
    if(isEmpty(pStack))
        return false;
    else
    {
        PNODE temp = pStack->pTop;
        *pItem = temp->data;
        pStack->pTop = temp->next;
        free(temp);
        temp = NULL;
        return true;
    }
}

void traverseStack(const PSTACK pStack)
{
    PNODE pNode = pStack->pTop;
    while(pStack->pBottom!=pNode)
    {
        printf("%d   ", pNode->data);
        pNode = pNode->next;
    }
    printf("\n");
    return;
}

bool isEmpty(const PSTACK pStack)
{
    if(pStack->pTop == pStack->pBottom)
        return true;
    else
        return false;
}

void clearStack(PSTACK pStack)
{
    if(isEmpty(pStack))
        return;
    else
    {
        PNODE p = pStack->pTop;
        PNODE temp = NULL;
        while(p!=pStack->pBottom)
        {
            temp = p->next;
            free(p);
            p = temp;
        }
        pStack->pTop = pStack->pBottom;
        return;
    }
}
{% endhighlight %}

