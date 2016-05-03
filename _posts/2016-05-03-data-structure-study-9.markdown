---
layout:     post
title:      "数据结构之图"
subtitle:   " Lesson 1 图的邻接矩阵实现"
date:       2016-05-02 23:55:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - Data Structure
    - Graph
    - C language
---

>  Talk is cheap, show me the code. ——Linus Torvalds

## 0x01 前言
俗话说的好“图样图森破”，但是在数据结构中，图可一点都不森破。记得我在[数据结构之二叉树（一）](/2016/05/02/data-structure-study-6/)
中说过，当年二叉树没怎么学好，图就更不用说了，这部分就记住了迪杰斯特拉和克鲁斯卡尔这两个名字，还不会写英文。那么，本篇博客
就会对图这一数据结构进行实现。

## 0x02 C语言实现
上代码，先干为敬！

graph.c
{% highlight c %}
/*以邻接表作为图的存储结构，实现连通无向图的深度优先和广度优先遍历。

以指定的结点作为起点，分别输出每种遍历下的结点访问序列。
*/
#include<stdio.h>

#include<stdlib.h>

#include<conio.h>

#include<math.h>

#define TRUE 1

#define FALSE 0

#define OK 1

#define ERROR 0

typedef int Status;

typedef struct node
{
    int elem;
    struct node *next;
}Node,*QNode;

typedef struct
{
    QNode front;
    QNode rear;
}Queue;

#define MAX 20
typedef struct ArcNode    //表节点

{
    int adjvex;             //该边所指向的顶点的位置

    struct ArcNode *nextarc;  //指向下一条边

}ArcNode;

typedef struct VNode        //头节点

{
    int data;               //顶点信息

    ArcNode *firstarc;      //指向第一条依附该节点的边的指针

}VNode,AdjList[MAX];

typedef struct
{
    AdjList vertices;        //头节点

    int vexnum;              //节点的个数

    int arcnum;              //边的条数

}Graph;

// ----------------------栈的操作----------------

Status initQueue(Queue *Q)
{
    Q->front = Q->rear = (QNode)malloc(sizeof(Node));
    if(!Q->front)
        exit(OVERFLOW);
    Q->front->next = NULL;
    return OK;
}

Status enQueue(Queue *Q, int e)
{
    QNode p = (QNode)malloc(sizeof(Node));
    if(!p)
        exit(OVERFLOW);
    p->elem = e;
    p->next = NULL;
    Q->rear->next = p;
    Q->rear = p;
    return OK;
}

Status deQueue(Queue *Q,int *e)
{
    if(Q->front == Q->rear)
        return ERROR;
    QNode p = Q->front->next;
    Q->front->next = p->next;
    if(Q->rear == p)
        Q->rear = Q->front;
    *e = p->elem;
    free(p);
    p=NULL;
    return OK;
}

Status QueueEmpty(Queue Q)
{
    return Q.front == Q.rear;
}

//--------------------图的操作-----------------

int locateVex(Graph *G, int v)   //返回节点V在图中的位置

{
    int i;
    for(i=0;i<G->vexnum;i++)
    {
        if(G->vertices[i].data == v)
            break;
    }
    if(i<G->vexnum)
        return i;
    return -1;
}

//以邻接表形式创建无向连通图G

