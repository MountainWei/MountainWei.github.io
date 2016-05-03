---
layout:     post
title:      "数据结构之二叉树（一）"
subtitle:   " Lesson 1 C实现基本二叉树"
date:       2016-05-02 23:45:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - BiTree
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
介绍完线性表、队列和栈后，接下来就轮到二叉树了。记得当年本科学数据结构的时候，对二叉树就一直不太理解，后来直到考研重新学习
了一遍数据结构后，才对二叉树的基本原理和操作有了比较深入的理解。不同于队列、栈这类一维的结构（我自己取的，因为从抽象来看，它们
在空间中呈现给我们是一条线，有木有……），二叉树则将线性表扩展到了二维，同时又具有很多变体，如平衡二叉树、红黑树、B+树等等，
综上所述，是时候来好好复习一下二叉树。

## 0x02 C语言实现
本篇博客首先会给出一个基本的二叉树实现，实现了三种遍历和求深度这些最基本操作，下面直接看代码吧。

bitree1.c

{% highlight c %}
/*第一个二叉树实现，功能简单，指针使用不熟练 */
#include<stdio.h>

#include<stdlib.h>

typedef int ElemType;

typedef struct BiTNode
{
    ElemType data;
    struct BiTNode *lchild;
    struct BiTNode *rchild;
}BiTNode,*BiTree;

int CreateBiTree(BiTree *T)
{
    ElemType ch;

    scanf("%d", &ch);
    while(getchar()!='\n')
        continue;

    if(-1==ch)
        *T = NULL;
    else
    {
        *T = (BiTNode *)malloc(sizeof(BiTNode));
        if(T==NULL)
        {
            fprintf(stderr, "Not enough memory to create bitree.\n");
            exit(-1);
        }
        (*T)->data = ch;
        printf("enter %d's lchild:", ch);
        CreateBiTree(&((*T)->lchild));
        printf("enter %d's rchild:", ch);
        CreateBiTree(&((*T)->rchild));
    }

    return 1;
}

//先序遍历二叉树
void TraverseBiTree(const BiTree *T)
{
    if(NULL == *T)
        return;
    printf("%d ", (*T)->data);
    TraverseBiTree(&((*T)->lchild));
    TraverseBiTree(&((*T)->rchild));
}

//中序遍历二叉树
void InOrderBiTree(const BiTree *T)
{
    if(NULL == *T)
        return;
    InOrderBiTree(&((*T)->lchild));
    printf("%d ", (*T)->data);
    InOrderBiTree(&((*T)->rchild));
}

//后序遍历二叉树
void PostOrderBiTree(const BiTree *T)
{
    if(NULL == *T)
        return;
    PostOrderBiTree(&((*T)->lchild));
    PostOrderBiTree(&((*T)->rchild));
    printf("%d ", (*T)->data);
}

//求二叉树的深度
int TreeDeep(const BiTree *T)
{
    int deep = 0;
    if(*T)
    {
        int leftdeep = TreeDeep(&((*T)->lchild));
        int rightdeep = TreeDeep(&((*T)->rchild));
        deep = leftdeep>=rightdeep?leftdeep+1:rightdeep+1;
    }
    return deep;
}

//求二叉树叶子结点个数
int Leafcount(const BiTree T, int *num)
{
    if(T)
    {
        if(T->lchild == NULL && T->rchild == NULL)
            (*num)++;
        Leafcount(T->lchild, num);
        Leafcount(T->rchild, num);
    }
    return *num;
}

int main(void)
{
    BiTree T;
    BiTree *p = (BiTree *)malloc(sizeof(BiTree));
    int deepth,num = 0;
    printf("请输入第一个结点的值，-1表示没有叶子结点:\n");
    CreateBiTree(&T);
    printf("先序遍历二叉树：");
    TraverseBiTree(&T);
    printf("\n");
    printf("中序遍历二叉树：");
    InOrderBiTree(&T);
    printf("\n");
    printf("后序遍历二叉树：");
    PostOrderBiTree(&T);
    printf("\n");

    printf("\n");
    deepth=TreeDeep(&T);
    printf("树的深度为:%d",deepth);
    printf("\n");
    Leafcount(T,&num);
    printf("树的叶子结点个数为:%d",num);
    printf("\n");

    return 0;
}
{% endhighlight %}

