---
layout:     post
title:      "数据结构复习笔记（一）"
subtitle:   " Lesson 1 C实现单链表"
date:       2016-05-02 22:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - List
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
马上就要面临找工作了，准备陆陆续续将数据结构、计算机网络、操作系统这些基础给复习一遍。在复习的过程中，我会不断的在博客里进行总结，一方面也起到督促的作用。

## 0x02 单链表
今天复习的是数据结构里面最基础的数据结构——单链表，用C语言实现，废话不多说，直接上代码。

单链表头文件list.h

```
/* list.h --简单列表类型的头文件 */
#ifndef LIST_H_
#define LIST_H_

#include<stdbool.h>

#define TSIZE 45
struct film
{
    char title[TSIZE];
    int rating;
};

typedef struct film Item;

typedef struct node
{
    Item item;
    struct node *next;
}Node;

typedef Node *List;

/* 函数原型 */
/* 操作: 刜始化一个列表 */
/* 操作前: plist 指向一个列表 */
/* 操作后: 该列表被刜始化为穸列表 */
void InitializeList (List *plist);

/* 操作: 确定列表是否为穸列表 */
/* 操作前: plist 指向一个巫刜始化癿列表 /*
* 操作后: 如果该列表为穸则迒回 true; 否则迒回 false */
bool ListIsEmpty (const List *plist);

/* 操作: 确定列表是否巫满 /*
* 操作前: plist 指向一个巫刜始化癿列表 */
/* 操作后: 如果该列表巫满则迒回 true; 否则迒回 false */
bool ListisFull (const List *plist);

/* 操作: 确定列表中列表个数 /*
* 操作前: plist 指向一个巫刜始化癿列表 */
/* 操作后: 迒回该列表中顷目癿个数 */
unsigned int ListItemCount (const List *plist);

/* 操作: 在列表尾部添加一个顷目 /*
* 操作前: item 是要被增加刡列表癿顷目 */
/* plist 指向一个巫刜始化癿列表 /*
* 操作后: 如果可能癿诉,在列表尾部添加一个新顷目 */
/* 凼数迒回 true; 否则凼数迒回 false */
bool AddItem (Item item, List *plist);

/* 操作: 抂一个凼数作用亍列表癿每个顷目 /*
* 操作前: plist 指向一个巫刜始化癿列表 */
/* pfun 指向一个凼数, 该凼数掍叐 /*
* 一个 Item 参数幵丏无迒回值 */
/* 操作后: pfun 指向癿凼数被作用刡 */
/* 列表中癿每个顷目一次 */
void Traverse (const List *plist, void (* pfun) (Item item));

/* 操作: 释放巫分配癿内存 (如果有) */
/* 操作前: plist 指向一个巫刜始化癿列表 /*
* 操作后: 为该列表分配癿巫被释放 幵丏该列表被置为穸列表 */
void EmptytheList (List *plist);
#endif //
```

链表操作相关函数原型的实现放在list.c文件中

```
/* list.c --支持列表操作的函数 */
#include<stdio.h>
#include<stdlib.h>
#include "list.h"

static void CopyToNode(Item item, Node *pnode);

void InitializeList(List *plist)
{
    *plist = NULL;
}

bool ListIsEmpty(const List *plist)
{
    if(*plist == NULL)
        return true;
    else
        return false;
}

bool ListIsFull(const List *plist)
{
    Node *pt;
    bool full;

    pt = (Node *)malloc(sizeof(Node));
    if(pt == NULL)
        full = true;
    else
        full = false;
    free(pt);
    return full;
}

unsigned int ListItemCount(const List *plist)
{
    unsigned int count = 0;
    Node *pnode = *plist;

    while(pnode!=NULL)
    {
        ++count;
        pnode = pnode->next;
    }
    return count;
}

bool AddItem(Item item, List *plist)
{
    Node *pnew;
    Node *scan = *plist;

    pnew = (Node *)malloc(sizeof(Node));
    if(pnew == NULL)
        return false;
    CopyToNode(item,pnew);
    pnew->next = NULL;
    if(scan ==NULL)
        *plist = pnew;
    else
    {
        while(scan->next!=NULL)
            scan = scan->next;
        scan->next = pnew;
    }
    return true;
}

void Traverse(const List *plist,void(* pfun)(Item item))
{
    Node *pnode = *plist;
    while(pnode!=NULL)
    {
        (*pfun)(pnode->item);
        pnode = pnode->next;
    }
}

void EmptytheList(List *plist)
{
    Node *psave;
    while(*plist!=NULL)
    {
        psave = (*plist)->next;
        free(*plist);
        *plist = psave;
    }
}

static void CopyToNode(Item item, Node *pnode)
{
    pnode->item = item;
}

```

测试链表，test.c

```
/* test.c --使用ADT风格的链表 */
#include<stdio.h>
#include<stdlib.h>
#include "list.c"

void showmovies(Item item);

int main(void)
{
    List movies;
    Item temp;

    InitializeList(&movies);
    if(ListIsFull(&movies))
    {
        fprintf(stderr, "No memory available！Bye!\n");
        exit(1);
    }
    puts("Enter first movie title:");
    while(gets(temp.title)!=NULL && temp.title[0]!='\0')
    {
        puts("Enter your rating<0-10>:");
        scanf("%d", temp.rating);
        while(getchar()!='\n')
            continue;
        if(AddItem(temp,&movies)==false)
        {
            fprintf(stderr,"Problem allocating memory\n");
            break;
        }
        if(ListIsFull(&movies))
        {
            puts("The list is now full.");
            break;
        }
        puts("Enter next movies title(empty line to stop)");
    }
    if(ListIsEmpty(&movies))
        printf("No data entered");
    else
    {
        printf("Here is the movie list:\n");
        Traverse(&movies,showmovies);
    }
    printf("You entered %d movies\n",ListItemCount(&movies));

    EmptytheList(&movies);
    printf("Bye!\n");
    return 0;
}

void showmovies(Item item)
{
    printf("Movie:%s Rating: %d\n", item.title, item.rating);
}

```