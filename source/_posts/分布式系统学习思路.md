---
title: 分布式系统学习思路
date: 2016-05-13 22:37:29
category: 分布式
tags:
  - 分布式
---

## 介绍

博主近段时间准备学习分布式系统相关的东西，本文整理了学习分布式系统的思路，此文还未经过实践，可能还需要不断调整，仅供参考。

分布式系统一般分为分布式K/V系统、分布式文件系统和分布式数据库等几个大类，在学习这几类系统的时候，需要掌握的知识或技能应该包括计算机基础知识、分布式算法和协议相关论文、分布式系统设计范型相关论文、开源的分布式系统案例以及造相关的轮子。

## 基础知识

根据博主目前经验来看，学习分布式系统首先要掌握以下基础知识：

*   OS相关
*   体系结构相关
*   Unix系统编程
*   Unix网络编程
*   并发编程
*   常用数据结构和算法

## 论文

分布式系统的论文主要分为两大方面，一方面是算法或者协议相关的论文，另一方面是系统设计相关的。

### 算法或者协议相关

*   Byzantine General

*   paxos

paxos made simple 和 paxos made live

*   raft
*   cap base
*   2pc 3pc
*   leases
*   acid
*   time and ordering

Time Clocks and the ordering of Events in a Distributed System

Virtual Time and Global States of Distributed System

Distributed Snapshots: Determining Global States of a Distributed System

*   mvcc
*   consensus
*   gossip
*   load balancing algorithms

### 系统设计范型相关

*   google file system
*   bigtable
*   mapreduce
*   chubby
*   spanner
*   dynamo
*   megastore
*   dremel
*   pregel
*   percolator
*   Sinfonia: A New Paradigm for Building Scalable Distributed Systems
*   google f1
*   Windows Azure Storage: A high available cloud storage service with strong consistency
*   facebook haystack

## 开源系统范例

主要分为分布式K/V，分布式文件系统和分布式数据库三个方面。

### 分布式K/V系统

*   redis cluster
*   tair

### 分布式文件系统

*   hdfs
*   ceph
*   swift
*   lustre
*   taobao filesystem

### 分布式数据库

*   clustrix
*   MemSQL
*   VoltDB

## 造轮子

可以自己造分布式K/V系统、分布式文件系统、分布式数据库系统的轮子，简单的，可以从分布式K/V系统开始。

## 整个思路

*   对于基础知识部分，有盲点就补
*   对于论文模块，常见的分布式算法、协议论文，经典的系统范型相关论文需要精读
*   对于开源系统模块，在学习完论文之后，每个部分精读一个系统的代码，其他的系统了解实现原理
*   对于造轮子，可以尽早开始，先按照自己思路来造，后面通过读论文、读开源系统来发现自己系统的缺陷，不断完善即可

最后，本文只是一个思路，很多东西还没有细化，需要不断完善。

## 参考文献

*   http://duanple.blog.163.com/blog/static/709717672011330101333271/
*   https://github.com/ty4z2008/Qix/blob/master/ds.md
*   http://blog.ivanyang.me/distributedsystem/2016/03/06/whatwetalkaboutwhenwetalkaboutds
