---
layout: post
title:  "postgresql由oid获取Relation对象简析"
date:   2019-05-18 21:40:10
categories: postgresql
---

### 前言
此文基于postgresql源码(version:devel 12)简单了解其内部表格存储的组织方式，对于不相关的细节留作以后深入理解。

### 源码解析
#### 入口 table.c table_open()
```
/* ----------------
 *		table_open - open a table relation by relation OID
 *
 *		This is essentially relation_open plus check that the relation
 *		is not an index nor a composite type.  (The caller should also
 *		check that it's not a view or foreign table before assuming it has
 *		storage.)
 * ----------------
 */
Relation
table_open(Oid relationId, LOCKMODE lockmode)
{
	Relation	r;

	r = relation_open(relationId, lockmode);

	if (r->rd_rel->relkind == RELKIND_INDEX ||
		r->rd_rel->relkind == RELKIND_PARTITIONED_INDEX)
		ereport(ERROR,
				(errcode(ERRCODE_WRONG_OBJECT_TYPE),
				 errmsg("\"%s\" is an index",
						RelationGetRelationName(r))));
	else if (r->rd_rel->relkind == RELKIND_COMPOSITE_TYPE)
		ereport(ERROR,
				(errcode(ERRCODE_WRONG_OBJECT_TYPE),
				 errmsg("\"%s\" is a composite type",
						RelationGetRelationName(r))));

	return r;
}
```
该函数主要有两个参数， 其中的lockmode指获取相应table所需要获取的锁，postgresql内置9种模式。
可以看到这里将主要工作交给了relation_open函数，table_open的主要作用为安全性检查，继续查看table_relation.
#### relation.c relation_open()
```
/* ----------------
 *		relation_open - open any relation by relation OID
 *
 *		If lockmode is not "NoLock", the specified kind of lock is
 *		obtained on the relation.  (Generally, NoLock should only be
 *		used if the caller knows it has some appropriate lock on the
 *		relation already.)
 *
 *		An error is raised if the relation does not exist.
 *
 *		NB: a "relation" is anything with a pg_class entry.  The caller is
 *		expected to check whether the relkind is something it can handle.
 * ----------------
 */
Relation
relation_open(Oid relationId, LOCKMODE lockmode)
{
	Relation	r;

	Assert(lockmode >= NoLock && lockmode < MAX_LOCKMODES);

	/* Get the lock before trying to open the relcache entry */
	if (lockmode != NoLock)
		LockRelationOid(relationId, lockmode);

	/* The relcache does all the real work... */
	r = RelationIdGetRelation(relationId);

	if (!RelationIsValid(r))
		elog(ERROR, "could not open relation with OID %u", relationId);

	/*
	 * If we didn't get the lock ourselves, assert that caller holds one,
	 * except in bootstrap mode where no locks are used.
	 */
	Assert(lockmode != NoLock ||
		   IsBootstrapProcessingMode() ||
		   CheckRelationLockedByMe(r, AccessShareLock, true));

	/* Make note that we've accessed a temporary relation */
	if (RelationUsesLocalBuffers(r))
		MyXactFlags |= XACT_FLAGS_ACCESSEDTEMPNAMESPACE;

	pgstat_initstats(r);

	return r;
}
```
核心为调用`RelationIdGetRelation()`及`pgstat_inistats()`
#### relcache.c RelationIdGetRelation()
```
/* ----------------------------------------------------------------
 *				 Relation Descriptor Lookup Interface
 * ----------------------------------------------------------------
 */

/*
 *		RelationIdGetRelation
 *
 *		Lookup a reldesc by OID; make one if not already in cache.
 *
 *		Returns NULL if no pg_class row could be found for the given relid
 *		(suggesting we are trying to access a just-deleted relation).
 *		Any other error is reported via elog.
 *
 *		NB: caller should already have at least AccessShareLock on the
 *		relation ID, else there are nasty race conditions.
 *
 *		NB: relation ref count is incremented, or set to 1 if new entry.
 *		Caller should eventually decrement count.  (Usually,
 *		that happens by calling RelationClose().)
 */
Relation
RelationIdGetRelation(Oid relationId)
{
	Relation	rd;

	/* Make sure we're in an xact, even if this ends up being a cache hit */
	Assert(IsTransactionState());

	/*
	 * first try to find reldesc in the cache
	 */
	RelationIdCacheLookup(relationId, rd);

	if (RelationIsValid(rd))
	{
		RelationIncrementReferenceCount(rd);
		/* revalidate cache entry if necessary */
		if (!rd->rd_isvalid)
		{
			/*
			 * Indexes only have a limited number of possible schema changes,
			 * and we don't want to use the full-blown procedure because it's
			 * a headache for indexes that reload itself depends on.
			 */
			if (rd->rd_rel->relkind == RELKIND_INDEX ||
				rd->rd_rel->relkind == RELKIND_PARTITIONED_INDEX)
				RelationReloadIndexInfo(rd);
			else
				RelationClearRelation(rd, true);

			/*
			 * Normally entries need to be valid here, but before the relcache
			 * has been initialized, not enough infrastructure exists to
			 * perform pg_class lookups. The structure of such entries doesn't
			 * change, but we still want to update the rd_rel entry. So
			 * rd_isvalid = false is left in place for a later lookup.
			 */
			Assert(rd->rd_isvalid ||
				   (rd->rd_isnailed && !criticalRelcachesBuilt));
		}
		return rd;
	}

	/*
	 * no reldesc in the cache, so have RelationBuildDesc() build one and add
	 * it.
	 */
	rd = RelationBuildDesc(relationId, true);
	if (RelationIsValid(rd))
		RelationIncrementReferenceCount(rd);
	return rd;
}
```
该函数涉及postgresql自己的缓存管理模块，由于我们需要了解table的存储组织结构，此处忽略缓存相关代码，
继续hack`RelationBuildDesc()`
#### relcache.c RelationBuildDesc()
```
/*
 *		RelationBuildDesc
 *
 *		Build a relation descriptor.  The caller must hold at least
 *		AccessShareLock on the target relid.
 *
 *		The new descriptor is inserted into the hash table if insertIt is true.
 *
 *		Returns NULL if no pg_class row could be found for the given relid
 *		(suggesting we are trying to access a just-deleted relation).
 *		Any other error is reported via elog.
 */
static Relation
RelationBuildDesc(Oid targetRelId, bool insertIt)
{
	Relation	relation;
	Oid			relid;
	HeapTuple	pg_class_tuple;
	Form_pg_class relp;

	/*
	 * This function and its subroutines can allocate a good deal of transient
	 * data in CurrentMemoryContext.  Traditionally we've just leaked that
	 * data, reasoning that the caller's context is at worst of transaction
	 * scope, and relcache loads shouldn't happen so often that it's essential
	 * to recover transient data before end of statement/transaction.  However
	 * that's definitely not true in clobber-cache test builds, and perhaps
	 * it's not true in other cases.  If RECOVER_RELATION_BUILD_MEMORY is not
	 * zero, arrange to allocate the junk in a temporary context that we'll
	 * free before returning.  Make it a child of caller's context so that it
	 * will get cleaned up appropriately if we error out partway through.
	 */
#if RECOVER_RELATION_BUILD_MEMORY
	MemoryContext tmpcxt;
	MemoryContext oldcxt;

	tmpcxt = AllocSetContextCreate(CurrentMemoryContext,
								   "RelationBuildDesc workspace",
								   ALLOCSET_DEFAULT_SIZES);
	oldcxt = MemoryContextSwitchTo(tmpcxt);
#endif

	/*
	 * find the tuple in pg_class corresponding to the given relation id
	 */
	pg_class_tuple = ScanPgRelation(targetRelId, true, false);

	/*
	 * if no such tuple exists, return NULL
	 */
	if (!HeapTupleIsValid(pg_class_tuple))
	{
#if RECOVER_RELATION_BUILD_MEMORY
		/* Return to caller's context, and blow away the temporary context */
		MemoryContextSwitchTo(oldcxt);
		MemoryContextDelete(tmpcxt);
#endif
		return NULL;
	}

	/*
	 * get information from the pg_class_tuple
	 */
	relp = (Form_pg_class) GETSTRUCT(pg_class_tuple);
	relid = relp->oid;
	Assert(relid == targetRelId);

	/*
	 * allocate storage for the relation descriptor, and copy pg_class_tuple
	 * to relation->rd_rel.
	 */
	relation = AllocateRelationDesc(relp);

	/*
	 * initialize the relation's relation id (relation->rd_id)
	 */
	RelationGetRelid(relation) = relid;

	/*
	 * normal relations are not nailed into the cache; nor can a pre-existing
	 * relation be new.  It could be temp though.  (Actually, it could be new
	 * too, but it's okay to forget that fact if forced to flush the entry.)
	 */
	relation->rd_refcnt = 0;
	relation->rd_isnailed = false;
	relation->rd_createSubid = InvalidSubTransactionId;
	relation->rd_newRelfilenodeSubid = InvalidSubTransactionId;
	switch (relation->rd_rel->relpersistence)
	{
		case RELPERSISTENCE_UNLOGGED:
		case RELPERSISTENCE_PERMANENT:
			relation->rd_backend = InvalidBackendId;
			relation->rd_islocaltemp = false;
			break;
		case RELPERSISTENCE_TEMP:
			if (isTempOrTempToastNamespace(relation->rd_rel->relnamespace))
			{
				relation->rd_backend = BackendIdForTempRelations();
				relation->rd_islocaltemp = true;
			}
			else
			{
				/*
				 * If it's a temp table, but not one of ours, we have to use
				 * the slow, grotty method to figure out the owning backend.
				 *
				 * Note: it's possible that rd_backend gets set to MyBackendId
				 * here, in case we are looking at a pg_class entry left over
				 * from a crashed backend that coincidentally had the same
				 * BackendId we're using.  We should *not* consider such a
				 * table to be "ours"; this is why we need the separate
				 * rd_islocaltemp flag.  The pg_class entry will get flushed
				 * if/when we clean out the corresponding temp table namespace
				 * in preparation for using it.
				 */
				relation->rd_backend =
					GetTempNamespaceBackendId(relation->rd_rel->relnamespace);
				Assert(relation->rd_backend != InvalidBackendId);
				relation->rd_islocaltemp = false;
			}
			break;
		default:
			elog(ERROR, "invalid relpersistence: %c",
				 relation->rd_rel->relpersistence);
			break;
	}

	/*
	 * initialize the tuple descriptor (relation->rd_att).
	 */
	RelationBuildTupleDesc(relation);

	/*
	 * Fetch rules and triggers that affect this relation
	 */
	if (relation->rd_rel->relhasrules)
		RelationBuildRuleLock(relation);
	else
	{
		relation->rd_rules = NULL;
		relation->rd_rulescxt = NULL;
	}

	if (relation->rd_rel->relhastriggers)
		RelationBuildTriggers(relation);
	else
		relation->trigdesc = NULL;

	if (relation->rd_rel->relrowsecurity)
		RelationBuildRowSecurity(relation);
	else
		relation->rd_rsdesc = NULL;

	/* foreign key data is not loaded till asked for */
	relation->rd_fkeylist = NIL;
	relation->rd_fkeyvalid = false;

	/* if a partitioned table, initialize key and partition descriptor info */
	if (relation->rd_rel->relkind == RELKIND_PARTITIONED_TABLE)
	{
		RelationBuildPartitionKey(relation);
		RelationBuildPartitionDesc(relation);
	}
	else
	{
		relation->rd_partkey = NULL;
		relation->rd_partkeycxt = NULL;
		relation->rd_partdesc = NULL;
		relation->rd_pdcxt = NULL;
	}
	/* ... but partcheck is not loaded till asked for */
	relation->rd_partcheck = NIL;
	relation->rd_partcheckvalid = false;
	relation->rd_partcheckcxt = NULL;

	/*
	 * initialize access method information
	 */
	switch (relation->rd_rel->relkind)
	{
		case RELKIND_INDEX:
		case RELKIND_PARTITIONED_INDEX:
			Assert(relation->rd_rel->relam != InvalidOid);
			RelationInitIndexAccessInfo(relation);
			break;
		case RELKIND_RELATION:
		case RELKIND_TOASTVALUE:
		case RELKIND_MATVIEW:
			Assert(relation->rd_rel->relam != InvalidOid);
			RelationInitTableAccessMethod(relation);
			break;
		case RELKIND_SEQUENCE:
			Assert(relation->rd_rel->relam == InvalidOid);
			RelationInitTableAccessMethod(relation);
			break;
		case RELKIND_VIEW:
		case RELKIND_COMPOSITE_TYPE:
		case RELKIND_FOREIGN_TABLE:
		case RELKIND_PARTITIONED_TABLE:
			Assert(relation->rd_rel->relam == InvalidOid);
			break;
	}

	/* extract reloptions if any */
	RelationParseRelOptions(relation, pg_class_tuple);

	/*
	 * initialize the relation lock manager information
	 */
	RelationInitLockInfo(relation); /* see lmgr.c */

	/*
	 * initialize physical addressing information for the relation
	 */
	RelationInitPhysicalAddr(relation);

	/* make sure relation is marked as having no open file yet */
	relation->rd_smgr = NULL;

	/*
	 * now we can free the memory allocated for pg_class_tuple
	 */
	heap_freetuple(pg_class_tuple);

	/*
	 * Insert newly created relation into relcache hash table, if requested.
	 *
	 * There is one scenario in which we might find a hashtable entry already
	 * present, even though our caller failed to find it: if the relation is a
	 * system catalog or index that's used during relcache load, we might have
	 * recursively created the same relcache entry during the preceding steps.
	 * So allow RelationCacheInsert to delete any already-present relcache
	 * entry for the same OID.  The already-present entry should have refcount
	 * zero (else somebody forgot to close it); in the event that it doesn't,
	 * we'll elog a WARNING and leak the already-present entry.
	 */
	if (insertIt)
		RelationCacheInsert(relation, true);

	/* It's fully valid */
	relation->rd_isvalid = true;

#if RECOVER_RELATION_BUILD_MEMORY
	/* Return to caller's context, and blow away the temporary context */
	MemoryContextSwitchTo(oldcxt);
	MemoryContextDelete(tmpcxt);
#endif

	return relation;
}
```
很关键的一个函数，用来构建关系的描述符。需要仔细了解下Relation, Oid, HeapTuple, Form_pg_class这几个结构体
##### Relation
```
typedef struct RelationData
{
	RelFileNode rd_node;		/* relation physical identifier */
	/* use "struct" here to avoid needing to include smgr.h: */
	struct SMgrRelationData *rd_smgr;	/* cached file handle, or NULL */
	int			rd_refcnt;		/* reference count */
	BackendId	rd_backend;		/* owning backend id, if temporary relation */
	bool		rd_islocaltemp; /* rel is a temp rel of this session */
	bool		rd_isnailed;	/* rel is nailed in cache */
	bool		rd_isvalid;		/* relcache entry is valid */
	bool		rd_indexvalid;	/* is rd_indexlist valid? (also rd_pkindex and
								 * rd_replidindex) */
	bool		rd_statvalid;	/* is rd_statlist valid? */

	/*
	 * rd_createSubid is the ID of the highest subtransaction the rel has
	 * survived into; or zero if the rel was not created in the current top
	 * transaction.  This can be now be relied on, whereas previously it could
	 * be "forgotten" in earlier releases. Likewise, rd_newRelfilenodeSubid is
	 * the ID of the highest subtransaction the relfilenode change has
	 * survived into, or zero if not changed in the current transaction (or we
	 * have forgotten changing it). rd_newRelfilenodeSubid can be forgotten
	 * when a relation has multiple new relfilenodes within a single
	 * transaction, with one of them occurring in a subsequently aborted
	 * subtransaction, e.g. BEGIN; TRUNCATE t; SAVEPOINT save; TRUNCATE t;
	 * ROLLBACK TO save; -- rd_newRelfilenode is now forgotten
	 */
	SubTransactionId rd_createSubid;	/* rel was created in current xact */
	SubTransactionId rd_newRelfilenodeSubid;	/* new relfilenode assigned in
												 * current xact */

	Form_pg_class rd_rel;		/* RELATION tuple */
	TupleDesc	rd_att;			/* tuple descriptor */
	Oid			rd_id;			/* relation's object id */
	LockInfoData rd_lockInfo;	/* lock mgr's info for locking relation */
	RuleLock   *rd_rules;		/* rewrite rules */
	MemoryContext rd_rulescxt;	/* private memory cxt for rd_rules, if any */
	TriggerDesc *trigdesc;		/* Trigger info, or NULL if rel has none */
	/* use "struct" here to avoid needing to include rowsecurity.h: */
	struct RowSecurityDesc *rd_rsdesc;	/* row security policies, or NULL */

	/* data managed by RelationGetFKeyList: */
	List	   *rd_fkeylist;	/* list of ForeignKeyCacheInfo (see below) */
	bool		rd_fkeyvalid;	/* true if list has been computed */

	struct PartitionKeyData *rd_partkey;	/* partition key, or NULL */
	MemoryContext rd_partkeycxt;	/* private context for rd_partkey, if any */
	struct PartitionDescData *rd_partdesc;	/* partitions, or NULL */
	MemoryContext rd_pdcxt;		/* private context for rd_partdesc, if any */
	List	   *rd_partcheck;	/* partition CHECK quals */
	bool		rd_partcheckvalid;	/* true if list has been computed */
	MemoryContext rd_partcheckcxt;	/* private cxt for rd_partcheck, if any */

	/* data managed by RelationGetIndexList: */
	List	   *rd_indexlist;	/* list of OIDs of indexes on relation */
	Oid			rd_pkindex;		/* OID of primary key, if any */
	Oid			rd_replidindex; /* OID of replica identity index, if any */

	/* data managed by RelationGetStatExtList: */
	List	   *rd_statlist;	/* list of OIDs of extended stats */

	/* data managed by RelationGetIndexAttrBitmap: */
	Bitmapset  *rd_indexattr;	/* identifies columns used in indexes */
	Bitmapset  *rd_keyattr;		/* cols that can be ref'd by foreign keys */
	Bitmapset  *rd_pkattr;		/* cols included in primary key */
	Bitmapset  *rd_idattr;		/* included in replica identity index */

	PublicationActions *rd_pubactions;	/* publication actions */

	/*
	 * rd_options is set whenever rd_rel is loaded into the relcache entry.
	 * Note that you can NOT look into rd_rel for this data.  NULL means "use
	 * defaults".
	 */
	bytea	   *rd_options;		/* parsed pg_class.reloptions */

	/*
	 * Oid of the handler for this relation. For an index this is a function
	 * returning IndexAmRoutine, for table like relations a function returning
	 * TableAmRoutine.  This is stored separately from rd_indam, rd_tableam as
	 * its lookup requires syscache access, but during relcache bootstrap we
	 * need to be able to initialize rd_tableam without syscache lookups.
	 */
	Oid			rd_amhandler;	/* OID of index AM's handler function */

	/*
	 * Table access method.
	 */
	const struct TableAmRoutine *rd_tableam;

	/* These are non-NULL only for an index relation: */
	Form_pg_index rd_index;		/* pg_index tuple describing this index */
	/* use "struct" here to avoid needing to include htup.h: */
	struct HeapTupleData *rd_indextuple;	/* all of pg_index tuple */

	/*
	 * index access support info (used only for an index relation)
	 *
	 * Note: only default support procs for each opclass are cached, namely
	 * those with lefttype and righttype equal to the opclass's opcintype. The
	 * arrays are indexed by support function number, which is a sufficient
	 * identifier given that restriction.
	 *
	 * Note: rd_amcache is available for index AMs to cache private data about
	 * an index.  This must be just a cache since it may get reset at any time
	 * (in particular, it will get reset by a relcache inval message for the
	 * index).  If used, it must point to a single memory chunk palloc'd in
	 * rd_indexcxt.  A relcache reset will include freeing that chunk and
	 * setting rd_amcache = NULL.
	 */
	MemoryContext rd_indexcxt;	/* private memory cxt for this stuff */
	/* use "struct" here to avoid needing to include amapi.h: */
	struct IndexAmRoutine *rd_indam;	/* index AM's API struct */
	Oid		   *rd_opfamily;	/* OIDs of op families for each index col */
	Oid		   *rd_opcintype;	/* OIDs of opclass declared input data types */
	RegProcedure *rd_support;	/* OIDs of support procedures */
	FmgrInfo   *rd_supportinfo; /* lookup info for support procedures */
	int16	   *rd_indoption;	/* per-column AM-specific flags */
	List	   *rd_indexprs;	/* index expression trees, if any */
	List	   *rd_indpred;		/* index predicate tree, if any */
	Oid		   *rd_exclops;		/* OIDs of exclusion operators, if any */
	Oid		   *rd_exclprocs;	/* OIDs of exclusion ops' procs, if any */
	uint16	   *rd_exclstrats;	/* exclusion ops' strategy numbers, if any */
	void	   *rd_amcache;		/* available for use by index AM */
	Oid		   *rd_indcollation;	/* OIDs of index collations */

	/*
	 * foreign-table support
	 *
	 * rd_fdwroutine must point to a single memory chunk palloc'd in
	 * CacheMemoryContext.  It will be freed and reset to NULL on a relcache
	 * reset.
	 */

	/* use "struct" here to avoid needing to include fdwapi.h: */
	struct FdwRoutine *rd_fdwroutine;	/* cached function pointers, or NULL */

	/*
	 * Hack for CLUSTER, rewriting ALTER TABLE, etc: when writing a new
	 * version of a table, we need to make any toast pointers inserted into it
	 * have the existing toast table's OID, not the OID of the transient toast
	 * table.  If rd_toastoid isn't InvalidOid, it is the OID to place in
	 * toast pointers inserted into this rel.  (Note it's set on the new
	 * version of the main heap, not the toast table itself.)  This also
	 * causes toast_save_datum() to try to preserve toast value OIDs.
	 */
	Oid			rd_toastoid;	/* Real TOAST table's OID, or InvalidOid */

	/* use "struct" here to avoid needing to include pgstat.h: */
	struct PgStat_TableStatus *pgstat_info; /* statistics collection area */
} RelationData;
```
Relation为RelationData结构的指针
#### postgres_ext.h Oid
```
/*
 * Object ID is a fundamental type in Postgres.
 */
typedef unsigned int Oid;
```
#### htup.h HeapTuple
```
/*
 * HeapTupleData is an in-memory data structure that points to a tuple.
 *
 * There are several ways in which this data structure is used:
 *
 * * Pointer to a tuple in a disk buffer: t_data points directly into the
 *	 buffer (which the code had better be holding a pin on, but this is not
 *	 reflected in HeapTupleData itself).
 *
 * * Pointer to nothing: t_data is NULL.  This is used as a failure indication
 *	 in some functions.
 *
 * * Part of a palloc'd tuple: the HeapTupleData itself and the tuple
 *	 form a single palloc'd chunk.  t_data points to the memory location
 *	 immediately following the HeapTupleData struct (at offset HEAPTUPLESIZE).
 *	 This is the output format of heap_form_tuple and related routines.
 *
 * * Separately allocated tuple: t_data points to a palloc'd chunk that
 *	 is not adjacent to the HeapTupleData.  (This case is deprecated since
 *	 it's difficult to tell apart from case #1.  It should be used only in
 *	 limited contexts where the code knows that case #1 will never apply.)
 *
 * * Separately allocated minimal tuple: t_data points MINIMAL_TUPLE_OFFSET
 *	 bytes before the start of a MinimalTuple.  As with the previous case,
 *	 this can't be told apart from case #1 by inspection; code setting up
 *	 or destroying this representation has to know what it's doing.
 *
 * t_len should always be valid, except in the pointer-to-nothing case.
 * t_self and t_tableOid should be valid if the HeapTupleData points to
 * a disk buffer, or if it represents a copy of a tuple on disk.  They
 * should be explicitly set invalid in manufactured tuples.
 */
typedef struct HeapTupleData
{
	uint32		t_len;			/* length of *t_data */
	ItemPointerData t_self;		/* SelfItemPointer */
	Oid			t_tableOid;		/* table the tuple came from */
#define FIELDNO_HEAPTUPLEDATA_DATA 3
	HeapTupleHeader t_data;		/* -> tuple header and data */
} HeapTupleData;
```
可以看出该结构主要用来表示一个内存中的元组，ItemPointerData表示该元祖在硬盘上的位置，由blockID和在block上的偏移量确定.
t_data一般情况下存储着元组头，紧跟着元祖头的内存空间存着元祖的数据。
##### itemptr.h
```
/*
 * ItemPointer:
 *
 * This is a pointer to an item within a disk page of a known file
 * (for example, a cross-link from an index to its parent table).
 * blkid tells us which block, posid tells us which entry in the linp
 * (ItemIdData) array we want.
 *
 * Note: because there is an item pointer in each tuple header and index
 * tuple header on disk, it's very important not to waste space with
 * structure padding bytes.  The struct is designed to be six bytes long
 * (it contains three int16 fields) but a few compilers will pad it to
 * eight bytes unless coerced.  We apply appropriate persuasion where
 * possible.  If your compiler can't be made to play along, you'll waste
 * lots of space.
 */
typedef struct ItemPointerData
{
	BlockIdData ip_blkid;
	OffsetNumber ip_posid;
}
```

#### Form_pg_class
```

/* ----------------
 *		pg_class definition.  cpp turns this into
 *		typedef struct FormData_pg_class
 * ----------------
 */
CATALOG(pg_class,1259,RelationRelationId) BKI_BOOTSTRAP BKI_ROWTYPE_OID(83,RelationRelation_Rowtype_Id) BKI_SCHEMA_MACRO
{
	/* oid */
	Oid			oid;

	/* class name */
	NameData	relname;

	/* OID of namespace containing this class */
	Oid			relnamespace BKI_DEFAULT(PGNSP);

	/* OID of entry in pg_type for table's implicit row type */
	Oid			reltype BKI_LOOKUP(pg_type);

	/* OID of entry in pg_type for underlying composite type */
	Oid			reloftype BKI_DEFAULT(0) BKI_LOOKUP(pg_type);

	/* class owner */
	Oid			relowner BKI_DEFAULT(PGUID);

	/* access method; 0 if not a table / index */
	Oid			relam BKI_LOOKUP(pg_am);

	/* identifier of physical storage file */
	/* relfilenode == 0 means it is a "mapped" relation, see relmapper.c */
	Oid			relfilenode;

	/* identifier of table space for relation (0 means default for database) */
	Oid			reltablespace BKI_DEFAULT(0) BKI_LOOKUP(pg_tablespace);

	/* # of blocks (not always up-to-date) */
	int32		relpages;

	/* # of tuples (not always up-to-date) */
	float4		reltuples;

	/* # of all-visible blocks (not always up-to-date) */
	int32		relallvisible;

	/* OID of toast table; 0 if none */
	Oid			reltoastrelid;

	/* T if has (or has had) any indexes */
	bool		relhasindex;

	/* T if shared across databases */
	bool		relisshared;

	/* see RELPERSISTENCE_xxx constants below */
	char		relpersistence;

	/* see RELKIND_xxx constants below */
	char		relkind;

	/* number of user attributes */
	int16		relnatts;

	/*
	 * Class pg_attribute must contain exactly "relnatts" user attributes
	 * (with attnums ranging from 1 to relnatts) for this class.  It may also
	 * contain entries with negative attnums for system attributes.
	 */

	/* # of CHECK constraints for class */
	int16		relchecks;

	/* has (or has had) any rules */
	bool		relhasrules;

	/* has (or has had) any TRIGGERs */
	bool		relhastriggers;

	/* has (or has had) child tables or indexes */
	bool		relhassubclass;

	/* row security is enabled or not */
	bool		relrowsecurity;

	/* row security forced for owners or not */
	bool		relforcerowsecurity;

	/* matview currently holds query results */
	bool		relispopulated;

	/* see REPLICA_IDENTITY_xxx constants */
	char		relreplident;

	/* is relation a partition? */
	bool		relispartition;

	/* heap for rewrite during DDL, link to original rel */
	Oid			relrewrite BKI_DEFAULT(0);

	/* all Xids < this are frozen in this rel */
	TransactionId relfrozenxid;

	/* all multixacts in this rel are >= this; it is really a MultiXactId */
	TransactionId relminmxid;

#ifdef CATALOG_VARLEN			/* variable-length fields start here */
	/* NOTE: These fields are not present in a relcache entry's rd_rel field. */
	/* access permissions */
	aclitem		relacl[1];

	/* access-method-specific options */
	text		reloptions[1];

	/* partition bound node tree */
	pg_node_tree relpartbound;
#endif
} FormData_pg_class;
```
可以把FormData_pg_class理解为一个表的概括。
#### RelationBuildDesc
有了上面的基础认知，继续阅读RelationBuildDesc的逻辑。
```
	/*
	 * find the tuple in pg_class corresponding to the given relation id
	 */
	pg_class_tuple = ScanPgRelation(targetRelId, true, false);
```
可以看出pg_class也是存储在一个关系中，根据oid可以找到对应关系的pg_class元组
```
	/*
	 * get information from the pg_class_tuple
	 */
	relp = (Form_pg_class) GETSTRUCT(pg_class_tuple);
```
`GETSTRUCT`的作用主要是获取元组的data部分（不含头～），然后将这部分数据标示为Form_pg_class.
忽略cache相关代码，观察如何获取关系的属性字典
```
	/*
	 * initialize the tuple descriptor (relation->rd_att).
	 */
	RelationBuildTupleDesc(relation);
```
##### relcache.c RelationBuildTupleDesc()
```
/*
 *		RelationBuildTupleDesc
 *
 *		Form the relation's tuple descriptor from information in
 *		the pg_attribute, pg_attrdef & pg_constraint system catalogs.
 */
static void
RelationBuildTupleDesc(Relation relation)
{
	HeapTuple	pg_attribute_tuple;
	Relation	pg_attribute_desc;
	SysScanDesc pg_attribute_scan;
	ScanKeyData skey[2];
	int			need;
	TupleConstr *constr;
	AttrDefault *attrdef = NULL;
	AttrMissing *attrmiss = NULL;
	int			ndef = 0;

	/* copy some fields from pg_class row to rd_att */
	relation->rd_att->tdtypeid = relation->rd_rel->reltype;
	relation->rd_att->tdtypmod = -1;	/* unnecessary, but... */

	constr = (TupleConstr *) MemoryContextAlloc(CacheMemoryContext,
												sizeof(TupleConstr));
	constr->has_not_null = false;
	constr->has_generated_stored = false;

	/*
	 * Form a scan key that selects only user attributes (attnum > 0).
	 * (Eliminating system attribute rows at the index level is lots faster
	 * than fetching them.)
	 */
	ScanKeyInit(&skey[0],
				Anum_pg_attribute_attrelid,
				BTEqualStrategyNumber, F_OIDEQ,
				ObjectIdGetDatum(RelationGetRelid(relation)));
	ScanKeyInit(&skey[1],
				Anum_pg_attribute_attnum,
				BTGreaterStrategyNumber, F_INT2GT,
				Int16GetDatum(0));

	/*
	 * Open pg_attribute and begin a scan.  Force heap scan if we haven't yet
	 * built the critical relcache entries (this includes initdb and startup
	 * without a pg_internal.init file).
	 */
	pg_attribute_desc = table_open(AttributeRelationId, AccessShareLock);
	pg_attribute_scan = systable_beginscan(pg_attribute_desc,
										   AttributeRelidNumIndexId,
										   criticalRelcachesBuilt,
										   NULL,
										   2, skey);

	/*
	 * add attribute data to relation->rd_att
	 */
	need = RelationGetNumberOfAttributes(relation);

	while (HeapTupleIsValid(pg_attribute_tuple = systable_getnext(pg_attribute_scan)))
	{
		Form_pg_attribute attp;
		int			attnum;

		attp = (Form_pg_attribute) GETSTRUCT(pg_attribute_tuple);

		attnum = attp->attnum;
		if (attnum <= 0 || attnum > RelationGetNumberOfAttributes(relation))
			elog(ERROR, "invalid attribute number %d for %s",
				 attp->attnum, RelationGetRelationName(relation));


		memcpy(TupleDescAttr(relation->rd_att, attnum - 1),
			   attp,
			   ATTRIBUTE_FIXED_PART_SIZE);

		/* Update constraint/default info */
		if (attp->attnotnull)
			constr->has_not_null = true;
		if (attp->attgenerated == ATTRIBUTE_GENERATED_STORED)
			constr->has_generated_stored = true;

		/* If the column has a default, fill it into the attrdef array */
		if (attp->atthasdef)
		{
			if (attrdef == NULL)
				attrdef = (AttrDefault *)
					MemoryContextAllocZero(CacheMemoryContext,
										   RelationGetNumberOfAttributes(relation) *
										   sizeof(AttrDefault));
			attrdef[ndef].adnum = attnum;
			attrdef[ndef].adbin = NULL;

			ndef++;
		}

		/* Likewise for a missing value */
		if (attp->atthasmissing)
		{
			Datum		missingval;
			bool		missingNull;

			/* Do we have a missing value? */
			missingval = heap_getattr(pg_attribute_tuple,
									  Anum_pg_attribute_attmissingval,
									  pg_attribute_desc->rd_att,
									  &missingNull);
			if (!missingNull)
			{
				/* Yes, fetch from the array */
				MemoryContext oldcxt;
				bool		is_null;
				int			one = 1;
				Datum		missval;

				if (attrmiss == NULL)
					attrmiss = (AttrMissing *)
						MemoryContextAllocZero(CacheMemoryContext,
											   relation->rd_rel->relnatts *
											   sizeof(AttrMissing));

				missval = array_get_element(missingval,
											1,
											&one,
											-1,
											attp->attlen,
											attp->attbyval,
											attp->attalign,
											&is_null);
				Assert(!is_null);
				if (attp->attbyval)
				{
					/* for copy by val just copy the datum direct */
					attrmiss[attnum - 1].am_value = missval;
				}
				else
				{
					/* otherwise copy in the correct context */
					oldcxt = MemoryContextSwitchTo(CacheMemoryContext);
					attrmiss[attnum - 1].am_value = datumCopy(missval,
															  attp->attbyval,
															  attp->attlen);
					MemoryContextSwitchTo(oldcxt);
				}
				attrmiss[attnum - 1].am_present = true;
			}
		}
		need--;
		if (need == 0)
			break;
	}

	/*
	 * end the scan and close the attribute relation
	 */
	systable_endscan(pg_attribute_scan);
	table_close(pg_attribute_desc, AccessShareLock);

	if (need != 0)
		elog(ERROR, "catalog is missing %d attribute(s) for relid %u",
			 need, RelationGetRelid(relation));

	/*
	 * The attcacheoff values we read from pg_attribute should all be -1
	 * ("unknown").  Verify this if assert checking is on.  They will be
	 * computed when and if needed during tuple access.
	 */
#ifdef USE_ASSERT_CHECKING
	{
		int			i;

		for (i = 0; i < RelationGetNumberOfAttributes(relation); i++)
			Assert(TupleDescAttr(relation->rd_att, i)->attcacheoff == -1);
	}
#endif

	/*
	 * However, we can easily set the attcacheoff value for the first
	 * attribute: it must be zero.  This eliminates the need for special cases
	 * for attnum=1 that used to exist in fastgetattr() and index_getattr().
	 */
	if (RelationGetNumberOfAttributes(relation) > 0)
		TupleDescAttr(relation->rd_att, 0)->attcacheoff = 0;

	/*
	 * Set up constraint/default info
	 */
	if (constr->has_not_null || ndef > 0 ||
		attrmiss || relation->rd_rel->relchecks)
	{
		relation->rd_att->constr = constr;

		if (ndef > 0)			/* DEFAULTs */
		{
			if (ndef < RelationGetNumberOfAttributes(relation))
				constr->defval = (AttrDefault *)
					repalloc(attrdef, ndef * sizeof(AttrDefault));
			else
				constr->defval = attrdef;
			constr->num_defval = ndef;
			AttrDefaultFetch(relation);
		}
		else
			constr->num_defval = 0;

		constr->missing = attrmiss;

		if (relation->rd_rel->relchecks > 0)	/* CHECKs */
		{
			constr->num_check = relation->rd_rel->relchecks;
			constr->check = (ConstrCheck *)
				MemoryContextAllocZero(CacheMemoryContext,
									   constr->num_check * sizeof(ConstrCheck));
			CheckConstraintFetch(relation);
		}
		else
			constr->num_check = 0;
	}
	else
	{
		pfree(constr);
		relation->rd_att->constr = NULL;
	}
}
```
该函数主要用来从pg_attribute表中读出与查询的relation相关的属性结构描述元组,即：
```
/*
 * This struct is passed around within the backend to describe the structure
 * of tuples.  For tuples coming from on-disk relations, the information is
 * collected from the pg_attribute, pg_attrdef, and pg_constraint catalogs.
 * Transient row types (such as the result of a join query) have anonymous
 * TupleDesc structs that generally omit any constraint info; therefore the
 * structure is designed to let the constraints be omitted efficiently.
 *
 * Note that only user attributes, not system attributes, are mentioned in
 * TupleDesc.
 *
 * If the tupdesc is known to correspond to a named rowtype (such as a table's
 * rowtype) then tdtypeid identifies that type and tdtypmod is -1.  Otherwise
 * tdtypeid is RECORDOID, and tdtypmod can be either -1 for a fully anonymous
 * row type, or a value >= 0 to allow the rowtype to be looked up in the
 * typcache.c type cache.
 *
 * Note that tdtypeid is never the OID of a domain over composite, even if
 * we are dealing with values that are known (at some higher level) to be of
 * a domain-over-composite type.  This is because tdtypeid/tdtypmod need to
 * match up with the type labeling of composite Datums, and those are never
 * explicitly marked as being of a domain type, either.
 *
 * Tuple descriptors that live in caches (relcache or typcache, at present)
 * are reference-counted: they can be deleted when their reference count goes
 * to zero.  Tuple descriptors created by the executor need no reference
 * counting, however: they are simply created in the appropriate memory
 * context and go away when the context is freed.  We set the tdrefcount
 * field of such a descriptor to -1, while reference-counted descriptors
 * always have tdrefcount >= 0.
 */
typedef struct TupleDescData
{
	int			natts;			/* number of attributes in the tuple */
	Oid			tdtypeid;		/* composite type ID for tuple type */
	int32		tdtypmod;		/* typmod for tuple type */
	int			tdrefcount;		/* reference count, or -1 if not counting */
	TupleConstr *constr;		/* constraints, or NULL if none */
	/* attrs[N] is the description of Attribute Number N+1 */
	FormData_pg_attribute attrs[FLEXIBLE_ARRAY_MEMBER];
} TupleDescData;
typedef struct TupleDescData *TupleDesc;
```
至此，关于relation对象的主要构造功能已完成（忽略缺失值处理，默认值等等细节）
### 总结
postgresql的许多结构描述性信息均以元组的形式存储在表格中，关系字典存储在pg_class表格中，属性字典存储在pg_attribute表中。
构造Relation对象的过程为，根据oid从pg_class表中获取与该关系相关的关系数据字典，而后根据relation id获取属性字典。
后续获取Relation的各个元组后，需要根据属性字典对它们进行解析。

