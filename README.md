Download Link: https://assignmentchef.com/product/solved-cs330-assignment-3
<br>
In this assignment, you will be implementing a simple debugger in gemOS. Please go through the following resources to understand how a debugger such as gdb works in Linux.

<ul>

 <li>http://www.alexonlinux.com/how-debugger-works</li>

 <li>https://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1</li>

 <li>https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints</li>

 <li>https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugginginformation</li>

</ul>

<h1>2      Overview</h1>

Breakpoints are used to stop the execution of a process at a particular location so that debugger can analyse the state of the process being debugged at that point.

<ul>

 <li>INIT (present in gemOS/src/user/init.c) will act as a debugger.</li>

 <li>INIT will fork a new child process. This child process will act as the process to be debugged (debuggee). Thus the debugger and debuggee will share the same code segment.</li>

 <li>Debugger process will use system calls such as set breakpoint(), remove breakpoint() to trace its child process.</li>

</ul>

We don’t want our debuggee process to finish executing before debugger gets a chance to set breakpoints or query register values. This means that the debuggee process shouldn’t start running as soon as it gets created. We’ve modified the fork implementation for this purpose. Refer: [debugger on fork in gemOS/src/debug.c and do fork in gemOS/src/entry.c]

Figure 1: High level overview of the working of debugger

<h1>3      Given</h1>

<ul>

 <li>You have been given a sample user-space implementations of the debuggers in gemOS/src/user/test/. You can copy the contents from any of these to gemOs/src/user/init.c and test the debug functionalities. However, you need to implement the system calls that the debugger will need to work properly.</li>

 <li>GemOS has already been configured to call int3 handler when INT3 (hexcode = 0xCC) interrupt is triggered. You just need to fill in the definition of int3 handler in gemOS/src/debug.c.</li>

 <li>The file gemOS/src/debug.c contains all the templates for the system call implementations you need to complete.</li>

</ul>

<h1>4       Assumptions</h1>

<ul>

 <li>In gemOS, code segment is shared between parent process and child process. Therefore, when some code segment is written by parent or child, the changes in the code segment will be reflected both in the parent and the child.</li>

 <li>A debugger will fork only one debuggee process and this debuggee process will not fork any child of its own. Thus, there will be only two processes, debugger and debuggee.</li>

 <li>Debugger (parent) will not call any function on which breakpoint has been set.</li>

 <li>Debuggee will not call any debugger system calls.</li>

 <li>To schedule another process (by replacing the calling process), schedule(P) is invoked, where P is the exec context of the process which is scheduled. Note that, any code written after the schedule() call will not be executed (even when the outgoing process is scheduled back). Consider the following system call snippet in gemOS.</li>

</ul>

int some_syscall(){ struct exec_context *ctx = some_other; printk(“Hello gemOS
”); schedule(some_other_ctx); printk(“Bye gemOS
”); }

If process P1 makes the above system call, “Bye gemOS” will not be printed. This is because, in gemOS, the process P1 will resume in userspace (after the system call invocation).

<h1>5         Task 1: Basic Debugger [80 marks]</h1>

In this task you will be implementing following system calls and functions to support debugging. Each sub-task is a functionality that you are expected to implement.

<h2>5.1       int become debugger( void *addr)</h2>

<strong>Description of args: addr </strong>: The address of the <em>do </em><em>end handler </em>function, which is present in user/init.c.

<strong>Description of functionality: </strong>Initialize the data structures related to the debugger inside this function. See the description for set breakpoint. The <em>do end handler </em>function address is passed as the argument <strong>addr</strong>, which is to be stored in the <em>end handler </em>field in the parent context’s struct debug info (see src/include/debug.h). If a breakpointed function has <em>end breakpoint enable </em>flag set (see below), then control should pass from the end of that breakpointed function to this <em>do end handler </em>function. In this <em>do end handler </em>function, <strong>INT3 </strong>interrupt should be raised. This will call the interrupt handler (see <em>int3 handler</em>) which should then schedule the parent. When child is scheduled back again later, it should continue from the next line in <em>do end handler </em>function. The <em>do end handler </em>function on returning, passes control to the place where the breakpointed function would have returned to. <strong>Return value: </strong>0 on success. -1 on failure.

<h2>5.2       int set breakpoint(void *addr, int flag)</h2>

<strong>Description of args: addr </strong>: The address of function where breakpoint is to be set <strong>flag </strong>: Can be either 0 or 1. Flag indicates if the debugee function at the end should return normally or go to end handler

<strong>Description of functionality: </strong>Set a breakpoint on debuggee process at address <strong>addr</strong>. You have to manipulate debuggee’s address space so that whenever the instruction at <strong>addr </strong>is executed, INT3 would be triggered.

If <strong>flag </strong>is set, then you must ensure that when the breakpointed function is about to return, control instead passes to the <em>do </em><em>end handler </em>function, rather than returning. You should save the flag in the end breakpoint enable member inside struct breakpoint info (see src/include/debug.h)

Save the relevant information about the breakpoint such as the address, end breakpoint flag and breakpoint number in parent process’s execution context.

Debugger process’s exec context (gemOS/src/include/context.h) contains a field struct debug info *dbg and struct debug info contains a field struct breakpoint info *head as shown below.

struct exec_context{ …

struct debug_info *dbg; … };

struct debug_info{ struct breakpoint_info *head;        // head of breakpoint list

};

struct breakpoint_info{

u32 num;            // – breakpoint number u64 addr;    // – address on which breakpoint is set struct breakpoint_info *next;    // – pointer to next node in breakpoint list int end_breakpoint_enable; // – whether control should pass to end_handler // at breakpointed function return

};

You should store information about new breakpoint in new struct breakpoint info object and insert it at the tail of the list pointed by <em>head</em>. Initially, <em>head </em>is NULL.

<strong>Example: </strong>Suppose, debugger makes following system calls. Assume MAX BREAKPOINT is 2. Breakpoint numbers assigned are shown in brackets: set breakpoint(addr1); (breakpoint number assigned: 1) set breakpoint(addr2); (breakpoint number assigned: 2) remove breakpoint(addr1); (breakpoint number 1 removed) set breakpoint(addr1); (breakpoint number assigned: 3) set breakpoint(addr1); (fails since MAX BREAKPOINT is 2)

Note that your breakpoint implementation must support recursion i.e. If we set a breakpoint on foo(), it should trigger an interrupt every time foo() is called, till the breakpoint is disabled or removed.

<strong>Assumptions:</strong>

<ul>

 <li>Breakpoint is always set only on functions.</li>

 <li><strong>addr </strong>is always a valid address i.e., the first instruction of a function.</li>

 <li>Maximum number of breakpoints set at any time (including both enabled as well as disabled) will be specified by MAX BREAKPOINTS (present in gemOS/src/include/debug.h).</li>

 <li>If MAX BREAKPOINTS number of breakpoints are already set and set breakpoint() is called again, return -1 (error).</li>

 <li>If a breakpoint corresponding <strong>addr </strong>already exists, return 0 (success) and set its status to <em>enabled</em>.</li>

 <li>The breakpoint number is incremented if and only if set breakpoint was successful.</li>

</ul>

<strong>Return Value: </strong>0 on success, -1 on error

<h2>5.3      int remove breakpoint(void *addr)</h2>

<strong>Description of args: addr </strong>: The address corresponding to the breakpoint to be removed

<strong>Description of functionality:</strong>

Once a debugger is done with the process of analysing the state of debuggee

at a particular address, the debugger generally removes the breakpoint from that address so that, when debuggee’s execution reaches this address again, no breakpoint is generated. In this system call, you have to implement this feature of removing any breakpoint.

You are required to find the breakpoint entry corresponding to the address specified by <strong>addr </strong>in list of breakpoints pointed to by <em>head </em>which is accessible from <em>dbg </em>pointer which is part of the exec context structure. You have to completely remove the information about this breakpoint that you have maintained. Also, make sure that when in the future the child process’s execution reaches <strong>addr</strong>, INT3 shouldn’t be generated anymore.

If no breakpoint corresponding addr exists, return -1 (error).

If remove breakpoint is called on a function that is currently active on the call stack of debuggee process (function has been called but not returned yet), AND its end breakpoint enable flag is set to the value 1, then also remove breakpoint should return -1 (error).

For example, suppose f1 has a breakpoint with end breakpoint enable set to 1, and f2 is another function with a breakpoint. Assume we are at the breakpoint of f2(). Let the call stack be main() → f1() → f2(). Note that f1() is still on the call stack. So here if user tries to do remove breakpoint(f1), it should fail. Note that this rule does not apply for functions with end breakpoint enable value 0.

