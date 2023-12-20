# [FA23] Project File System Design
Group 24

| Name         | Autograder Login | Email                     |
| ------------ | ---------------- | ------------------------- |
| Sydney Tsai  | student134       | sydneytsai@gmail.com      |
| Anusha Muley | student143       | anusha.muley@berkeley.edu |
| Cindy Yang   | student258       | cindycy@berkeley.edu      |
| Qi Li Yang   | student85        | qiyang2@berkeley.edu      |



----------
# Buffer Cache
## Data Structures and Functions

Data structures/variables added:

    /* (not sure if theres an existing file instead) In new filesys/buffer.h */
    struct buffer_cache {
      struct lock* buf_lock; // to ensure synch when modding the buffer
      int curr_clock; // where the current clock hand is pointing [0, 64)
      struct list waiting_queue; // list of the next vals to add to cache if full and all elems are used
      struct hash* hash_table; // use to lookup the sector index and block_sector pointer
        // struct hash {
        // size_t elem_cnt;      = 64 elems 
        // size_t bucket_cnt;    = 2^6 buckets (one per elem)
        // struct list* buckets; = pintos list of the buckets
        // hash_hash_func* hash; = directly map the elem to a bucket equal to their index
     } 
    struct cache_elem{ // similar to an fdt elem, index and block
      struct lock hash_lock; 
      int index; // index [0-64) for the block of the buffer
      struct block_sector_t block; // block where the buffer has been allocated
      uint_8 data[BLOCK_SECTOR_SIZE]; // actual info stored
      struct hash_elem* elem; // basically a list_elem, used to add in hash_table
      bool is_dirty;
      bool ref_bit; // clock policy
      bool is_unused;
      bool is_loaded;
    }

Data structures/variables modified:

    /* In userprog/syscall.c */
      struct lock global_file_lock; /* REMOVED */
      lock_acq/rel of global_file_lock in every file syscall /* REMOVED */
    

Functions added:

    /* In new filesys/buffer.c  */
    void cache_init(); /* (1) init buffer cache here  */
    void cache_read(struct block*, block_sector_t, void* buffer); // (2) read an entry stored in the cache and loads it to a buffer? 
    void cache_write(struct block*, block_sector_t, const void* buffer); // (3) writes to the cache element sector from buffer
    void flush_cache(); // (4) flush every cache entry with a dirty bit and write back to memory. Do not evict a page when flushed but clear dirty bit. Used in filesys_done, scheduled flush kernel thread (concept check) */
    void get_entry (block_sector_t sector, struct cache_elem* hashed_e); // (5) get the entry number from the cache if it exists, assign a new one if it doesnt, add to waiting list if all entries are in use. 
    void cache_read_disk(struct block*, block_sector_t, struct cache_elem e); // (6) load an entry from disk into the cache_elem given 
    void cache_write_disk(struct block*, block_sector_t, struct cache_elem e); // (7) flush the individual cache_elem given to disk and set unused
    uint8_t hash_func(block_sector_t block); // return the hashed cache_elem block for the sector number 
    int evict_next(); // evict an entry in the cache by clock replacement policy 

Functions modified:

    /* filesys.c */
    void filesys_init(bool format); /* call cache_init here */
    void filesys_done(void); /* before closing the filesys, parse through the hash table of all the currently mapped sectors and if the dirty bit is set, make sure to write to mem before closing the filesys, deallocate buffer cache entries in the hashmap and the struct using free() */
    
    /* In userprog/syscall.c (from removing fs_lock) */ 
    void syscall_init(void);  /* remove lock_init() */
    /* rem lock acq/rel fs_lock, acquire block specific lock at lower inode level for currently executing file/passed in file and release once syscall completes */
    void safe_file_close(struct file* file); 
    int sys_exec(const char* ufile); 
    int sys_create(const char* ufile, unsigned initial_size); 
    int sys_remove(const char* ufile); 
    int sys_open(const char* ufile); 
    int sys_filesize(int handle);
    int sys_read(int handle, void* udst_, unsigned size);
    int sys_write(int handle, void* usrc_, unsigned size);
    int sys_seek(int handle, unsigned position);
    int sys_tell(int handle);
    int sys_close(int handle);
    
    /* in inode.c */
    // Change EVERY instance using block_read and block_write on inode->sector in this file to instead to go through the cache read/write functions
    // remove bounce buffer here
    struct inode* inode_open(block_sector_t sector);
    void inode_close(struct inode* inode);
    off_t inode_read_at(struct inode* inode, void* buffer_, off_t size, off_t offset);
    off_t inode_write_at(struct inode* inode, void* buffer_, off_t size, off_t offset);
    
    /* USE: devices/block.h */
    void block_read(struct block*, block_sector_t, void*);
    void block_write(struct block*, block_sector_t, const void*);
