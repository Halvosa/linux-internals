## Tracing the Kernel



ftrace is a tracing utility that lives in the kernel. One can interact with it by writing and reading from different files in the tracefs filesystem mounted at /sys/kernel/tracing. This is quite cumbersome, however, so there is a command called trace-cmd that makes it a little easier. 



## Navigating the Code Base

First we need to clone the linux repo.
```
git clone https://github.com/torvalds/linux.git
```

Install ctags for fast code base navigation.
```
sudo apt install -y ctags
```

Create a ctags index of the repo. This operation takes a minute or two. When done, you will get a "tags" directory.
```
cd linux
ctags -R
```

We will use vim as our editor. Open the repo root directory with vim. Vim will automatically register the tags directory.
```
vim .
```


We can jump to a tag by using the vim's tag command:
```
:tag start_kernel
```

Often there are many hits for a tag name. We can get a list of all of them with
```
:ts TAG_NAME
```

The list contains a column named "kind" that tells if the hit is a function definition, prototype, variable definition and so on. To list the meaning of each "kind letter": 
```
$ ctags --list-kinds=c
c  classes
d  macro definitions
e  enumerators (values inside an enumeration)
f  function definitions
g  enumeration names
l  local variables [off]
m  class, struct, and union members
n  namespaces
p  function prototypes [off]
s  structure names
t  typedefs
u  union names
v  variable definitions
x  external and forward variable declarations [off]
```
