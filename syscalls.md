# Linux System Calls

## Finding Syscall Source Code

Syscall function definitions are defined using the SYSCALL\_DEFINE macros. The command `:ltag /^SYSCALL_DEFINE.` followed by `:lwindow` gives a searchable list containing all syscall definitions. For example, searching for "clone" (using regular '/' vim search) gives:
<pre>
216 ipc/mqueue.c|^\VSYSCALL\_DEFINE2(mq\_notify, mqd\_t, mqdes,\$| SYSCALL\_DEFINE2
217 ipc/msg.c|^\VSYSCALL\_DEFINE2(msgget, key\_t, key, int, msgflg)\$| SYSCALL\_DEFINE2
218 kernel/capability.c|^\VSYSCALL\_DEFINE2(capget, cap\_user\_header\_t, header, cap\_user\_data\_t, dataptr)\$| SYSCALL\_DEFINE2
219 kernel/capability.c|^\VSYSCALL\_DEFINE2(capset, cap\_user\_header\_t, header, const cap\_user\_data\_t, data)\$| SYSCALL\_DEFINE2
220 kernel/fork.c|^\VSYSCALL\_DEFINE2(<b>clone</b>3, struct clone\_args \_\_user \*, uargs, size\_t, size)\$| SYSCALL\_DEFINE2
221 kernel/futex/syscalls.c|^\VSYSCALL\_DEFINE2(set\_robust\_list, struct robust\_list\_head \_\_user \*, head,\$| SYSCALL\_DEFINE2
222 kernel/groups.c|^\VSYSCALL\_DEFINE2(getgroups, int, gidsetsize, gid\_t \_\_user \*, grouplist)\$| SYSCALL\_DEFINE2
223 kernel/groups.c|^\VSYSCALL\_DEFINE2(setgroups, int, gidsetsize, gid\_t \_\_user \*, grouplist)\$| SYSCALL\_DEFINE2
224 kernel/module/main.c|^\VSYSCALL\_DEFINE2(delete\_module, const char \_\_user \*, name\_user,\$| SYSCALL\_DEFINE2
225 kernel/nsproxy.c|^\VSYSCALL\_DEFINE2(setns, int, fd, int, flags)\$| SYSCALL\_DEFINE2
[Location List] ltag /^SYSCALL\_DEFINE.                                                                                                    220,34          36%
/clone
</pre>

## Implementation of Calling Convention


The below is taken from `tools/include/nolibc/arch-x86_64.h`:
```
/* Syscalls for x86_64 :
 *   - registers are 64-bit
 *   - syscall number is passed in rax
 *   - arguments are in rdi, rsi, rdx, r10, r8, r9 respectively
 *   - the system call is performed by calling the syscall instruction
 *   - syscall return comes in rax
 *   - rcx and r11 are clobbered, others are preserved.
 *   - the arguments are cast to long and assigned into the target registers
 *     which are then simply passed as registers to the asm code, so that we
 *     don't have to experience issues with register constraints.
 *   - the syscall number is always specified last in order to allow to force
 *     some registers before (gcc refuses a %-register at the last position).
 *   - see also x86-64 ABI section A.2 AMD64 Linux Kernel Conventions, A.2.1
 *     Calling Conventions.
 *
 * Link x86-64 ABI: https://gitlab.com/x86-psABIs/x86-64-ABI/-/wikis/home
 *
 */

#define my_syscall0(num)                                                      \
({                                                                            \
	long _ret;                                                            \
	register long _num  __asm__ ("rax") = (num);                          \
	                                                                      \
	__asm__  volatile (                                                   \
		"syscall\n"                                                   \
		: "=a"(_ret)                                                  \
		: "0"(_num)                                                   \
		: "rcx", "r11", "memory", "cc"                                \
	);                                                                    \
	_ret;                                                                 \
})

#define my_syscall1(num, arg1)                                                \
({                                                                            \
	long _ret;                                                            \
	register long _num  __asm__ ("rax") = (num);                          \
	register long _arg1 __asm__ ("rdi") = (long)(arg1);                   \
	                                                                      \
	__asm__  volatile (                                                   \
		"syscall\n"                                                   \
		: "=a"(_ret)                                                  \
		: "r"(_arg1),                                                 \
		  "0"(_num)                                                   \
		: "rcx", "r11", "memory", "cc"                                \
	);                                                                    \
	_ret;                                                                 \
})

#define my_syscall2(num, arg1, arg2)                                          \
({                                                                            \
	long _ret;                                                            \
	register long _num  __asm__ ("rax") = (num);                          \
	register long _arg1 __asm__ ("rdi") = (long)(arg1);                   \
	register long _arg2 __asm__ ("rsi") = (long)(arg2);                   \
	                                                                      \
	__asm__  volatile (                                                   \
		"syscall\n"                                                   \
		: "=a"(_ret)                                                  \
		: "r"(_arg1), "r"(_arg2),                                     \
		  "0"(_num)                                                   \
		: "rcx", "r11", "memory", "cc"                                \
	);                                                                    \
	_ret;                                                                 \
})
```
