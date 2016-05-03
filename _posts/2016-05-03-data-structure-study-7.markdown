---
layout:     post
title:      "数据结构之二叉树（二）"
subtitle:   " Lesson 2 二叉树实现改进"
date:       2016-05-02 23:48:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - BiTree
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
上一篇：[数据结构之二叉树（一）](/2016/05/02/data-structure-study-6/)中我介绍了二叉树的基本实现，实现过程中指针的使用不太熟练，
实现的功能也比较优先，只有三种递归遍历和求深度。要学好二叉树，这些是远远不够的。因此，本篇博客中我将重新实现二叉树，对二叉树的
常见14种操作都进行实现，代码中的指针使用也更加规范。下面就一起来看看吧。

## 0x02 C语言实现
千言万语都在代码中，上代码！

bitree2.c

{% highlight c %}
/* 改进版的二叉树，对指针及变量的使用更加规范）*/
#include<stdio.h>

#include<stdlib.h>

typedef char ElemItem;

typedef struct TreeNode
{
    ElemItem item;
    struct TreeNode *lchild;
    struct TreeNode *rchild;
}BTNode, *BiTree;

void createBTree(BiTree *root)
{
    ElemItem ch;
    ch = getchar();
    if(ch=='#')
        *root = NULL;
    else
    {
        *root = (BTNode *)malloc(sizeof(BTNode));
        (*root)->item = ch;
        (*root)->lchild = NULL;
        (*root)->rchild = NULL;
        printf("Enter %c's lchild(# means no child):",ch);
        CreateBTree(&(*root)->lchild);
        printf("Enter %c's rchild(# means no child):", ch);
        CreateBTree(&(*root)->rchild);
    }
}

void preOrder(const BiTree *root)
{
    if(NULL == *root)
        return;
    printf("%-3c", (*root)->item);
    preOrder((*root)->lchild);
    preOrder((*root)->rchild);
}

void inOrder(const BiTree *root)
{
    if(NULL == *root)
        return;
    inOrder((*root)->lchild);
    printf("%-3c", (*root)->item);
    inOrder((*root)->rchild);
}

void postOrder(const BiTree *root)
{
    if(NULL == *root)
        return;
    postOrder((*root)->lchild);
    postOrder((*root)->rchild);
    printf("%-3c", (*root)->ch);
}

//输出叶子节点

void displayLeaf(const BiTree *root)
{
    if(NULL == root)
        return;
    if((*root)->lchild==NULL && (*root)->rchild == NULL)
        printf("%-3c", (*root)->item);
    else
    {
        displayleaf(&(*root)->lchild);
        displayleaf(&(*root)->rchild);
    }
}

//左结点插入

void inseartLeftNode(BiTree *root, ElemItem item)
{
    BTNode * p, newNode;
    if(NULL == *root)
        return;
    p = (*root)->lchild;
    newNode = (BTNode *)malloc(sizeof(BTNode));
    if(NULL == newNode)
        return;
    newNode->item = item;
    newNode->rchild = NULL;
    newNode->lchild = p;
    (*root)->lchild = newNode
}

//右结点插入

void inseartRightNode(BiTree *root, ElemItem item)
{
    BTNode *p, *newNode;
    if(NULL == *root)
        return;
    p = (*root)->rchild;
    newNode = (BTNode *)malloc(sizeof(BTNode));
    if(NULL == newNode)
        return;
    newNode->item = item;
    newNode->lchild = NULL;
    newNode->rchild = p;
    (*root)->rchild = newNode;
}

//销毁一棵二叉树

void clear(BiTree *root)
{
    BTNode *pl,*pr;
    if(NULL == *root)
        return;
    pl = (*root)->lchild;
    pr = (*root)->rchild;
    (*root)->lchild=NULL;
    (*root)->rchild=NULL;
    free(*root);
    *root=NULL;
    clear(&pl);
    clear(&pr);
}

//删除左子树

void deleteLeftTree(BiTree *root)
{
    if(NULL == *root)
        return;
    clear(&(*root)->lchild);
    root->lchild = NULL;
}

void deleteRightTree(BiTree *root)
{
    if(NULL == *root)
        return;
    clear(&(*root)->rchild);
    root->rchild = NULL;
}

//通过先序遍历查找结点

BTNode *search(const BiTree *root, ElemItem item)
{
    BTNode *p;
    if(NULL == *root)
        return NULL;
    if(item == (*root)->item)
        return *root;
    else
    {
        if((p=search(&(*root)->lchild, item))!=NULL)
            return p
        else
            return search(&(*root)->rchild, item);
    }
}

//求二叉树的高度

int BTreeHeight(const BiTree *root)
{
    int lchildHeight,rchildHeight;
    if(NULL == *root)
        return 0;
    lchildHeight = BTreeHeight(&(*root)->lchild);
    rchildHeight = BTreeHeight(&(*root)->rchild);
    return (lchildHeight>rchildHeight?(1+lchildHeight):(1+rchildHeight));
}

//求叶子结点的个数

int countLeaf(const BiTree *root)
{
    if(NULL = *root)
        return 0;
    if((*root)->lchild == NULL && (*root)->rchild ==NULL)
        return 1;
    else
    {
        return countLeaf(&(*root)->lchild)+countLeaf(&(*root)->rchild);
    }
}

//求所有结点个数

int countAll(const BiTree *root)
{
    if(NULL == *root)
        return 0;
    return countAll(&(*root)->lchild)+countAll(&(*root)->rchild)+1;
}

//复制二叉树

BitTree copyBTree(const BiTree *root)
{
    BTNode *p,*lchild, *rchild;
    if(NULL == *root)
        return NULL;
    lchild = copyBTree(&(*root)->lchild);
    rchild = copyBTree(&(*root)->rchild);
    p = (BTNode *)malloc(sizeof(BTNode));
    p->item = (*root)->item;
    p->lchild = lchild;
    p->rchild = rchild;
    return p;
}
{% endhighlight %}

