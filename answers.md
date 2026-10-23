# HW1 answers

Fill in this file and hand it in with your code. Replace each
`(your answer)` with two or three sentences. Keep the question numbers
(part.question), and do not delete the questions.

- **Name:** Giang Ta
- **Machine and environment you tested on** (CADE lab machine with QEMU, or Bochs): my own Fedora Linux laptop (Intel CPU with VT-x, nested KVM), QEMU 10.2.2 with KVM (`make qemu-nox`)
- **How far did you get?** (for example "all six parts work", or "stuck in part 4, block B"): all six parts work, `runvm helloguest` prints the greeting and halts at rip 0x10004E, repeatedly, with different CPUs setup: 1, 2 and 4


## Part 1: Entering VMX operation

### Question 1.1

> In Step 1, why does the function `return` when VMX is missing instead
> of calling `panic()`?

**Answer:** xv6 can run without VT-x, only the hypervisor needs it. Returning leaves `vmx_ok`
at 0, so the kernel still boots and the VM system calls fail cleanly instead
of stopping the whole system.

### Question 1.2

> In Step 2, on most machines the firmware has already enabled and
> locked VMX. Why does the code still handle the unlocked case instead
> of just failing?

**Answer:** Some firmware leaves `IA32_FEATURE_CONTROL` unlocked, so the kernel can still enable VMX outside
SMX and set the lock bit itself. Failing immediately would reject machines
that could support the hypervisor. If firmware has already locked VMX off, the
kernel cannot change that.

### Question 1.3

> In Step 3, what would happen if we skipped `lcr4(cr4)` and went
> straight to `VMXON`?

