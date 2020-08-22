---
layout: post
title: 从零开始的搜索引擎（更新中） 
date: 2020-08-18 15:05:00 +0800
tags: [search engine, updating]
---

* MVP
  * 分词
    * 去标点
    * 字典
  * 关键词库
    * tokenId -> token
  * 文档库
    * docId -> doc
  * 倒排索引
    * tokenId -> docId[]
  * 查询
    * or tokens
    * and tokens
      * 归并多个docId[]
* 可用性改进
  * 结果高亮  
    * 分词偏移量
  * 查询结果排序
  * 数据热更新
    * 文档
      * 去重
      * 删除标记
    * 关键词
* 改进
  * 数据持久化
    * 海量数据
      * 索引和数据分离
        * B+
        * LSM
      * 数据合并
  * 数据去重
    * bloom filter
  * 索引更新
  * 查询
    * 权重排序
    * 结果缓存
    * 热度排序
      * 结果反馈
    * 纠错
    * 相关推荐
  * 更好的分词
    * 混合粒度

### 参考资料
