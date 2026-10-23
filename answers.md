# HW1 answers

Fill in this file and hand it in with your code. Replace each
`(your answer)` with two or three sentences. Keep the question numbers
(part.question), and do not delete the questions.

- **Name:**
- **Machine and environment you tested on** (CADE lab machine with QEMU, or Bochs):
- **How far did you get?** (for example "all six parts work", or "stuck in part 4, block B"):


## Part 1: Entering VMX operation

### Question 1.1

> In Step 1, why does the function `return` when VMX is missing instead
> of calling `panic()`?

**Answer:** VT-x is an optional feature: xv6 itself doesn't need it,
and only the hypervisor (`runvm`) uses it. `panic()` would halt the
whole kernel at boot, so any machine without VMX could not run xv6 at
all. Returning instead leaves `cpus[id].vmx_ok` at 0, so the system
still boots normally (as with `QEMUKVM=`), and `vmcreate()` and
`vmrun()` check that flag and simply refuse to create or run VMs on
that CPU.

### Question 1.2

> In Step 2, on most machines the firmware has already enabled and
> locked VMX. Why does the code still handle the unlocked case instead
> of just failing?

**Answer:** Not all firmware locks the MSR. Virtual firmware such as
Bochs's BIOS, and some real BIOSes, leave IA32_FEATURE_CONTROL
unlocked, and there VMXON would fault because it requires the lock bit
to be set. While the MSR is unlocked we are allowed to write it, so we
set the lock bit and "enable VMX outside SMX" ourselves and VMX works.
Just failing would needlessly disable the hypervisor on machines that
are perfectly able to run it; we only give up when the firmware has
locked VMX off, which we cannot change.

### Question 1.3

> In Step 3, what would happen if we skipped `lcr4(cr4)` and went
> straight to `VMXON`?

**Answer:** The new value would only exist in the local variable `cr4`,
so the real CR4 would still have VMXE = 0 (and might not obey the
FIXED0/FIXED1 rules). VMXON executed with CR4.VMXE = 0 raises an
invalid-opcode exception (#UD) instead of returning an error code, so
the `vmxon()` wrapper never gets to report a failure: the fault happens
in the kernel, and xv6 panics at boot on every CPU.

### Question 1.4

> In Step 4, why does each CPU need its own VMXON region? What could go
> wrong if all CPUs shared one?

**Answer:** VMX operation is per logical processor, and while a CPU is
in VMX operation it may use its VMXON region to hold its own internal,
implementation-specific VMX state; the SDM (27.11.5) therefore tells
software to use a separate region for each logical processor. If all
CPUs shared one page, they would overwrite each other's private state
in it, which could make VMX behave unpredictably on any of them. A
shared page would also be fragile to manage: for example, if one CPU's
VMXON failed and it freed the page, the other CPUs would still be
using freed memory.

## Part 2: Creating a VM, its memory, and the EPT

### Question 2.1

> An ordinary x86-64 page table translates virtual addresses to physical
> addresses. What does the EPT translate, and who controls it? What
> would go wrong if the hypervisor gave the guest direct access to the
> host's physical memory instead?

**Answer:** The EPT translates guest-physical addresses (what the guest believes
is RAM) to host-physical addresses (real memory), as a second stage
after the guest's own page tables. The hypervisor controls it: it
builds the tables and gives the CPU the EPTP through the VMCS, and the
guest can neither see nor change it. With direct access to host
physical memory, the guest could read or overwrite anything, including
the host kernel, other processes and other VMs, so there would be no
isolation; it would also expect RAM at addresses the host already
uses for itself.

### Question 2.2

> In Step B7 we zero every page of guest memory, and in `eptwalk()` we
> zero every new table. These are two different reasons to zero. State
> each, and say what the guest or the CPU would see if we skipped it.

**Answer:** Guest pages (Step B7) are zeroed so the guest cannot see what was
there before: kalloc() returns recycled pages full of junk or of
leftover data from the kernel or other processes. Without it the guest
could read host data (a security leak), its behavior would depend on
what ran earlier, and helloguest's uninitialized globals would not be
zero, since a flat binary has no loader to clear them. New EPT tables
are zeroed for a different reason: an all-zero entry means "not
present". Without it, junk entries (kfree() fills pages with 0x01, so
the R bit is set) would look present to the CPU and to eptwalk(), and
would point at bogus physical addresses, giving the guest random host
memory, causing EPT misconfigurations, or making eptfree() free pages
that were never ours.

