# General Stuff To Take Note Of

## Compiler Stuff

* The keyword `register` is deprecated. Originally, the keyword was used in a variable declaration as a hint to the compiler that the variable will be accessed often and should therefore be stored in a processor register instead of memory (for example, instead of the stack). Today's compilers are smart enough to decide for themselves.

* 

The compiler may or may not follow that hint.
## EXPORT\_SYMBOL

The macro EXPORT\_SYMBOL can be found everywhere in the linux source code. It is used to make internal kernel functions available to dynamically loaded kernel modules; if you write your own kernel module, you can directly call functions exported using this macro.  

##

Show the filesystem superblock
```sh
dumpe2fs /dev/<volume>
```

## C language

* goto keyword
* anonymous structs and unions
* asmlinkage