## Algorithms
    void cache_init(); // (1)
     malloc buffer_cache struct size
    // (do not malloc the cache_elems that will be done dynamically)
     lock_init(buf_lock)
     lock_init(hash_lock)
     is_unused = true for every cache_elem


    void cache_read(struct block*, block_sector_t, void* buffer); // (2)
      index = get_entry(block_sector)
     lock_acq(hashmap.get(index)->hash_lock)
     if hashed_e-> is_loaded = false: lock_rel(hashmap.get(index)->hash_lock)
     else: memcpy cache_elem->data to buffer
     lock_rel(hashmap.get(index)->hash_lock)
    
    void cache_write(struct block*, block_sector_t, const void* buffer); // (3)
     index = get_entry(block_sector)
     lock_acq(hashmap.get(index)->hash_lock)
     if hashed_e-> is_loaded = false: lock_rel(hashmap.get(index)->hash_lock)
     else: memcpy buffer to cache_elem->data
     hashmap.get(index)->is_dirty = 1
     lock_rel(hashmap.get(index)->hash_lock)
    


    void flush_cache(); // (4)
     lock acq buf_lock to synch the hashmap
     for(int i = 0; i < 64; i ++){
       lock_acq(hash_table.get(i)->hash_lock)
       if (hashmap.get(i).is_dirty == true):
         writeback to disk: block_write (hash_table.get(i).block)
       lock_rel (hash_table.get(i).hash_lock);
     lock_rel buf_lock 


    void cache_read_disk(struct block*, block_sector_t); // (6) 
    // e = malloc new cache_elem
    // e->is_loaded = false
    // lock_acq(buf_lock)
    // hash.insert(hashfn(block_sector_t), e)
    // lock_release(buf_lock)
    // lock_acq(e->hash_lock)
    // block_read(block, block_sector_t, e->data)
    // e->is_dirty = false
    // e->ref_bit = true
    // e->is_loaded = true
    // lock_rel(e->hash_lock)
    void cache_write_disk(struct block*, block_sector_t, struct cache_elem e); // (7) 
    // lock_acq(e->hash_lock)
    // block_write(block, block_sector_t, e->data)
    // e->is_dirty = false
    // lock_rel(e->hash_lock)


    int get_entry (block_sector_t sector); //  (5)
    // lock_acq(buf_lock)
    // if the sector is already assigned a cache block, retrieve the entry index (the hash of the block sector should be in the hashmap?)
    // else if the hashed_e is not in the cache:
      // new_index = -1
      // for every element, if lock_try_acq(elem->hash_lock) and elem->is_unused = true:
        // new_index = elem->index
        // break;
      // else if new_index = -1 // all in use need to evict
          // if evict_next()== -1, failed and all the sectors are in use
            // add to waiting queue
          // else: return evict_next()
    // lock_rel(buf_lock)
    
    int evict_next(); // assume buf_lock acquired
    // int starting = curr_clock; 
    // while (curr_clock != starting &&  == 1){
      // if hashmap.get(curr_clock)->ref_bit = 1 {
          // s = lock_try_acq (hash_table.get(curr_clock).hash_lock);
          // if s: 
              if is_dirty == 1
              // cache_write_disk (get_block_ptr(i)) , 
                 hash.remove(cache_elem)
              // lock_rel(hash_table.get(curr_clock).hash_lock);
              // set bitmap: is_loaded=0, is_dirty=0, ref_bit stays 0
              // return curr_clock
      // else if hashmap.get(curr_clock)->ref_bit = 0;
      // if(curr_clock == 63): curr_clock = 0; else: curr_clock += 1;
    //return -1 (could not find a lock that could be acquired, all ref bit still set)
    


- **Implementation:**
    - Store each slot of the cache in a hashmap for easy access and mapping
        - fully associative cache design 
    - Have a data struct in each cache elem that holds the data on the heap instead of using the inode_disk to get it from memory. 
    - remove the bounce buffer implementation in inode.c
    - evict entries with clock replacement policy 
    - when system shuts down, flush the cache 
- **Rationale:**
    - We chose a Hashmap since we can easily access each member in constant time and the ordering is not important since we keep an index for the clock replacement policy
    - We chose to create a new struct for the buffer and a new file to maintain all the buffer specific variables and make our code isolated and cleaner
    - We chose to implement Clock replacement instead of LRU because we do not have to maintain a sorted list of recent uses which has high overhead so this is easier with the hashmap implementation. 
    - We chose to only keep one lock per cache elem struct instead of having S and X locks to make sure we synchronize reads while another process is writing to the same cache block for atomicity of the transactions
    - Dynamically allocate cache elems because its easier to maintain a clean hashmap
- **Complexity:**
    - **Time:** O(1) Access to specific Hashmap element in 1 to 1 mapping is constant time
    - **Memory:** O(1) Fixed size hashmap bc exactly 64 entries. 
- **Synchronization:** 
    - **Needed:** Yes - buffer_cache struct 
    - **Reasoning:**  Need to ensure that only one thread can modify the buffer cache struct at a time otherwise there might be race conditions for assigning a new buffer block or evicting the wrong pages etc. The entire struct is protected with a buf_lock that must be acquired and released before any read and writes to the buffer_cache variables. The lock must be acquired and released in:
        - filesys_done()
        - get_block_ptr(uint8_t sector_num); 
        - evict_next();
        - modifying the buffer cache struct
        - etc
    
    - **Needed:** Yes - sector/page specific locks
    - **Reasoning:** We use buf_lock and the is_loaded boolean bit to synchronize adding a new sector to the hash map or evicting the sector from the buffer but we also want to synchronize file operations to a specific page in the buffer when we already have the cache index to make sure multiple processes arent reading/writing simultaneously.  We have to have a lock_acquire/release hash_lock for the specific cache_elem struct before calling these methods:
        - void block_read(struct block*, block_sector_t, void*);
        - void block_write(struct block*, block_sector_t, const void*);
    
    - **Needed:** Yes - eviction policy (cross process access synch)
    - **Reasoning:** If a page is currently in use and being written to, we have a lock around the cache element to ensure we do not evict while it is in use. We also have a second chance reference bit to check if the clock policy can be applied. Given the element from the hashmap was acquired, both processes may lock_try_acquire the element lock but whichever process gets the lock first will not be interrupted until their read or write to buffer is completed
    
    - **Needed:** No - inside the block.h methods 
    - **Reasoning:** we will only be calling these methods within the critical sections of the cache_elem struct modifications so the hash_lock over the over the individual sector will also mean that only one process will be able to call the block_read and block_write method on that sector at a time so no modifications have to be made inside these methods
    
    - **Needed:** No - global file lock in syscall.c
    - **Reasoning:**  We are replacing the global file lock that prevents all file operations on any file to be more granular with the cache specific and sector specific locks. Thus we will be removing the synchronization lock in this file. 

// should be passing all the userprog tests at this point so make sure we arent failing any synch stuff here. 

## Changes: 
- using LRU with a list instead of a hashmap bc its easier to code and simialr functionality
- moved to the back of the list during get_entry call to order by most recently accessed
- added size parameters that will probably stay unused
- created a null element entry in case the entry is in the waiting queue or not found
- call cache_flush in filesys done instead of freeing
- process_waiting_list function that checks if the waiting list is empty, parses through all the entries to the first unpinned one, and evicts and frees if the lock can be acquired
- Currently Failing:
    - FAIL tests/userprog/wait-twice
    - FAIL tests/userprog/exec-missing
    - FAIL tests/userprog/write-bad-ptr
    - FAIL tests/userprog/no-vm/multi-oom
----------


# Extensible Files
## Data Structures and Functions

Data structures/variables added:

    /* in src/filesys/free-map.c */
    struct lock free_file_lock; // add a lock to synchronize changes to the free_map file
    /* in src/filesys/inode.c */
    struct lock open_inode_lock; // lock to synchronize the open inode list
    struct inode {
      ...
      struct lock inode_lock;
    }

Data structures/variables modified:

    /* In src/filesys/inode.c */
      struct inode_disk {
       block_sector_t start [REMOVE]
      - block_sector_t direct[124];  // we calculated the unused space left from the inode_disk size and then allocated it all to direct pointers
      - block_sector_t indirect; 512 * (128)
      - block_sector_t doubly_indirect*; 512 * 128 * 128
    }

Functions added:

    /* in syscall.c */
    uint32_t sys_inumber(int fd); // returns the inode number of the fd

Functions modified:

    /* in free-map.c/.h */
    void free_map_read(void);
    void free_map_create(void);
    void free_map_open(void);
    void free_map_close(void);
    bool free_map_allocate(size_t, block_sector_t*);
    void free_map_release(block_sector_t, size_t); // ensure synch with the lock before releasing the block 
    
    /* in inode.c (modify initialization/calloc calls to account for added pointers) */
    bool inode_create(block_sector_t sector, off_t length); // modify calloc for ptrs
    void inode_close(struct inode* inode); // free inode_disk ptrs before free(disk_inode)
    off_t inode_write_at & off_t inode_read_at; // both use ->
    static block_sector_t byte_to_sector(const struct inode* inode, off_t pos); // currently only reads from the inode start attribute to block size. Maybe pass in the direct/indirect/doubly indirect pointers from the write/read fns add an input parameter 
## Algorithms
    bool inode_create(block_sector_t sector, off_t length) {
      // create a new inode by calling inode_resize with a size of 0
    }
    
    void inode_resize(struct inode_disk* inode, off_t  newsize) {
    // handle direct pointers
    // initially, we need to memset all of the buffers to have a value of 0. 
    
    // have a temporary variable to store the file's original size
    
    /* Handle direct pointers. */
    for (int i = 0; i < 124; i++) {
      if (size <= BLOCK_SECTOR_SIZE * i && id->direct[i] != 0) {
        /* Shrink. */
        block_free(id->direct[i]);
        id->direct[i] = 0;
      } else if (size > BLOCK_SECTOR_SIZE * i && id->direct[i] == 0) {
        /* Grow. */
        id->direct[i] = block_allocate();
      }
    }
    
    
    // handle indirect pointers
    for (int i = 0; i < 128; i++) {
      if (size <= (124 + i) * BLOCK_SECTOR_SIZE && buffer[i] != 0) {
        /* Shrink. */
        block_free(buffer[i]);
        buffer[i] = 0;
      } else if (size > (124 + i) * BLOCK_SECTOR_SIZE) && buffer[i] == 0) {
        /* Grow. */
        buffer[i] = block_allocate();
      }
    }
    if (size <= 12 * BLOCK_SECTOR_SIZE) {
      /* We shrank the inode such that indirect pointers are no longer  required. */
      block_free(id->indirect);
      id->indirect = 0;
    } else {
      
      block_write(id->indirect, buffer);
    }
    id->length = size;
    
    
    
    // handle doubly indirect pointers
    for (int i = 0; i < 128; i++) {
      for (int j = 0; j < 128; i++) {
        if (size <= (124 + i + j) * BLOCK_SECTOR_SIZE && buffer[i] != 0 && buffer2[j] != 0) {
        /* Shrink. */
        block_free(buffer[i]);
        buffer[i] = 0;
        } else if (size > (124 + i + j) * BLOCK_SECTOR_SIZE) && buffer[i] == 0 && buffer2[j] != 0) {
        /* Grow. */
        buffer[i] = block_allocate();
      }
      }
    }
    
    
    // if while calling block_allocate check to see if there is an error
    
    sector = allocate_block();
    if (sector == 0) {
       inode_resize(id, id->length);
       return false;
    }
    // in this case, we can free the inodes into the file's original size. If we are unable to allocate blocks, we roll back the inodes to the file's original size
    
    
    }
    
    void inode_close(struct inode_disk* inode) {
    //loop through the inode starting from the back and free each element
    // 
    }
