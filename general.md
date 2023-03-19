# General Stuff To Take Note Of

## EXPORT\_SYMBOL

The macro EXPORT\_SYMBOL can be found everywhere in the linux source code. It is used to make internal kernel functions available to dynamically loaded kernel modules; if you write your own kernel module, you can directly call functions exported using this macro.  
