---
title: 在 C 语言中实现 ArrayList
date: 2022-01-20 20:56:53
tags: C
---

最近在读[《妙趣横生的算法(C语言实现)》](https://book.douban.com/subject/4710825/)中关于有序序列的章节，其中有List 相关内容的实现。
我在此基础上实现了 `IntList`，并且使用面向对象的思想增加了相关对列表操作的接口。下面是其实现及测试代码。

<!--more-->

```c
#ifndef ARRAY_LIST_H
#define ARRAY_LIST_H
#include <limits.h>
#include <stdlib.h>
#define MaxSize 10

typedef struct
{
  int *elem;
  int length;
  int capacity;
  void (*toString) (void *);
  void (*add)(void*,int data);
  void (*insertAt) (void *, int position, int data);
  void (*deleteAt) (void *, int position);
} IntList;

void printArrayList (void *data);
void insertAt (void *list, int i, int data);
void add(void *list,int data);
void deleteAt (void *list, int i);
void
initIntList (IntList * arrayList)
{
  arrayList->elem = malloc (MaxSize * sizeof (int));
  if (!arrayList->elem)
    {
      exit (0);
    }
  arrayList->length = 0;
  arrayList->capacity = MaxSize;
  arrayList->toString = printArrayList;
  arrayList->insertAt = insertAt;
  arrayList->add = add;
  arrayList->deleteAt = deleteAt;
}

IntList *
obtainIntList ()
{
  IntList *list = (IntList *) malloc (sizeof (IntList));
  initIntList (list);
  return list;
}

void add(void *out,int data){
    IntList *list = (IntList*)out;
    list->insertAt(out,list->length,data);
}

void
insertAt (void *out, int i, int data)
{
  IntList *list = (IntList *) out;
  int *base, *insertPtr, *p;
  if (i < 0 || i > list->length)
    {
      exit (0);
    }
  if (list->length >= list->capacity)
    {
      base = realloc (list->elem, (list->capacity + 10) * sizeof (int));
      list->elem = base;
      list->capacity = list->capacity + 10;
    }
  insertPtr = &(list->elem[i]);
  for (p = &(list->elem[list->length - 1]); p >= insertPtr; p--)
    {
      *(p + 1) = *p;
    }
  *insertPtr = data;
  list->length++;
}

void
deleteAt (void *out, int position)
{
  IntList *list = (IntList *) out;
  int *deleteItem,*q;
  if (position < 0 || position > list->length - 1)
    {
      exit (0);
    }
  deleteItem = &(list->elem[position]);
  q = list->elem + list->length - 1;
  for (deleteItem; deleteItem < q; ++deleteItem)
    {
      *(deleteItem) = *(deleteItem + 1);
    }
  list->length--;
}

void
printArrayList (void *data)
{
  IntList *list = (IntList *) data;
  printf ("[");
  for (int i = 0; i < list->length; i++)
    {
      if (i != list->length - 1)
  {
    printf ("%d,", list->elem[i]);
  }
      else
  {
    printf ("%d", list->elem[i]);
  }

    }
  printf ("]\n");
}

#endif

```

```c
#include <stdio.h>
#include "IntList.h"
int
main ()
{
  IntList *list = obtainIntList ();
  printf ("init an empty int list\n");
  list->toString (list);
  for (int i = 0; i < 5; i++)
    {
      list->insertAt (list, i, i);
    }
  printf ("add 5 item to list\n");
  list->toString (list);
  for (int i = 5; i < 10; i++)
    {
      list->add (list, i);
    }
  printf ("add 5 item to list\n");
  list->toString (list);
  printf ("delete the item at index of 2\n");
  list->deleteAt (list, 2);
  list->toString (list);
  free (list);
  return 0;
}

```

一下是运行结果：
```
init an empty int list
[]
add 5 item to list
[0,1,2,3,4]
add 5 item to list
[0,1,2,3,4,5,6,7,8,9]
delete the item at index of 2
[0,1,3,4,5,6,7,8,9]
```