- **Implementation:**
    - In struct inode_disk we are removing the original block_sector_t start pointer and adding direct pointers, one indirect pointer and one doubly indirect pointer. This will support 8MiB files because 126 * 512 + 512 * (128) + 512 * 128 * 128 = 8,516,656 which is greater than 8 MiB.
    - We are also removing the inode_disk pointer from struct inode. 
    - In addition, we calculated the amount of unused space left from the inode_disk and filled it will direct pointers instead. This was calculated by 512 - 12 (the original number of direct pointers)*4 - 4 (size of indirect pointers)- 4 (size of doubly indirect pointers) = 114 extra bytes. This brings the total amount of direct pointers that we can store up to 114 + 12  = 126.
    - when we create a new inode, we call inode_resize with a size 0. We will then memset all of the buffers to have values of 0 in them so that none of them will be filled with garbage data. 
    - When we need to resize files, we will continue to call the inode_resize with the new size.
        - We will first check to see whether all of the direct pointers are filled. To do this, we loop through all of the direct pointers and check to see whether its data is 0 and its size is large enough. If so, then we grow the data.
        - We then check to see whether we need to call the indirect pointers. If we do, then we loop through all of the indirect pointers and check to see whether the direct pointer space they are pointing to is free. If so, we grow the data.
        - Similarly, we check to see whether we need to call the doubly indirect pointers. If we do, then we loop through all of the doubly and regular indirect pointer spaces. Whilst doing this, we ensure that we have two buffers- one for the indirect pointers and one for the direct pointers. We make sure that both of these buffers are free before growing our file. 
        - Whilst looping through all of the pointers, we make sure to error check before block_allocate is called. If block_allocate fails, then we need to revert our file. We do this by keeping track of the size of the file before we resized it, and if block_allocate fails, we just free the file back to its original size.
