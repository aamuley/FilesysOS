# Filesys

Added functionality for a Unix FFS based file system in the OS with the following capabilities
- Created a buffer cache to reduce the writes to disk and optimize our actions. This was 64 blocks in size and used static blocks, a true LRU eviction policy, and synchronized operations to read/write from separate blocks to maximize througput
- Created support for Extensible files with dynamic allocation and growth up to 8 MiB per file. We used implemented the FFS style filesystem with inodes, direct, indirect, and doubly indirect pointers to maximize the size of the files.
- Allowed for subdirectories to be created and syscalls to support user initialted changes to these with functionality for a process to have a current working directory, change/delete/create subdirectories, and efficiently lookup paths to certain files by storing file contents in the first inode allocated for the directory.

  Please see the detailed design and reflection for more details! 