**Answer:** Changing the local variable does not update CR4 until `lcr4(cr4)` runs. Without it,
`CR4.VMXE` stays 0 and VMXON raises an invalid-opcode exception (#UD), causing a
kernel panic rather than returning an error through the wrapper.

### Question 1.4

> In Step 4, why does each CPU need its own VMXON region? What could go
> wrong if all CPUs shared one?

**Answer:** Each logical processor uses its VMXON region for its own internal VMX state,
so the SDM requires a separate region for each CPU. Sharing one could let CPUs
overwrite each other's state and cause unpredictable behavior.

## Part 2: Creating a VM, its memory, and the EPT

### Question 2.1

> An ordinary x86-64 page table translates virtual addresses to physical
> addresses. What does the EPT translate, and who controls it? What
> would go wrong if the hypervisor gave the guest direct access to the
> host's physical memory instead?

**Answer:** EPT translates guest-physical addresses to host-physical addresses, and the
hypervisor controls those mappings. Giving the guest direct access to host
physical memory would let it read or overwrite the kernel, other processes, or
other VMs.

### Question 2.2

> In Step B7 we zero every page of guest memory, and in `eptwalk()` we
> zero every new table. These are two different reasons to zero. State
> each, and say what the guest or the CPU would see if we skipped it.

**Answer:** We zero guest pages to prevent leaks of old host data and give the guest clean
initial memory. We zero EPT tables so unused entries are marked not present.
Otherwise, junk entries could look valid and send the CPU or `eptwalk()` to bogus
addresses.

### Question 2.3

> In Step B8, why does `vmcreate()` copy the image page by page through
> `eptwalk()`, instead of one `memmove` of `len` bytes to the guest's
> first page?

**Answer:** The guest's memory is contiguous in guest-physical addresses, but its backing
host pages come from separate `kalloc()` calls and may be scattered. One large `memmove()`
could run past the first host page and overwrite unrelated memory. Using `eptwalk()`
for each page finds the correct destination for every part of the image.

### Question 2.4

> An EPT entry is "present" if any of R, W or X is set, not if one
> particular bit is set. A common bug is to test only bit 0. Name one
> kind of entry that this bug would treat as missing but that the CPU
> treats as valid (hint: see Appendix A.10 for execute-only support), and
> say why our code never creates such an entry.

**Answer:** An execute-only entry has X set but R and W clear, and is valid when the CPU
supports execute-only EPT mappings. Testing only bit 0 would incorrectly treat
it as missing. Our code never creates one because both table entries and leaf
mappings use `EPT_RWX`, which includes R.

## Part 3: The VMCS, describing the guest and the host

### Question 3.1

> In `adjustctl()`, why is it enough to compute
> `(want | low) & high`? What does the function do in the case where
> `want` has a bit that `high` has cleared, and why is returning an error
> better than silently dropping the bit?

**Answer:** OR-ing with `low` sets the required bits, and AND-ing with `high` clears the
unsupported bits. If this drops a requested bit, `adjustctl()` detects it and returns
-1. Reporting the error immediately avoids running a VM without a feature the
hypervisor depends on.

### Question 3.2

> The exception bitmap is all ones. If it were 0, what would happen to a
> page fault in the guest, step by step, given that the guest's IDTR limit
> is 0? Why is the all-ones setting more useful to someone debugging a
> guest?

**Answer:** With a zero bitmap, the CPU tries to deliver the page fault through the
guest's IDT, but vector 14 is outside its limit. Delivery raises a general-protection
fault, then a double fault, and finally a triple-fault VM exit
because none of those handlers can be reached. An all-ones bitmap catches the
original exception and reports its vector, error code, and RIP, which makes
debugging much easier.

### Question 3.3

> Why must "enable EPT" be 1 whenever "unrestricted guest" is 1? Think
> about what the CPU does with a guest linear address when `CR0.PG = 0`
> (SDM 28.6).

**Answer:** When `CR0.PG` is 0, a guest linear address is used directly as a guest-physical
address. EPT still translates that address to host memory and limits what the
guest can access. The CPU therefore requires EPT when unrestricted guest is
enabled, and rejects VM entry if it is missing.

### Question 3.4

> The real `CR4` in the VMCS has `VMXE` = 1 but the guest reads it as 0.
> Which two VMCS fields make that work? What happens if the guest tries to
> set `CR4.VMXE` to 1, and why do we not let it?

**Answer:** The CR4 guest/host mask marks VMXE as host-owned, and the CR4 read shadow
supplies the 0 that the guest sees. Trying to set it to 1 causes a control-register-access
VM exit, which our monitor does not emulate. We keep the real
bit set to satisfy VMX requirements while hiding it because we do not support
nested virtualization.

## Part 4: Entering and leaving the guest

### Question 4.1

> `struct guestregs` has no `rsp` field. Why can the guest's stack
> pointer not be handled the way the other registers are? Where does the
> CPU keep it instead, and what would go wrong if `vmx_exit` tried to
> store `%rsp` into `*regs` like the others?

**Answer:** The CPU loads and saves the guest's stack pointer through `GUEST_RSP` in the VMCS. The
assembly needs `%rsp` for the host stack until entry, and on exit the CPU has
already replaced it with `HOST_RSP`. Saving `%rsp` in `vmx_exit` would therefore record the
host's stack pointer instead of the guest's.

### Question 4.2

> `HOST_RIP` is written once, in `vmcsinit()`, but `HOST_RSP` is written
> in `vmx_enter` on every entry. What makes one constant and the other
> not? Describe a concrete situation in which the `HOST_RSP` written by
> a previous entry would be wrong.

**Answer:** `HOST_RIP` always points to the fixed `vmx_exit` label, while `HOST_RSP` points to the stack frame
prepared for this particular entry. If a later entry runs at a different stack
depth, the old value points to the wrong frame. The exit code would then
restore registers and a return address from the wrong stack locations.

### Question 4.3

> `vmrun()` always calls `vmx_enter(&vm->regs, 0)`, and so always uses
> `VMLAUNCH`. Which instruction clears the launch state, and where does
> `vmrun()` execute it? What would the VM-instruction error be if the
> code passed `launched = 1` on the first entry?

**Answer:** VMCLEAR resets the VMCS launch state to clear. `vmrun()` executes it after reading
the exit information and saving guest state, before re-enabling interrupts.
Passing `launched = 1` on the first entry would use VMRESUME on a clear VMCS and fail
with VM-instruction error 5.

### Question 4.4

> `vmxhoststate()` is called before every entry, not once. Name two
> fields it writes that can differ between two calls to `vmrun()`, and
> say what is different (CPU, process, or both) for each. What would
> happen at the next VM exit if the field kept the old value?

**Answer:** `HOST_CR3` depends on the process; a stale value would load the wrong page table at
VM exit and could crash the kernel. `HOST_TR_BASE` depends on the CPU; a stale value
would make the exit load another CPU's TSS base. A later interrupt from user
mode could then switch to the wrong kernel stack and corrupt it.

## Part 5: Running the guest, the vmrun loop and exit handling

### Question 5.1

> In Step 8, `vmclear()` must come before `popcli()`. Describe a
> concrete sequence of events, with two CPUs, in which the opposite
> order corrupts the VMCS. Which sentence of SDM 27.11.1 do you rely on?

**Answer:** If `popcli()` ran first, a timer interrupt could move the process from CPU 0 to CPU 1
before VMCLEAR ran. Clearing on CPU 1 would leave CPU 0's cached VMCS active,
so loading and modifying it on CPU 1 could conflict with CPU 0's stale copy.
SDM 27.11.1 requires VMCLEAR on the CPU where the VMCS is active before
loading it on another CPU, or the VMCS may be corrupted.

### Question 5.2

> We pass `launched = 0` to `vmx_enter` on every entry, so we always use
> `VMLAUNCH`. What would the program have to do differently if we kept
> the VMCS loaded between exits and used `VMRESUME` instead? What would
> we give up?

**Answer:** We would need to track whether the VMCS is launched and which CPU holds it,
using VMRESUME for later entries on that CPU. Before migrating, we would have
to clear it on the old CPU and use VMLAUNCH on the new one. This saves
repeated clears and loads, but gives up the simple rule that the VMCS is
inactive whenever the scheduler can move the process.

### Question 5.3

> On an external-interrupt exit, `vmxhandle()` does nothing but return 1,
> yet the interrupt is handled by the host before the guest runs again.
> Where and when exactly does the host handle it? What would change if
> the exit control "acknowledge interrupt on exit" were set?

**Answer:** Because the exit does not acknowledge the interrupt, it stays pending in the
local APIC. After VMCLEAR, `popcli()` re-enables interrupts and the CPU delivers it
through the host IDT to xv6's normal trap handler before the guest runs again.
If acknowledge-on-exit were set, the hypervisor would need to dispatch the
vector recorded in the exit information and arrange the EOI itself.

### Question 5.4

> The `WRMSR` exit and the `OUT` exit are both followed by
> `vm->rip += inslen`. What would happen if the code used a fixed
> length (for instance 2) instead? Name one instruction in `helloguestasm.S`
> or `helloguest.c` that shows the problem.

**Answer:** A fixed length could resume the guest in the middle of an instruction or skip
an instruction entirely. For example, WRMSR is two bytes, but the guest's `out %al,(%dx)`
is only one byte. Advancing RIP by two after that OUT would put execution in
the middle of the next instruction.

## Part 6: The other half: the monitor (runvm) and the guest

### Question 6.1

> In `doio()`, why does an `IN` need the monitor to write the guest's
> RAX with `vmsetreg()`, instead of `doio()` just returning a value?
> And why does `completein()` read RAX with `vmgetreg()` for a 1-byte
> `IN` but not for a 4-byte one?

**Answer:** The guest registers live in the kernel, so returning a value from `doio()` would
not update them; the monitor must use `vmsetreg()`. A one-byte IN changes only AL, so
`completein()` reads the old RAX to preserve the other bits. A four-byte IN writes EAX
and clears the upper half of RAX, so it does not need the old value.

### Question 6.2

> `helloguest.ld` links the guest at `0x100000`. Suppose you change it
> to `0x200000` but leave `VM_LOAD_ADDR` alone. Name two things in the
> guest that will go wrong and say why. (Look at the page tables, the
> `gdtdesc`, and where the first instruction is.)

**Answer:** The first instruction still runs at `0x100000`, but the guest's absolute addresses
now point 1 MB above where the image was actually loaded. CR3 points to zeroed
memory instead of the real PML4, so instruction fetching faults once paging is
enabled. Also, `lgdt` reads from the wrong address and loads a zero base and
limit into GDTR, leaving the real GDT elsewhere and unavailable for the far
jump.

### Question 6.3

> The guest in Step 8 sets `CR4.PAE` before `CR0.PG`. What happens if
> you swap the order of these steps? Then, for the Checkpoint's
> `runvm helloguest 1` experiment: which message do you get, and which
> check in the kernel produces it?

**Answer:** Setting `CR0.PG` while `EFER.LME` is 1 and `CR4.PAE` is 0 raises a general-protection fault,
which the monitor reports as guest exception 13. `runvm helloguest 1` prints `runvm: vmcreate failed`. The check `VM_LOAD_ADDR + len > memsz`
rejects it because the image starts at 1 MB and cannot fit inside 1 MB of
guest memory.

### Question 6.4

> Each `OUT` in the guest causes a VM exit, a trip through the kernel,
> and a return to `runvm`, which calls `write()`. Why did the design
> put the serial port in `runvm` and not in the kernel? What would you
> gain, and what would you lose, by emulating it inside the kernel?

**Answer:** Keeping serial emulation in `runvm` lets it use normal system calls and keeps
device bugs out of the kernel. Emulating it in the kernel would avoid
returning to user space and calling `vmrun()` again for every character. That would
be faster, but would add device code and output policy to the kernel.