- **Rationale:**
    - as mentioned, we support 8MiB files with our pointers in inode_disk because the total size of blocks we can hold is greater than 8 MiB. 
        - By calculating the size that each pointer can point to, we are able to ensure that we can fit the maximum size of a file with direct, indirect, and doubly indirect pointers
    - We also filled out the rest of the unused space with direct pointers so that we can support larger file sizes, access memory faster and limit the need of having to access the indirect and doubly indirect pointers.
    - We make sure to error check that block_allocate works before we allocate a blocks to the file. This allows us to make sure that if it fails, we can revert the state back to its previous form. 
    - We also memset our buffers and direct values to be 0 initially. This guarantees that we won’t have garbage data in the arrays and so we can accurate run the if statement checks to ensure that it is empty.
- **Complexity:**
    - **Time:** O(n^2). In the worst case scenario, we will have to loop through the doubly indirect pointers. Since this contains a nested for loop, the worst case is O(n^2).
    - **Memory:** O(n) linear with respect to length of file
    
    **Synchronization:** 
    - **Needed:** **Yes -** in struct inode and list of open inodes
    - **Reasoning:** ****
        - Need to add a lock in struct inode to ensure synchronization of variables when multiple threads access the same inode
        - Need a lock to make sure we do not close one of the open inodes in `static struct list open_inodes;`  before another function is done using them. 
    Error stuffs:
    Calling→
        [ ] inode_init () 
        [ ] inode_create ()
        [ ] bytes_to_sectors
        [ ] inode_resize resize to 512
        [ ]  inode_open (sector=0)
        [ ] inode_write_at (inode=0xc010610c, buffer_=0xc010700c, size=512, offset=0) 
        [ ] byte_to_sector (id=0xc010720c, pos=0)
        [ ] inode_length (inode=0xc010610c)
        [ ] inode_create (sector=1, length=320)
        [ ] bytes_to_sectors (size=320)
        [ ] inode_resize (inode=0xc010760c, new_size=320)
        [ ] inode_write_at (inode=0xc010610c, buffer_=0xc010700c, size=512, offset=0)
        [ ] inode_close (inode=0xc010610c)
        [ ] …. similar pattern
        [ ] inode_resize (inode=0xc010980c, new_size=67516) // is this too big
        [ ] inode_deny_write (inode=0xc01061cc)
