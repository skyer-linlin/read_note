<!--
 * @Author: your name
 * @Date: 2020-06-10 16:18:49
 * @LastEditTime: 2020-06-11 10:18:01
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: \undefinedh:\note\java基础\泛型\spring\read note\blog_note\为什么 MySQL 使用 B+ 树.md
-->

# 为什么 MySQL 使用 B+ 树

原文链接

> [为什么 MySQL 使用 B+ 树](https://draveness.me/whys-the-design-mysql-b-plus-tree/)

## 概述

主键索引和辅助索引都会以 B+树的形式存储

- 主键索引，存储形式：`<id, row>`
- 辅助索引, 存储形式: `<index, id>`

## 设计

- OLAP: Online Transaction Processing
- OLTP: Online Analytical Processing

### 数据加载

计算机在读写文件的时候, 以页为单位将数据加载到内存中, 页的大小一般为 4K, 通过`getconf PAGE_SIZE`查看

哈希虽然能够提供 O(1) 的单数据行操作性能，但是对于范围查询和排序却无法很好地支持，最终导致全表扫描

**B 树和 B+树的区别是: B 树可以再非叶结点中存储数据, B+树所有数据都存储在叶子节点中**