Status createGraph(Graph *G)
{
    int m,n,i,j,k,v1,v2,flag=0;
    ArcNode *p1,*q1,*p,*q;
    printf("Please input the number of VNode:");
    scanf("%d", &m);
    printf("Please input the number of ArcNode:");
    scanf("%d", &n);
    G->vexnum = m;
    G->arcnum = n;
    for(i=0;i<G->vexnum;i++)
    {
        G->vertices[i].data = i+1;
        G->vertices[i].firstarc = NULL;
    }
    printf("Output the message of VNode:\n");
    for(i=0;i<G->vexnum;i++)
        printf("v%d\n", G->vertices[i].data);
    for(k=0;k<G->arcnum;++k)
    {
        printf("Please input the %d edge beginpoint and endpoint:", k+1);
        scanf("%d%d", &v1,&v2);
        i = locateVex(G, v1);
        j = locateVex(G, v2);
        if(i>=0&&j>=0)
        {
            ++flag;
            p = (ArcNode *)malloc(sizeof(ArcNode));
            p->adjvex = j;
            p->nextarc = NULL;
            if(!G->vertices[i].firstarc)
                G->vertices[i].firstarc = p;
            else
            {
                p1 = G->vertices[i].firstarc;
                while(p1->nextarc)
                    p1 = p1->nextarc;
                p1->nextarc = p;
            }

            q = (ArcNode *)malloc(sizeof(ArcNode));
            q->adjvex = i;
            q->nextarc = NULL;
            if(!G->vertices[j].firstarc)
                G->vertices[j].firstarc = q;
            else
            {
                q1 = G->vertices[j].firstarc;
                while(q1->nextarc)
                    q1 = q1->nextarc;
                q1->nextarc = q;
            }
        }
        else
        {
            printf("Not have this edge！\n");
            k = flag;
        }
    }
    printf("The Adjacency List is:\n");
    for(i=0;i<G->vexnum;i++)
    {
        printf("\t%d v%d->", i, G->vertices[i].data);
        p = G->vertices[i].firstarc;
        while(p->nextarc)
        {
            printf("%d->", p->adjvex);
            p = p->nextarc;
        }
        printf("%d\n", p->adjvex);
    }

    return OK;
}

//返回V的第一个邻接顶点

 int firstAdjVex(Graph G, int v)
 {
     if(G.vertices[v].firstarc)
        return G.vertices[v].firstarc->adjvex;
     else
        return -1;
 }

 //返回V中相对于w的下一个邻接顶点
 
 int nextAdjVex(Graph G, int v, int w)
{
    int flag = 0;
    ArcNode *p;
    p = G.vertices[v].firstarc;
    while(p)
    {
        if(p->adjvex == w)
        {
            flag = 1;
            break;
        }
        p = p->nextarc;
    }
    if(flag && p->nextarc)
        return p->nextarc->adjvex;
    else
        return -1;
}

//深度优先遍历

int visited[MAX];
void DFS(Graph G, int v)
{
    int w;
    visited[v] = TRUE;
    printf("v%d ", G.vertices[v].data);
    for(w=firstAdjVex(G, v);w>=0;w=nextAdjVex(G,v,w))
    {
        if(!visited[w])
            DFS(G,w);
    }
}

void DFSTraverse(Graph G)
{
    int v;
    for(v=0;v<G.vexnum;v++)
    {
        visited[v]=FALSE;
    }
    for(v=0;v<G.vexnum;v++)
    {
        if(!visited[v])
            DFS(G,v);
    }
}

//广度优先遍历

void BFSTraverse(Graph G)
{
    int v, v1,w;
    Queue q;
    for(v=0;v<G.vexnum;v++)
        visited[v]=FALSE;
    initQueue(&q);
    for(v=0;v<G.vexnum;v++)
    {
        if(!visited[v])
        {
            visited[v]=TRUE;
            printf("v%d ", G.vertices[v].data);
            enQueue(&q, v);
            while(!QueueEmpty(q))
            {
                deQueue(&q, &v1);
                for(w=firstAdjVex(G, v1);w>=0;w=nextAdjVex(G, v1, w))
                {
                    if(!visited[w])
                    {
                        visited[w]=TRUE;
                        printf("v%d ", G.vertices[w].data);
                        enQueue(&q, w);
                    }
                }
            }
        }
    }
}

int main(void)
{
    Graph G;
    createGraph(&G);
    printf("Depth first search:\n");
    DFSTraverse(G);
    printf("\nBreadth first search:\n");
    BFSTraverse(G);
    printf("\n");
    getch();
    return 0;
}



{% endhighlight %}