----------


# Subdirectories
## Data Structures and Functions

Data structures/variables added:

    struct inode_disk{
      ...
      bool is_dir; // true if inode is for a dir false if file
      int inode;  //inode number
    }

Data structures/variables modified:

    /* in process.h */
    struct process {
      ...
      struct dir* cwd; //set to file system root at process init
    } //child processes must inherit cwd of parent
    
    struct file_descriptor {
      ...
      struct dir* dir;  // for the directory/path to the file or the directory only if this is an fd for a dir
    };
    
    struct inode {
      ...
      struct lock dir_lock; // to synchronize directories
    }

Functions added:

    bool chdir(const char* dir); //change current working directory to dir


    bool mkdir(const char* dir); //create directory named dir
- make sure that the newly created dir has “.” and “..” entries


    bool readdir(int fd, char* name); //read directory entry from fd and store filename in name
    // change READDIR_MAX_LEN so that it supports file paths longer than 14


    bool isdir(int fd); //return true if fd is a directory, else false


    int inumber(int fd); //return inode number of fd


    /* in src/filesys/directory.c */ 
    struct dir* dir_path(char* path); // takes in a relative or absolute path (distinguished by whether the path begins with a '/' char) and returns the directory struct of the final directory

Functions modified:

    /* switch places we use dir lookup to include the relative path from the current directory */
    
    /* in pwd.c */
    static bool get_inumber(const char* file_name, int* inum) // store inode number of file_name into inum
    
    /* in syscall.c */
    int sys_exec(const char* ufile); // child process inherits cwd of parent
    int sys_open(const char* ufile); // look at cwd and open file in that dir (should also be able to open directories)
    int sys_remove(const char* ufile); // delete empty directories that are not the root (directory must not contain any files or subdirectories)
    // files cannot be created or opened in removed directory
    int sys_read(int handle, void* udst_, unsigned size); // disallow reading from directories
    int sys_write(int handle, void* usrc_, unsigned size); // disallow writing to directories
    
    /* in file.c */
    void file_close(struct file* file) // allow for relative and absolute paths
    
    /* in filesys.c */
    struct dir* dir = dir_open_root(); // should be changed in the following functions to use the dir_path function and 
    bool filesys_create(const char* name, off_t initial_size); /* also modify to have different logic if the file to be made is a directory, automatically add '.' and '..' but dont add to the children list */
        
    struct file* filesys_open(const char* name);
    bool filesys_remove(const char* name); // also modify to have different logic if the file to be made is a directory
