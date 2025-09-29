---
created: 2025-09-28T20:26
updated: 2025-09-28T20:48
---

## Intro
- Commands for monitoring and managing memory usage

### Monitor (memory)
```
# display current memory usage
free -h

# list PID of top 10 memory-consuming processes
ps aux --sort=-%mem | head -10
```

### Monitor (GPU)
```
# display GPU utilization
nvidia-smi

# Show GPU processes sorted by memory usage
nvidia-smi pmon -c 1
```

### Release memory
- memory types
	- **Pagecache**: cache frequent accessed files
	- **Dentries (Directory Entries)**: metadata about directory structures
	- **Inodes**: metadata about individual files
```
# free pagecache, dentries, and inodes
sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

# kill specific process
kill <PID>           # might be ignored
kill -9 <PID>        # force stop immediately
kill <PID1> <PID2>   # can kill multiple at once
```