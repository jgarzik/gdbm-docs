# The GDBM File Format

## Overview

The GDBM file format is an on-disk format inspired by in-memory hash
table data structures.  The *header* describes the entire database.  The
*directory* is a vector of storage offsets for buckets, which is the
core, 1st level hash table.  A *bucket* is 2nd, smaller hash table which
contains *bucket elements*, the actual key/value data items themselves.

## Database Header

The **GDBM Header** data structures summarizes the entire database file,
similar to a filesystem superblock.  The Header provides the root of all
data lookups and updates.

```
typedef struct
{
  int32_t   header_magic;  /* Version of file. */
  int32_t   block_size;    /* The optimal i/o blocksize from stat. */
  off_t     dir;           /* File address of hash directory table. */
  int32_t   dir_size;      /* Size in bytes of the table.  */
  int32_t   dir_bits;      /* The number of address bits used in the table.*/
  int32_t   bucket_size;   /* Size in bytes of a hash bucket struct. */
  int32_t   bucket_elems;  /* Number of elements in a hash bucket. */
  off_t     next_block;    /* The next unallocated block address. */
} gdbm_file_header;
```

## Bucket Directory

The 1st hash table accessed during lookups and storage is the *bucket
directory*.  This is a vector of file offsets.  Each vector element is a file offset, pointing to a single bucket.

```
typedef struct
{
  off_t bucket_offset_vector[1]; /* The table. Make it look like an array. */
} gdbm_bucket_directory;
```

## Bucket

A bucket is a small, 2nd hash table.  A GDBM database consists of a large number of buckets, indexed by hash via the bucket directory.

A bucket consists of a number of bucket elements plus some bookkeeping fields.  The number of elements depends on the optimum blocksize for the storage device and on a parameter given at file creation time.  This bucket takes one block.

When one of these tables gets full, it is split into two hash buckets.
The contents are split between them by the use of the first few bits
of the 31 bit hash function.  The location in a bucket is the hash
value modulo the size of the bucket.  The in-memory images of the
buckets are allocated by malloc using a calculated size depending of
the file system buffer size.  To speed up write, each bucket will have
BUCKET_AVAIL avail elements with the bucket.

```
typedef struct
{
  int32_t   av_count;            /* The number of bucket_avail entries. */
  avail_elem bucket_avail[BUCKET_AVAIL];  /* Distributed avail. */
  int32_t   bucket_bits;         /* The number of bits used to get here. */
  int32_t   count;               /* The number of element buckets full. */
  bucket_element h_table[1];     /* The table.  Make it look like an array.*/
} hash_bucket;
```

## Bucket Element

A hash bucket element is the final, atomic unit resultng from a lookup or update: a single key/value data record.

It contains the full 31 bit hash value, the "pointer" to the key and data (stored together) with their sizes.  It also
has a small part of the actual key value.  It is used to verify the first
part of the key has the correct value without having to read the actual
key.

```
typedef struct
{
  int32_t   hash_value;   /* The complete 31 bit value. */
  char  key_start[SMALL]; /* Up to the first SMALL bytes of the key.  */
  off_t data_pointer;     /* The file address of the key record. The
                             data record directly follows the key.  */
  int32_t   key_size;     /* Size of key data in the file. */
  int32_t   data_size;    /* Size of associated data in the file. */
} bucket_element;
```

## Avail Table

The available file space is stored in an "avail" (available free space) table.  The one with most activity is contained in the file header. When that one fills up, it is split in half and half is pushed on an "avail stack."  When the active avail table is empty and the "avail stack" is not empty, the top of the stack is popped into the active avail table.
   
In other words, this is a database-wide free list; a list of allocated, on-disk storage space (free space) available for re-use.  It is stored as a linked list of *avail blocks*, with the initial avail block referenced in the database header.

## Avail Block

A data structure containing a portion of the free list (*avail
elements*), and a pointer to the next block in the linked list.

```
typedef struct
{
  int32_t   size;         /* The number of avail elements in the table.*/
  int32_t   count;        /* The number of entries in the table. */
  off_t     next_block;   /* The file address of the next avail block. */
  avail_elem av_table[1]; /* The table.  Make it look like an array.  */
} avail_block;
```

## Avail Element

A single (offset, length) pair indicating a byte range within the
on-disk file that may be re-used for the next database storage
operation.

```
typedef struct
{
  int32_t  av_size;            /* The size of the available block. */
  off_t    av_adr;             /* The file address of the available block. */
} avail_elem;
```
