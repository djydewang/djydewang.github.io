---
layout: post
title:  "postgresql之页面布局"
date:   2019-05-15 16:55:10
categories: postgresql
---

## 页面布局概况
page相关的定义：[src/include/storage/bufpage.h](https://github.com/postgres/postgres/blob/master/src/include/storage/bufpage.h)
页面的管理信息存储在PageHeadData中
```
typedef struct PageHeaderData
{
	/* XXX LSN is member of *any* block, not only page-organized ones */
	PageXLogRecPtr pd_lsn;		/* LSN: next byte after last byte of xlog
								 * record for last change to this page */
	uint16		pd_checksum;	/* checksum */
	uint16		pd_flags;		/* flag bits, see below */
	LocationIndex pd_lower;		/* offset to start of free space */
	LocationIndex pd_upper;		/* offset to end of free space */
	LocationIndex pd_special;	/* offset to start of special space */
	uint16		pd_pagesize_version;
	TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
	ItemIdData	pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;
```

#### pd_lsn 4B
>   WAL records are appended to the WAL logs as each new record is written. 
The insert position is described by a Log Sequence Number (LSN) that is a byte 
offset into the logs, increasing monotonically with each new record.
LSN values are returned as the datatype pg_lsn.
Values can be compared to calculate the volume of WAL data that separates them,
so they are used to measure the progress of replication and recovery.

>   The LSN is used by the buffer manager to enforce the basic rule of WAL:
    "thou shalt write xlog before data".  A dirty buffer cannot be dumped
    to disk until xlog has been flushed at least as far as the page's LSN.

page的pg_lsn用来定位上一次对该page的修改的日志记录，若要将page缓存写入硬盘，前提为将相应日志记录已经写入硬盘。

#### pd_checksum 2B
页面校验和
>  pd_checksum stores the page checksum, if it has been set for this page;
   zero is a valid value for a checksum. If a checksum is not in use then
   we leave the field unset. This will typically mean the field is zero
   though non-zero values may also be present if databases have been
   pg_upgraded from releases prior to 9.3, when the same byte offset was
   used to store the current timelineid when the page was last updated.
   Note that there is no indication on a page as to whether the checksum
   is valid or not, a deliberate design choice which avoids the problem
   of relying on the page contents to decide whether to verify it. Hence
   there are no flag bits relating to checksums.
#### pd_flags 2B
> pd_flags contains the following flag bits.  Undefined bits are initialized
 to zero and may be used in the future.

> PD_HAS_FREE_LINES is set if there are any LP_UNUSED line pointers before
 pd_lower.  This should be considered a hint rather than the truth, since
 changes to it are not WAL-logged.

> PD_PAGE_FULL is set if an UPDATE doesn't find enough free space in the
 page for its new tuple version; this suggests that a prune is needed.
 Again, this is just a hint.
#### pd_lower 2B
空闲空间起始位置偏移量
#### pd_upper 2B
空闲空间末端偏移量
#### pd_sepcial 2B
特殊空间的偏移量，一般来说当页面存的是表内容时，这部分空间是无用的，当页面存存储B+树索引时，该空间用来指向节点的左右兄弟。
#### pd_pagesize_version 2B
>  The page version number and page size are packed together into a single
   uint16 field.  This is for historical reasons: before PostgreSQL 7.3,
   there was no concept of a page version number, and doing it this way
   lets us pretend that pre-7.3 databases have page version number zero.
   We constrain page sizes to be multiples of 256, leaving the low eight
   bits available for a version number.
#### pd_linp
ItemIdData定义
```
typedef struct ItemIdData
{
	unsigned	lp_off:15,		/* offset to tuple (from start of page) */
				lp_flags:2,		/* state of line pointer, see below */
				lp_len:15;		/* byte length of tuple */
} ItemIdData;

typedef ItemIdData *ItemId;

/*
 * lp_flags has these possible states.  An UNUSED line pointer is available
 * for immediate re-use, the other states are not.
 */
#define LP_UNUSED		0		/* unused (should always have lp_len=0) */
#define LP_NORMAL		1		/* used (should always have lp_len>0) */
#define LP_REDIRECT		2		/* HOT redirect (should have lp_len=0) */
#define LP_DEAD			3		/* dead, may or may not have storage */
```
他是用来表示一个item的起始位置和长度以及标记。对照实际的页面布局
```
  +----------------+---------------------------------+
  | PageHeaderData | linp1 linp2 linp3 ...           |
  +-----------+----+---------------------------------+
  | ... linpN |                                      |
  +-----------+--------------------------------------+
  |           ^ pd_lower                             |
  |                                                  |
  |          v pd_upper                              |
  +-------------+------------------------------------+
  |          | tupleN ...                            |
  +-------------+------------------+-----------------+
  |	   ... tuple3 tuple2 tuple1| "special space" |
  +--------------------------------+-----------------+
```
在PageHeaderData后紧接着item指针数组，指示各个item的存放位置。可以发现页面的剩余空间，item是从末端向前延伸，
item指针是从头向后延伸。此处需注意，linp1不一定是指向tuple1的。
#### item
item一般泛指存储在page中的数据项，它可以是关系元组，也可以是各种索引的索引项。
具体分析下关系元组的组织结构
一个关系元祖项起始的23字节为元祖头，里面记录着和事务操作相关的记录及存放元组数据的偏移量，仅接着是指示各个属性是否为null的bitmap，
而后是Object ID(若元祖头中的相应标志位被设置了)，接着就是正真的元组数据了。关于如何解析这些数据，一般是需要从pg_attribute表中读取该关系的
所有属性字典，即每个属性的长度（若不是变长），序号，存储对齐方式等等。



