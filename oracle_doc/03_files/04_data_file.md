## Data File
Data File & Redo Log are most important file in database.

### File System
Store your data in:
- Cooked OS file system: you can check via 'ls' or windows resouce management system
- Raw partition: a raw disc without OS, no buffer
- Automatic Storage Management(ASM): a database file system, only used to store table, index, backup file etc
- Cluster file system: provide a sharing file system cross nodes

### Oracle Storage System
Tablespace is a logical storage container, includes multiple data files.

#### Segment
Tablespace main structure. When you create a table, it will create a table segment. When you create a partition, it will create parition segment, create index will create index segment, rollback segment, temporary segment, cluster segment, etc.

#### extent
A segment contains one or more extent. Extent is one logical consistent space of a file.
- Extent will create when INSERT data
- it may store in different files
- it may have different size (block <= x <= 2GB)

#### block

