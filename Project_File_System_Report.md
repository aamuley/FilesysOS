# Project File System Report

# Group 24
| Name         | Autograder Login | Email                     |
| ------------ | ---------------- | ------------------------- |
| Sydney Tsai  | student134       | sydneytsai@gmail.com      |
| Anusha Muley | student143       | anusha.muley@berkeley.edu |
| Cindy Yang   | student258       | cindycy@berkeley.edu      |
| Qi Li Yang   | student85        | qiyang2@berkeley.edu      |

# Changes
## Buffer Cache

Some of the changes we made to the buffer cache implementation were to use an LRU eviction policy with a list of the cache elements instead of a hashmap since it was easier to code and better functionality. So whenever we got the cache element, we moved it to the back of the list during get_entry call to order by most recently accessed. 
I also created a null element entry in case the entry is in the waiting queue or not found so we always have an allocated element to return. Additionally I realized we have to call cache_flush in filesys done instead of freeing the cache entirely. I also added a process_waiting_list function that checks if the waiting list is empty, parses through all the entries to the first unpinned one, and evicts and frees if the lock can be acquired. 
Overall a lot of the base design did change but the algorithms for finding/changing the cache elements was pretty similar

## Extensible Files

We had to modify our byte_to_sector function to compute based on the indices of the block sector for that byte instead of passing in a size and checking every block before the sector that we need.  We also used this logic to only grow beyond the current size instead of checking the already allocated space. 

Within inode_resize, we also split the function into two parts: inode_grow and inode_clear, one of which is responsible to growing the inodes and the other for shrinking. This allowed us to focus our code on either growing or shrinking and being able to call the respective functions if we need to roll it back.

When implementing inode_grow, we also used a ‘ceiling’ index that is the minimum of either the number of direct pointers and the index we need (when calculating direct pointers), or the minimum of 128 and the index (when calculating indirect and doubly indirect pointers). We did this instead of directly incrementing through the number of direct pointers we have, the number of direct pointers within the indirect pointers, and more. This allowed us to traverse exactly the number of pointers we needed instead of all of them. 

Instead of checking ‘if’ statements of the size of the blocks to decide whether or not to grow or shrink, we kept track of an index that decremented whenever we allocated space for the direct/ indirect/ doubly indirect pointers. Then, we could check whether free_map_allocate returned true for the nodes that we want and roll it back if needed.

When shrinking, we also changed a lot of the logic. Instead of just directly freeing the block and setting it to 0 within resize, we looped through all of the pointers that were over allocated and freed them accordingly. 


## Subdirectories

In inode.h, we added a lot of additional helper functions to simplify the code logic such as a function to check whether a directory is empty. Additionally created a path_resolve_final function that returns on the last part/file of the path passed in. In the file descriptor struct we used the file/directory variables to determine if the descriptor referred to a file or directory. We added support to go in between inodes, file/dir structs, and block sectors. Other than that, the majority of our implementation was the same as our design document.


----------
# Reflection

Anusha:
I think the Buffer Cache was tedious to design because of how many additions had to be made but overall a lot simpler with minimal debugging compared to what I was expecting. I think we spent a lot of time on extensible files since we tried to use the template that we coded during discussion but ended up having to change a lot of the logic and scrap a lot of that. I think we started the buffer cache early so I had more time to help on the other sections and had more stuff passing by the deadline which made it feel more productive and I think all of us go a lot better at efficiently debugging our programs and pinpointing errors. 

Sydney:
I worked mainly on the subdirectories portion of the project, and since it was dependent on the extensible files portion of the project, we found ourselves having to rush the debugging process of this last task. Also, once the subdirectories portion of the project was implemented, a lot of the other tests started to fail, which made the debugging process even more tedious. We quickly realized that we should have tried to complete the other tasks earlier on, as it would have allowed us to debug this portion more. Overall, I think we learned a lot from this project in terms of course material as well as time management.

Qi: I worked on implementing extensible files. When we started debugging it was rough because we didn’t know where our errors stemmed from which resulted in us resorting to rewriting a portion of the code to attempt to fix the issue. That worked but we ended up spending a lot of time on this part of the project which delayed testing and debugging the rest of the project. As a result we were in a big rush to finish subdirectories which was quite stressful. I think we should have speeded up the first part of the project so that we wouldn’t have had to cram the last part. This project made me realize how important task organization and having a timeline would be in completing a huge project like this. 

Cindy: I worked on implementing extensible files and testing. I think implementing extensible files was quite difficult because we tried to follow the logic we talked about during design review and discussion. However, it did not pan out as we had expected and was really difficult for us to debug.  This caused us to spend a lot of time on this function and not on the other parts such as subdirectories which was a little bit stressful. In addition, it was a little rushed towards the end with testing files because we spent a long time trying to debug the previous tasks. Overall, we could have timelines the project better to make sure there was a set expectation on when certain tasks would be finished. 

----------
# Testing

Our buffer cache rate test tests the buffer cache’s effectiveness. We do this by determining the cache hit rate of a cold cache and then comparing it with another read. If the cache hit rate is effective, then the second read should have a lower miss rate / higher hit rate. 

In our test case, we first opened our file, wrote to it, and then cleared the cache. On our first read, we iterated around 40 times (since 2x this is greater th the 64 block size but not greater than 128) and read from the file every time. From there, we calculated the respective hit and miss rates of the reads. Next, we read through the file for a second time by iterating sequentially and calculated the updated hit and miss rates. We then compared these numbers to the first read to determine whether it was actually more effective.


    - (cache_rate) begin
    - (cache_rate) make "cache_rate_i"
    - (cache_rate) create "cache_rate_i"
    - (cache_rate) open "cache_rate_i"
    - (cache_rate) clearing cache
    - (cache_rate) buffer cache effective
    - (cache_rate) end


![](https://paper-attachments.dropboxusercontent.com/s_CDE249D145AFC37F71C8DB2018F651FABC70981D2887E285F9B7F1654B242A20_1702021810594_image.png)


Two non-trivial potential bugs

    1. Not having an accurate buffer. If my kernel didnt cache the elements correctly and had to block_read the elements instead, the test output would be the cache doesnt have an effective hit rate. 
    2. inefficient cache flushing: if our cache flushes are inefficient, it could lead to delayed writes or incomplete updates. This could cause outdated data to be read into our cache which we don’t want. In these cases, it could increase our miss rates since the data would be inaccurate and cause our buffer cache to be less effective than it should be. 

In the second test case, we are adding random bytes to the blocks one by one with write syscalls and then reading the bytes one by one with read syscalls. Because these are going 1 byte at a time we can test the ability of the cache to limit the writes to the block directly and instead only write whole block increments. 


    1. If my kernel did not limit the size of the buffer cache to be 64 blocks, there would not be enough space in the heap to allocate the new blocks and other processes such as the inode write/create methods would fail. Because of the size of the bytes being written there is a chance it could take up a lot of space with constant resizes
    2. Another kernel bug we could face is if we have to read in the entire block before writing with the bounce buffer and we were reading from disk, it would be a high overhead with the amount of reads and writes to make this efficient. if we didnt have enough space or the write count is too high the test would return that the count is above the set threshold