### Question 2.3

> In Step B8, why does `vmcreate()` copy the image page by page through
> `eptwalk()`, instead of one `memmove` of `len` bytes to the guest's
> first page?

**Answer:** Guest-physical memory is contiguous, but the host pages backing it
are not: Step B7 got each one separately from kalloc(), so they are
scattered and in no particular order. A single memmove of len bytes
starting at the host page behind the first guest page would run off
the end of that 4 KB page into unrelated host memory, corrupting the
kernel, while the guest's later pages stayed empty. So we translate
each guest page through the EPT, as the CPU will, and copy at most one
page at a time.

### Question 2.4

> An EPT entry is "present" if any of R, W or X is set, not if one
> particular bit is set. A common bug is to test only bit 0. Name one
> kind of entry that this bug would treat as missing but that the CPU
> treats as valid (hint: see Appendix A.10 for execute-only support), and
> say why our code never creates such an entry.

**Answer:** An execute-only entry (X = 1, R = W = 0), which is valid on CPUs that
report execute-only support in IA32_VMX_EPT_VPID_CAP bit 0 (Appendix
A.10). A bit-0 test would call it missing: eptwalk() with alloc set
would overwrite it with a new table, losing the mapping, and
eptfree() would skip it and leak what it maps. Our code never creates
one because every entry it writes has R set: upper-level entries get
EPT_RWX and leaf entries get EPT_RWX | EPT_MEMTYPE_WB.

## Part 3: The VMCS, describing the guest and the host

### Question 3.1

> In `adjustctl()`, why is it enough to compute
> `(want | low) & high`? What does the function do in the case where
> `want` has a bit that `high` has cleared, and why is returning an error
> better than silently dropping the bit?

**Answer:** The low half lists bits that must be 1, so OR-ing it in forces them
on; the high half has a 0 for every bit that must be 0, so AND-ing
with it clears those; every other bit keeps the value we asked for, so
the result satisfies both rules (the same idea as CR0/CR4 FIXED0 and
FIXED1 in part 1). If `want` has a bit that `high` clears, the AND
drops it, `(v & want) != want` detects that, and the function prints
the missing bits and the MSR and returns -1, so `vmcreate()` fails.
Silently dropping the bit would start a VM without a feature we rely
on (for example EPT or I/O exiting), and the failure would show up
later as a confusing entry error or misbehaving guest, far from its
cause.

### Question 3.2

> The exception bitmap is all ones. If it were 0, what would happen to a
> page fault in the guest, step by step, given that the guest's IDTR limit
> is 0? Why is the all-ones setting more useful to someone debugging a
> guest?

**Answer:** With a 0 bitmap the page fault is delivered through the guest's IDT.
Vector 14 is beyond the IDTR limit of 0, so delivering it raises a
general-protection fault; delivering that also fails, which becomes a
double fault (vector 8); that fails too, so the CPU has a triple fault,
which in VMX non-root operation is a VM exit with no trace of the
original fault. With all ones, the first exception itself exits, and
the exit reports its vector, error code and guest RIP (and for a page
fault the faulting address), so the person debugging sees the real
cause instead of a bare triple fault.

### Question 3.3

> Why must "enable EPT" be 1 whenever "unrestricted guest" is 1? Think
> about what the CPU does with a guest linear address when `CR0.PG = 0`
> (SDM 28.6).

**Answer:** With unrestricted guest the guest may run with CR0.PG = 0, and then
the CPU uses each guest linear address directly as a guest-physical
address (SDM 28.6), with no page tables in between. Without EPT that
address would go straight to host physical memory, so the guest could
read and write all of the host's RAM. EPT is the only translation left
that the hypervisor controls, so the SDM requires it whenever
unrestricted guest is on (VM entry fails otherwise).

### Question 3.4

> The real `CR4` in the VMCS has `VMXE` = 1 but the guest reads it as 0.
> Which two VMCS fields make that work? What happens if the guest tries to
> set `CR4.VMXE` to 1, and why do we not let it?

