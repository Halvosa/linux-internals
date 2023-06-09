# Virtual File System (VFS)

The linux virtual file system is a software layer in the kernel that provides a unified interface for interacting with devices. It implements system calls such as open, read, write, stat, chmod and so on.

It turns out that almost all device can be categorized into only a few different categories.  

Let us start with a simple system call: stat. What happens when we run the stat command from bash?
```

```


 We know it must call the stat system call, but what exact C code identfier is called inside the kernel on the perimiter between user space and kernel space? 

trace-cmd -p function_graph -g __sys_newfstat

```
sudo trace-cmd record -p function_graph --max-graph-depth 10 -g __x64_sys_newfstat -F stat test.tx
```

```
echo "linux" > test
sudo trace-cmd record -p function_graph --max-graph-depth 2 -e syscalls -F stat test
sudo trace-cmd report | less
``

Reading the report, you will realize that the C identifiers for the system calls starts with "__x64_sys_". By searching for the regex "_sys_.*stat", we find:
```
stat-105448 [003] 95503.957132: funcgraph_entry:                   |  __x64_sys_newfstat() {
stat-105448 [003] 95503.957132: funcgraph_entry:        2.219 us   |    vfs_fstat();
stat-105448 [003] 95503.957135: funcgraph_entry:        0.772 us   |    cp_new_stat();
stat-105448 [003] 95503.957136: funcgraph_exit:         3.960 us   |  }
stat-105448 [003] 95503.957136: funcgraph_entry:                   |  syscall_exit_work() {
stat-105448 [003] 95503.957136: sys_exit_newfstat:    0x0
stat-105448 [003] 95503.957137: funcgraph_exit:         0.504 us   |  }
stat-105448 [003] 95503.957137: funcgraph_entry:                   |  exit_to_user_mode_prepare() {
stat-105448 [003] 95503.957137: funcgraph_entry:        0.229 us   |    fpregs_assert_state_consistent();
stat-105448 [003] 95503.957137: funcgraph_exit:         0.683 us   |  }
```

We already get a reference to VFS: the function vfs_fstat.   Now that we have found our entrypoint into the kernel, let us increase the max depth of the function graph tracer and see what we get:

