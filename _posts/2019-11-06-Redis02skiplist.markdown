---
layout:     post
title:      "Redis02 skiplist"
subtitle:   "skiplist"
date:       2019-11-06 21:22:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - DataBase
    - Redis
---

```c++
typedef struct zskiplistNode {
    sds ele;  // 存储的数据
    double score;  // 用于排序的分值
    struct zskiplistNode *backward;  // 后向指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;  // forward指向的节点和本节点之间的节点数，注意相邻节点span为1
    } level[];  // 层级数组
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

skiplist是zset的内部数据结构，接口不是很多。

```c++
/* Create a skiplist node with the specified number of levels.
 * The SDS string 'ele' is referenced by the node after the call. */
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;  // ele归这个节点了
    return zn;
}

/* Create a new skiplist. */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);  // 头节点拥有所有层级
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

插入节点比较复杂，和普通的链表插入类似，先找到位置，再设置各种参数。比较难的地方在于前面寻找位置的时候，就记下每一层前一个节点和新插入节点每层rank，减少后续重复遍历

```c++
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {  //自顶向下
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];  //累计排名
        // 层级i，如果有前向节点，而且比前向节点排名更后
        // 排名更后是指，分数更高，或者同分且字符串字母序更高
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;  // 累计forward跳过的节点数
            x = x->level[i].forward;  // 我跳
        }
        update[i] = x;  // 记录下本层需要更新的节点，就是本层在新插入节点前面的节点，或者是该层最后一个节点
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    level = zslRandomLevel();  // 新插入节点的层级是随机而来的
    if (level > zsl->level) {  // 新插入节点的层级比当前skiplist层级要高
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;  // 准备更新header
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {  // 从下往上更新，因为这样span才正确
        x->level[i].forward = update[i]->level[i].forward;  // 插入节点
        update[i]->level[i].forward = x;  // update[i]是层级i中，新插入节点前一个节点

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

删除节点，

```c++
/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele). */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;  // 和插入的类似，记录每层需要删除节点的前一个节点，或者是该层最后一个节点
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);  // 修正指针/span/level/...
        if (!node)
            zslFreeNode(x);  // 删除sds和node本身
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```

zslUpdateScore更新节点分数，更新后还要看一下是否需要更改节点位置，如果需要的话就比较麻烦了，由于这里的skiplist实现不支持直接插入已分配好的节点，所以只能先删除，再插入。代码略去。

skiplist有一部分接口是针对字典序的，不过它们不会遍历所有节点，只是针对连续节点，一般是用于分值一样或者连续的有序集合。这部分其实就是忽略分值，默认集合是字典序的。

```
> zadd sortset 3 a 4 d 5 b 6 f 7 g

> zrange sortset 0 -1
1) "a"
2) "d"
3) "b"
4) "f"
5) "g"

> zrangebylex sortset [b (g
1) "d"
2) "b"
3) "f"

```

元素较少（默认64）且插入元素的字符串长度较短，zset会用ziplist实现，否则用的是dict和skiplist