<strong>Assumptions:</strong>

(A) Address specified by <strong>addr </strong>can be any address.

<strong>Return Value: </strong>0 on success, -1 on error

<h2>5.4      s64 wait and continue()</h2>

<strong>Description of functionality:</strong>

Stop execution of the debugger process and continue execution of the debugee process. The debugger should go to the WAITING state and the debugee should go to the READY state and should be scheduled now.

The Debugger process’s execution will be resumed in the following scenarios:

<ul>

 <li>INT3 interrupt has occurred because of the debugee process hitting a breakpoint.</li>

 <li>Debugee process has exited.</li>

</ul>

<strong>Return Value:</strong>

<ul>

 <li>If the debuggee process has reached a breakpoint, return the address of the breakpoint.</li>

 <li>If debuggee has exited, return CHILD EXIT (defined in gemOS/src/include/debug.h)</li>

 <li>If an error occurred, return -1.</li>

</ul>

<h2>5.5      void int3 handler()</h2>

<strong>Description of functionality:</strong>

Whenever an INT3 instruction (Byte value: 0xCC) gets executed, interrupt number 3 is generated and control goes to the interrupt 3 handler. We have already done the configuration so that int3 handler() is called when an INT3 instruction gets executed. You just have to fill the definition of this function.

If you have setup breakpoint correctly, control will reach int3 handler when breakpoint is encountered in the debugee process. In int3 handler, you have to stop the execution of the debugee process and start execution of the debugger process, so that the debugger can analyse the current state of the debuggee.

If the <em>end breakpoint enable </em>flag of the breakpointed function is set, then the aforementioned control transfer should also happen when returning from the breakpointed function. In int3 handler, you should figure out where the control came from (from beginning of function or from <em>end handler</em>) and handle accordingly.

<strong>Return Value: </strong>0 on success, -1 on error

<h2>5.6         void debugger on exit(struct exec context *ctx)</h2>

Note that, this isn’t a system call but a function called from the process exit handler (do exit in gemOS/src/entry.c) executed whenever a process exits.

<strong>Description of functionality:</strong>

When a debugger exits, you have to make to sure that memory allocated by debugger is freed. e.g, Memory allocated for storing information about breakpoints. When a debuggee exits, debugger should be scheduled and informed that debuggee has exited (See the description of wait and continue).

<strong>Assumption: </strong>Debugger will never exit before debuggee.

<h2>5.7       int info breakpoints(struct breakpoint *bp)</h2>

<strong>Description of args: bp </strong>is an array of struct breakpoint defined in gemOS/src/include/debug.h. Size of this array is MAX BREAKPOINTS

<strong>Description of functionality: </strong>info breakpoints is used by debugger to get information about all breakpoints currently set. For each breakpoint, following information is provided.

<ul>

 <li>Break point address</li>

 <li>End-Breakpoint flag i.e. whether breakpoint at end-of-function is enabled</li>

</ul>

(1) or disabled (0).

<ul>

 <li>Break point number</li>

</ul>

Array of struct breakpoint i.e. <strong>bp </strong>will be passed from the user space as the argument to info breakpoints(). You have to fill information about the breakpoints in this array of structure and return the number of set breakpoints. Entries in bp should be stored as per the increasing values of breakpoint number i.e. breakpoint with lowest breakpoint number should be stored in bp[0], the second lowest in bp[1] and so on.

<strong>Return: </strong>Number of breakpoints set (0 to MAX BREAKPOINTS) in case of success.

-1 in case of error.

<h2>5.8       int info registers(struct registers *reg)</h2>

<strong>Description of args: </strong>Address of an element of type struct registers defined in gemOS/src/user/ulib.h amd gemOS/src/include/debug.h

<strong>Description of functionality: </strong>info registers is used to find the values present in registers just before a breakpoint occurs during the execution of debuggee. Address of a struct register type variable will be passed from user space as the argument of info registers. You are required to fill the register values of child process just before the INT3 instruction gets executed (i.e. just before breakpoint occurred) into this structure passed as argument.

<strong>Return: </strong>0 on success, -1 on error

<strong>Testing: </strong>While testing the correctness of register state obtained using info registers, we will only check the values corresponding the following registers:

<ol>

 <li>RIP, RSP, RBP, RAX</li>

 <li>Registers used to store arguments passed to functions i.e. RDI, RSI, RDX, RCX, R8, and R9</li>

</ol>

<h1>6          Task 2: Adding stack back-trace functionality [20 marks]</h1>

In this question you will be implementing the following system call.

<h2>6.1       int backtrace(unsigned long *bt)</h2>

<strong>Description of args:</strong>

<strong>bt </strong>points to an array of u64 elements (i.e., each element = 8bytes) of size

MAX BACKTRACE (gemOS/src/include/debug.h). Result of backtrace will be loaded in this array.

<strong>Description of functionality: </strong>Backtrace allows a debugger to analyse the call stack of the debuggee process. In this question you will be implementing a simple version of backtrace. This simple version of backtrace will only report the return addresses pushed on to the stack as one function calls another function. All the return addresses (starting from the function where breakpoint is set) till the main function should be filled in the <strong>bt </strong>array.

<strong>Example: </strong>Suppose main() calls fnA and fnA calls fnB. Suppose breakpoint is set on fnB (with end breakpoint enable’s value equal to 0). In this case, bt[0] should contain the address of first instruction of function on which breakpoint occurred i.e. fnB in this case. Return address in fnA (saved in stack when fnB was called) should be loaded in bt[1]. Return address in main (saved in stack when fnA was called) should be loaded in bt[2]. Return address stored in main’s stack frame is END ADDR (defined in gemOS/src/include/debug.h). When this address is encountered, the back tracing should stop. END ADDR should not be included in <strong>bt</strong>. So, for the above example, 3 values will be stored in bt[0], bt[1], bt[2] and the backtrace system call should return 3.

Note that, if end breakpoint enable on fnB had the value 1, and breakpoint hit at the end of fnB, then the backtrace will contain only 2 entries, i.e., the function address of fnB will not be in the backtrace, since fnB has exited.

<strong>Return: </strong>On success, return the number of return addresses that are part of the backtrace. In case of error, return -1.

<strong>Assumptions</strong>: There will be at most MAX BACKTRACE number of return addresses as part of backtrace testing.

<strong>7        Helper API’s:</strong>

These are some of the scheduling and memory related API’s provided by gemOS that maybe helpful to you.

<ul>

 <li>struct exec context *get ctx by pid(u32 pid); – Given a pid, it returns a pointer to the exec context struct associated with it.</li>

 <li>struct exect context *get current ctx(void); – Returns the pointer to exec context struct associated with the current running process.</li>

 <li>struct exect context *pick next context(struct exec context *ctx); – Returns pointer to the execcontext struct of the next process that can be scheduled. Pointer to execcontext struct of the current running process is passed as the argument.</li>

 <li>void schedule(struct exec context *ctx); – Schedules the process corresponding to the passed exec context</li>

 <li>void *os alloc(u32 size); – Allocates a memory region of size Note that you cannot use this function to allocate regions of size greater than 2048 bytes.</li>

 <li>void os free(void *ptr, u32 size); – De-allocates the memory region allocated by os alloc.</li>

</ul>

<h1>8       Modifications allowed</h1>

<ul>

 <li>Among non-header files, you can only modify gemOS/src/debug.c</li>

 <li>You can’t modify any header file apart from gemOS/src/include/debug.h (C) You can’t remove anything from gemOS/src/include/debug.h</li>

 <li>You can add new structures (and new members in existing structures) in gemOS/src/include/debug.h</li>

 <li>struct debug info provided by us contains a field struct breakpoint info *head. Don’t remove/rename it.</li>

</ul>

<h1>9      Testing</h1>

<ul>

 <li>Please make sure that you are compiling and testing your code inside the provided docker environment.</li>

 <li>In the GemOS terminal (accessed using the telnet command), you can type init to execute the user space process (debugger).</li>

 <li>A sample debugger code is available in gemOS/src/user/init.c</li>

 <li>You need to write your test cases in c to validate your implementation.</li>

 <li>The sample test-cases (in gemOS/src/user/test) can be copied into c to use them.</li>

 <li>Test your implementation by running these tests (and also your own testcases) and ensuring that the expected output is observed.</li>

 <li>The user and kernel code are compiled into a single binary file, i.e., kernel when built using ’make’ from the src directory.</li>

</ul>