---
layout:     post
title:      "数据结构之二叉树（三）"
subtitle:   " Lesson 3 二叉树遍历非递归实现"
date:       2016-05-02 23:50:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - BiTree
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
上一篇：[数据结构之二叉树（二）](/2016/05/02/data-structure-study-7/)中我实现了二叉树常见的14种操作，但是在学习二叉树的过程中，
还有一类问题经常会遇到，那就是

——“来来来，把二叉树的三种遍历实现一下”

——“好的！”

——“对了，刚忘记说，不能用递归。”

——“&#*@！……@#”

既然这样，那这篇博客就说一下二叉树遍历的非递归实现。

## 0x02 C语言实现
干了这碗代码！

bitree3.c

{% highlight c %}
/* 非递归实现二叉树的遍历 */
#include<stdio.h>

#include<stdlib.h>

#define OK 1

#define ERROR 0

#define TRUE 1

#define FALSE 0

#define OVERFLOW -1

#define STACK_INIT_SIZE 100

#define STACKINCREMENT 20

typedef int Status;

typedef char ElemType;

typedef struct bitnode
{
    ElemType data;
    struct bitnode *lchild;
    struct bitnode *rchild;
    unsigned int isOut;
}BiTNode, *BiTree;

typedef BiTree SElemType;

typedef struct stack
{
    SElemType *base;
    SElemType *top;
    int stacksize;
}Sqstack;

Status initStack(Sqstack *s);
Status destroyStack(Sqstack *s);
Status clearStack(Sqstack *s);
Status stackEmpty(Sqstack s);
int stackLength(Sqstack s);
Status getTop(Sqstack s, SElemType *e);
Status push(Sqstack *s, SElemType e);
Status pop(Sqstack *s, SElemType *e);

BiTree createBiTree(BiTree T);
Status preOrderRecursionTraverse(BiTree T,Status (*visit)(ElemType e));
Status preOrderNonRecursionTraverse(BiTree T, Status (*visit)(ElemType e));
Status inOrderRecursionTraverse(BiTree T, Status (*visit)(ElemType e));
Status inOrderNonRecursionTraverse(BiTree T, Status (*visit)(ElemType e));
Status postOrderRecursionTraverse(BiTree T, Status (*visit)(ElemType e));
Status postOrderNonRecursionTraverse(BiTree T, Status (*visit)(ElemType e));
Status visit(ElemType e);


int main()
{
    BiTree T=NULL;
    Status(*visit)(ElemType e)=visit;
    printf("请按先序遍历输入二叉树元素（每个结点一个字符，空结点为'#'）:\n");
    T=createBiTree(T);
    printf("\n递归先序遍历：\n");
    preOrderRecursionTraverse(T,visit);
    printf("\n递归中序遍历：\n");
    inOrderRecursionTraverse(T,visit);
    printf("\n递归后序遍历：\n");
    postOrderRecursionTraverse(T,visit);
    printf("\n非递归先序遍历：\n");
    preOrderNonRecursionTraverse(T,visit);
    printf("\n非递归中序遍历：\n");
    inOrderNonRecursionTraverse(T,visit);
    printf("\n非递归后序遍历：\n");
    postOrderNonRecursionTraverse(T,visit);
    printf("\nEnd of main.\n");
    return 0;
}



BiTree createBiTree(BiTree T)
{
    char ch;
    scanf("%c", &ch);
    if(ch=='#')
        T=NULL;
    else
    {
        if(!(T=(BiTNode *)malloc(sizeof(BiTNode))))
            exit(OVERFLOW);
        T->data = ch;
        T->lchild = createBiTree(T->lchild);
        T->rchild = createBiTree(T->rchild);
    }
    return T;
}

Status preOrderRecursionTraverse(BiTree T, Status (*visit)(ElemType e))
{
    if(T)
    {
        if(!(visit(T->data)))
            return ERROR;
        preOrderRecursionTraverse(T->lchild, visit);
        preOrderRecursionTraverse(T->rchild, visit);
    }
    return OK;
}

Status inOrderRecursionTraverse(BiTree T, Status (*visit)(ElemType e))
{
    if(T)
    {
        inOrderRecursionTraverse(T->lchild, visit);
        if(!(visit(T->data)))
            return ERROR;
        inOrderRecursionTraverse(T->rchild, visit);
    }
    return OK;
}