**Answer:** The CR4 guest/host mask (with the VMXE bit set, so the host owns
VMXE) and the CR4 read shadow (with VMXE = 0): a guest read of CR4
takes owned bits from the shadow, so it sees 0. If the guest writes
CR4 with VMXE = 1, the value differs from the shadow in an owned bit,
so the write causes a VM exit (a control-register access) instead of
taking effect; our run loop does not emulate it, so runvm stops. We do
not let the guest change it because the real VMXE must stay 1 while
the host is in VMX operation, and we do not support nested
virtualization, so the guest must not believe it can use VMX.

## Part 4: Entering and leaving the guest

### Question 4.1

> `struct guestregs` has no `rsp` field. Why can the guest's stack
> pointer not be handled the way the other registers are? Where does the
> CPU keep it instead, and what would go wrong if `vmx_exit` tried to
> store `%rsp` into `*regs` like the others?

**Answer:** (your answer)

### Question 4.2

> `HOST_RIP` is written once, in `vmcsinit()`, but `HOST_RSP` is written
> in `vmx_enter` on every entry. What makes one constant and the other
> not? Describe a concrete situation in which the `HOST_RSP` written by
> a previous entry would be wrong.

**Answer:** (your answer)

### Question 4.3

> `vmrun()` always calls `vmx_enter(&vm->regs, 0)`, and so always uses
> `VMLAUNCH`. Which instruction clears the launch state, and where does
> `vmrun()` execute it? What would the VM-instruction error be if the
> code passed `launched = 1` on the first entry?

**Answer:** (your answer)

### Question 4.4

> `vmxhoststate()` is called before every entry, not once. Name two
> fields it writes that can differ between two calls to `vmrun()`, and
> say what is different (CPU, process, or both) for each. What would
> happen at the next VM exit if the field kept the old value?

**Answer:** (your answer)

## Part 5: Running the guest, the vmrun loop and exit handling

### Question 5.1

> In Step 8, `vmclear()` must come before `popcli()`. Describe a
> concrete sequence of events, with two CPUs, in which the opposite
> order corrupts the VMCS. Which sentence of SDM 27.11.1 do you rely on?

**Answer:** (your answer)

### Question 5.2

> We pass `launched = 0` to `vmx_enter` on every entry, so we always use
> `VMLAUNCH`. What would the program have to do differently if we kept
> the VMCS loaded between exits and used `VMRESUME` instead? What would
> we give up?

**Answer:** (your answer)

### Question 5.3

> On an external-interrupt exit, `vmxhandle()` does nothing but return 1,
> yet the interrupt is handled by the host before the guest runs again.
> Where and when exactly does the host handle it? What would change if
> the exit control "acknowledge interrupt on exit" were set?

**Answer:** (your answer)

### Question 5.4

> The `WRMSR` exit and the `OUT` exit are both followed by
> `vm->rip += inslen`. What would happen if the code used a fixed
> length (for instance 2) instead? Name one instruction in `helloguestasm.S`
> or `helloguest.c` that shows the problem.

**Answer:** (your answer)

## Part 6: The other half: the monitor (runvm) and the guest

### Question 6.1

> In `doio()`, why does an `IN` need the monitor to write the guest's
> RAX with `vmsetreg()`, instead of `doio()` just returning a value?
> And why does `completein()` read RAX with `vmgetreg()` for a 1-byte
> `IN` but not for a 4-byte one?

**Answer:** (your answer)

### Question 6.2

> `helloguest.ld` links the guest at `0x100000`. Suppose you change it
> to `0x200000` but leave `VM_LOAD_ADDR` alone. Name two things in the
> guest that will go wrong and say why. (Look at the page tables, the
> `gdtdesc`, and where the first instruction is.)

**Answer:** (your answer)

### Question 6.3

> The guest in Step 8 sets `CR4.PAE` before `CR0.PG`. What happens if
> you swap the order of these steps? Then, for the Checkpoint's
> `runvm helloguest 1` experiment: which message do you get, and which
> check in the kernel produces it?

**Answer:** (your answer)

### Question 6.4

> Each `OUT` in the guest causes a VM exit, a trip through the kernel,
> and a return to `runvm`, which calls `write()`. Why did the design
> put the serial port in `runvm` and not in the kernel? What would you
> gain, and what would you lose, by emulating it inside the kernel?

**Answer:** (your answer)
