---
title: 并查集学习笔记
author: sanenchen
date: 2025-08-14 09:28:00 +0800
categories: [数据结构, 算法, C++]
tags: [C++, UnionFind]
render_with_liquid: false
---

## 并查集

并查集是一种用于管理元素所属集合的数据结构，实现为一个森林，其中每棵树表示一个集合，树中的节点表示对应集合中的元素。  
顾名思义，并查集支持两种操作：  
`合并（Union）：合并两个元素所属集合（合并对应的树）`
`查询（Find）：查询某个元素所属集合（查询对应的树的根节点），这可以用于判断两个元素是否属于同一集合`
并查集在经过修改后可以支持单个元素的删除、移动；
使用动态开点线段树还可以实现可持久化并查集。

### 代码实现
## V1 版本

```c++
class UnionFind {
    unordered_map<int, int> parent;

public:
    // 初始化
    explicit UnionFind(vector<int> p) {
        for (int i = 0; i < p.size(); ++i) {
            parent[p[i]] = i;
        }
    }

    // 查找
    int find(int x) {
        while (parent[x] != x) {
            x = parent[x];
        }
        return x;
    }

    // 合并
    void union_n(int x, int y) {
        const int root_x = find(x);
        const int root_y = find(y);
        parent[root_y] = root_x;
    }
};
```

这里便会出现一个问题：如果这个合并操作很多，就会导致这个树很深，find 操作效率下降。
可以尝试 `按秩合并`
把个子矮的树 挂在个子高的树下

## V2 版本 按秩合并

```c++
// 合并
void union_n(int x, int y) {
    int root_x = find(x);
    const int root_y = find(y);

    // 按秩合并
    if (rank[root_x] > rank[root_y])
        parent[root_y] = root_x;
    else if (rank[root_x] < rank[root_y])
        parent[root_x] = root_y;
    else {
        parent[root_y] = root_x;
        rank[root_x] += 1;
    }
}
```

## V2.1 版本 按大小合并

```c++
// 合并
void union_n(int x, int y) {
    int root_x = find(x);
    const int root_y = find(y);

    // 按大小合并
    if (size[root_x] > size[root_y]) {
        parent[root_y] = root_x;
        size[root_x] += size[root_y];
    }
    if (size[root_x] < size[root_y]) {
        parent[root_x] = root_y;
        size[root_y] += size[root_x];
    }
}
```

## V3.0 路径压缩
``` c++
// 查找
int find(int x) {
    // 路径压缩
    if (parent[x] != x)
        parent[x] = find(parent[x]);
    return parent[x];
}
```