- should delete empty directories (other than the root) in addition to regular files
## Algorithms
    struct inode* path_resolve(const char* name);
- **Implementation:**
    - `bool dir_lookup(const struct dir*, const char* name, struct inode**);`
    - everytime we are calling a syscall or function that passes in a file/dir path, we will use `dir_lookup` to resolve the path
    - recursively enter directories to look for files that match the name argument
    - return the inode struct set in `dir_lookup` (will return NULL if file is not found)
    - make sure to support `.` and `..` when resolving paths
    - syscalls would be modified so that they go through this path_resolve function before calling the associated functions to complete the sycall (by passing in the associated inode that is returned by the `path_resolve` function)
- **Rationale:**
    - This function will allow us to resolve a path if given either an absolute or relative path
- **Complexity:**
    - **Time:** O(n) - traverses directory entry, so time complexity depends on length of entry
    - **Memory:** O(1) - file name should be less than or equal to MAX variable
    
    static int get_next_part(char part[NAME_MAX + 1], const char** srcp);
- **Implementation:**
    - copy characters of file name part until a slash is reached
    - return 1 if successful, 0 at end of string, and -1 for too-long file name part
- **Rationale:**
    - This function will allow us to traverse file paths (for functions that take in char* file/directory paths)
- **Complexity:**
    - **Time:** O(1) - pointer is updated each time, so function just copies char* from path to part until “/”
    - **Memory:** O(1) memory is constant because file name lengths are set to max


    int sys_remove(const char* ufile);
- **Implementation:**
    - check if the char* corresponds to an empty directory and is not the root directory
        - check open_cnt to see if there are any open directories in use by a process. If so, do not remove the directory
    - delete empty directories that are not the root
    - maybe prevent cwd from being removed