Status postOrderRecursionTraverse(BiTree T, Status (*visit)(ElemType e))
{
    if(T)
    {
        postOrderRecursionTraverse(T->lchild, visit);
        postOrderRecursionTraverse(T->rchild, visit);
        if(!(visit(T->data)))
            return ERROR;
    }
    return OK;
}

Status preOrderNonRecursionTraverse(BiTree T, Status (*visit)(ElemType e))
{
    Sqstack S;
    SElemType p;
    initStack(&S);
    push(&S,T);
    while(!stackEmpty(S))
    {
        pop(&S, &p);
        if(!(visit(p->data)))
            return ERROR;
        if(p->rchild)
            push(&S, p->rchild);
        if(p->lchild)
            push(&S, p->lchild);
    }
    destroyStack(&S);

    return OK;
}

Status inOrderNonRecursionTraverse(BiTree T, Status (*visit)(ElemType e))
{
    Sqstack S;
    SElemType p;
    initStack(&S);
    p = T;
    while(p||stackEmpty(S))
    {
        if(p)
        {
            push(&S,p);
            p = p->lchild;
        }
        else
        {
            pop(&S, &p);
            if(!visit(p->data))
                return ERROR;
            p = p->rchild;
        }
    }
    destroyStack(&S);
    return OK;
}

Status postOrderNonRecursionTraverse(BiTree T, Status (*visit)(ElemType e))
{
    Sqstack S;
    SElemType p,q;
    initStack(&S);
    push(&S,T);
    while(!stackEmpty(S))
    {
        while(getTop(S,&p) && p)
            push(&S, p->lchild);
        pop(&S,&p); //空指针出栈
        getTop(S, &p);
        if(p->rchild)
        {
            push(&S, p->rchild);
            continue;
        }
        if(!stackEmpty(S))
        {
            pop(&S, &p);
            if(!visit(p->data))
                return ERROR;
            while(getTop(S, &q) && q && p==q->rchild)
            {
                pop(&S, &p);
                if(!visit(p->data))
                    return ERROR;
            }
            getTop(S, &p);
            if(p->rchild)
            {
                push(&S, p->rchild);
                continue;
            }
            else
            {
                pop(&S, &p);
                if(!visit(p->data))
                    return ERROR;
            }
        }
    }
    destroyStack(&S);
    return OK;
}

Status visit(ElemType e)
{
    if(e=='\0')
        return ERROR;
    else
        printf("%c",e);
    return OK;
}


//-------------顺序栈操作-----------------//

Status initStack(Sqstack *S)
{
    S->base = (SElemType *)malloc(STACK_INIT_SIZE*sizeof(SElemType));
    if(!S->base)
    {
        printf("分配内存失败.\n");
        exit(0);
    }
    S->top = S->base;
    S->stacksize = STACK_INIT_SIZE;
    return OK;
}

Status destroyStack(Sqstack *S)
{
    if(!S)
    {
        printf("指针为空，释放失败.\n");
        exit(0);
    }
    free(S->base);
    return OK;
}

Status clearStack(Sqstack *S)
{
    if(!S)
        return FALSE;
    S->top = S->base;
    return OK;
}

Status stackEmpty(Sqstack S)
{
    if(S.top ==S.base)
        return TRUE;
    else
        return FALSE;
}

int StackLength(Sqstack S)
{
    return S.stacksize;
}

Status getTop(Sqstack S, SElemType *e)
{
    if(S.top == S.base)
        return FALSE;
    else
    {
        *e = *(S.top-1);
        return OK;
    }
}

Status push(Sqstack *S, SElemType e)
{
    if(S->top - S->base >= S->stacksize)
    {
        S->base = (SElemType *)realloc(S->base,(S->stacksize+STACKINCREMENT)*sizeof(SElemType));
        if(!S->base)
        {
            printf("重新申请空间失败。\n");
            exit(0);
        }
        S->top = S->base+S->stacksize;
        S->stacksize+=STACKINCREMENT;
    }
    *S->top++ = e;
}

Status pop(Sqstack *S, SElemType *e)
{
    if(S->top == S->base)
        return ERROR;
    *e = *(--S->top);
    return OK;
}
{% endhighlight %}