- **Rationale:**
    - Checking whether or not we can delete a directory allows us to prevent any directories in use from being deleted and from losing any files or additional directories that have been placed in the directory we are trying to remove.
- **Complexity:**
    - **Time:** O(1) - constant time to remove files because they are all referenced by inode


- **Synchronization:** 
    - **Needed**: Yes. Lock
    - **Reasoning**: A lock for directories is needed (placed in struct inode) so that when we are modifying directories or calling functions that utilize directories, everything is synchronized.


    - **Needed:** No. Modifying inode/sectors
    - **Reasoning:** Should be taken care of by the buffer cache section since all the filesys calls should be redirected through the cache. We are taking care of the locking and synchronization there so the filesys can simple call remove for example and know that we will wait for the lock to be released by another process and acquired before removing a directory and its children. 
----------


# Concept check
> Answers only


1. write-behind
    1. periodically flush dirty blocks back to file system block device 
    2. use an idle thread to call timer_sleep so that the idle thread periodically wakes up and does the write-behind flush, then goes back to sleep
2. read-ahead
    1. prefetch blocks: read next block even though application has not asked for it yet. spawn a prefetching thread so the current thread isnt blocked by this. 


OH Ticket 11/15 3pm: 
Buffer Cache:

    1. What are the advantages/disadvantages to adding a bitmap to the cache struct to iterate over to ensure that we are doing stuff efficiently instead of adding multiple booleans into our cache element struct? We still have a cache struct lock and a cache element lock. 
        1. bitmap is okay but nobody does this implementation method oops, the is_loaded means it should work out though? 
        2. 
    2. In addition to calling flush_buffer() in filesys_done, the spec said adding the flush buffer to timer sleep is also good. I thought at some point there was a thing in lecture about not having I/Os in interrupts because we are taking away CPU time and this should be done in the background to have efficient checkpointing (not sure if this was in 186 tho). Is it ok in timer sleep though? 
        1. apparently we dont have to implement the write behind or read ahead it was just supposed to be part of the concept check, so conceptual
        2. follow up on ed
        3. create a new kernel thread and schedule intermittently (idk if this makes sense if we dont have prio scheduling and stuff? )
    3. Is the block size known and static or do we have to palloc 64*BlockSize at runtime? 
    4. 
    Subdirectories:
    1. Since the spec says that if the directory changes while it is opened, then some entries could be skipped or read twice, does that mean we do not need any synchronization when modifying directories?
        1. the reading will be synched bc inode_read_at is going through the buf cache which is synched
    2. Kinda confused on what readdir does
        1. from what I understand, it only reads 1 entry in a directory
        2. when called again, does it read another entry?
        3. should we include some type of variable that indicates that the entry has already been used by readdir func?
            1. expected behavior: return first entry?  https://github.com/Berkeley-CS162/group24/blob/d2bb0ed1df02328ccba704efb3f5fee8503e2adb/src/filesys/directory.c#L190
            2. account for the . and .. since those are also dirs
    3. To support syscalls where a filename is provided by the caller, would it be feasible to create a function that always returns the full path of the file to be used by the syscalls? → this full path function is called at the beginning of each syscall where a filename is provided → to be used throughout the rest of the function.
        1. resolve path fn. and separate relative and exact path. in inode disk only keep the relative path that the user passed in and verify the path_resolve(cwd + rel_path) exists every time. if its not resolving, the cwd was changed and we close the dir/file. if an absolute path was passed in store that and the resolve path should always work. 

OH Ticket 11/15 3pm: 
Buffer Cache:

    - Our implementation is to use a hashmap with a hash function on the block sector to get the corresponding entry in constant time. We were planning to allocate all the cache elements and data statically in the beginning of the cache program. However when replacing that entry during eviction, the hash of the new block sector will not match the hash of the old sector. How can we remove the element from the hash and reinsert another one? Or should we instead be deleting the entire malloced cache elem as we evict a record and then malloc a new one if we are creating a new entry? that seems like a lot of overhead? 
        -  how would the initial insert even work what would be the hash ahhh i fucked up 

