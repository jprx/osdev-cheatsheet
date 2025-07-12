# OSDev-Cheatsheet

A quick reference for the `X86_64`, `AARCH64`, and `RISCV64` ISAs.

## Registers

|                  | RISCV64      | AARCH64   | X86_64                                           |
|------------------|--------------|-----------|--------------------------------------------------|
| General Purpose  | X1-X31       | X0-X30    | RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP, R8-R15   |
| Program Counter  | PC           | PC        | RIP                                              |
| Return Address   | RA (X1)      | LR (X30)  | On Stack                                         |
| Stack Pointer    | SP (X2)      | SP        | RSP                                              |
| Frame Pointer    | FP (X8/ S0)  | FP (X29)  | RBP                                              |
| Zero Register    | ZERO (X0)    | XZR / WZR | --                                               |
| Segmentation     | --           | --        | CS, SS, DS, ES, FS, GS                           |

## Calling Conventions

|                    | RISCV64               | AARCH64     | X86_64                          |
|--------------------|-----------------------|-------------|---------------------------------|
| Arguments          | A0-A7 (X10-X17)       | X0-X7       | RDI, RSI, RDX, RCX, R8, R9      |
| Return Value       | A0 (X10)              | X0          | RAX                             |
| Callee-Saved       | S0-S11, SP (X2)       | X19-X30, SP | RBX, RBP, R12-R15, RSP          |
| Caller-Saved       | RA (X1), A0-A7, T0-T6 | X0-X18      | RAX, RCX, RDX, RSI, RDI, R8-R11 |

## Processor Modes

|                         | RISCV64 | AARCH64 | X86_64 |
|-------------------------|---------|---------|--------|
| User Mode               | U-Mode  | EL0     | Ring 3 |
| Kernel Mode             | S-Mode  | EL1     | Ring 0 |
| Hypervisor Mode         | --      | EL2     | --     |
| Machine / Firmware Mode | M-Mode  | EL3     | --     |

## Interrupt Mode

|                     | RISCV64          | AARCH64            | X86_64                         |
|---------------------|------------------|--------------------|--------------------------------|
| Interrupts Enabled? | sstatus.sie, sie | DAIF (I,F)         | RFLAGS.IF                      |
| Disable Interrupts  | csrw sie, zero   | msr daifset, 0xf   | cli                            |
| Restore Interrupts  | restore sie      | restore daif       | sti, if IF was set previously  |

## Exceptions / Interrupts / Syscalls (AKA "traps")

|                        | RISCV64          | AARCH64        | X86_64 (Exceptions / Interrupts)  | X86_64 (Syscalls)     |
|------------------------|------------------|----------------|-----------------------------------|-----------------------|
| Exception Handler Base | stvec            | vbar_el1       | Interrupt Descriptor Table (IDT)  | LSTAR                 |
| Saved PC               | sepc             | elr_el1        | On stack (RIP)                    | RCX                   |
| Saved Privilege Level  | sstatus.spp      | spsr_el1.M     | On stack (CS)                     | STAR (assumed ring 3) |
| Saved Interrupt Mode   | sstatus.spie     | spsr_el1.{I,F} | On stack (RFLAGS.IF)              | R11 (saved RFLAGS.IF) |
| Exception Cause        | scause           | esr_el1        | On stack (sometimes)              | --                    |
| Saved SP               | --               | --             | On stack (SS:RSP)                 | --                    |
| New SP                 | --               | --             | tss.rsp0 (only on cpl 3 -> 0)     | --                    |
| Scratch Register       | sscratch         | --             | --                                | gs                    |

## Privilege Transitions

### X86_64

On `syscall`:
 1. `rcx`    <- old `rip`
 1. `r11`    <- old `rflags`
 1. `rip`    <- `lstar`
 1. `rflags` <- `rflags` & `~sfmask`
 1. `cs`     <- `star.syscall_cs`
 1. `ss`     <- `star.syscall_cs + 8`

On `sysret` (64-bit):
 1. `rip`    <- `rcx`
 1. `rflags` <- `r11`
 1. `cs`     <- `star.sysret_cs + 16`
 1. `ss`     <- `star.sysret_cs + 8`

On interrupt/ exception:
 1. `rsp`    <- `tss.rsp0`, only if we were in ring 3 before (otherwise `rsp` unchanged)
 1. push `ss`
 1. push `rsp`
 1. push `rflags`
 1. push `cs`
 1. push `rip`
 1. (for some exceptions) push an error code
 1. `rip`    <- `idt[idx].offset`
 1. `cs`     <- `idt[idx].selector`

On `iret` (64-bit):
 1. pop `rip`
 1. pop `cs`
 1. pop `rflags`
 1. pop `rsp`
 1. pop `ss`

### AARCH64

On system call (`svc`) / interrupt/ exception:
 1. `elr_el1`    <- old `pc`
 1. `esr_el1`    <- trap reason
 1. `spsr_el1`   <- old `pstate`
 1. `pc`         <- `vbar_el1 + offset`, where `offset` is dependent on what kind of trap this is

On `eret`:
 1. `pc`         <- `elr_el1`
 1. `pstate`     <- `spsr_el1` (sets privilege level (EL0 / EL1), interrupt mode (DAIF), among other things)

### RISCV64

On system call (`ecall`) / interrupt/ exception:
 1. `sepc`           <- old `pc`
 1. `sstatus.spp`    <- old privilege mode
 1. `sstatus.spie`   <- old `sstatus.sie` (interrupt enable for supervisor mode)
 1. `sstatus.sie`    <- 0
 1. `scause`         <- trap reason
 1. `stval`          <- (for some exceptions) extra information about the trap
 1. privilege mode   <- supervisor (`0b01`)
 1. `pc`             <- `stvec`

On `sret`:
 1. `pc`             <- `sepc`
 1. privilege mode   <- `sstatus.spp`
 1. `sstatus.sie`    <- `sstatus.spie`

Note that `sstatus.sie` is ignored in U mode and is treated as always 1.

## Virtual Memory

|                    | RISCV64    | AARCH64              | X86_64 |
|--------------------|------------|----------------------|--------|
| Page Table Pointer | satp       | ttbr0_el1, ttbr1_el1 | cr3    |
| Paging Enabled     | satp.mode  | sctlr_el1.mmu        | cr0.pg |

## System Instructions

|                    | RISCV64               | AARCH64            | X86_64       |
|--------------------|-----------------------|--------------------|--------------|
| System Call        | ecall                 | svc #X             | syscall      |
| Trap Return        | sret                  | eret               | iret, sysret |
| Flush TLB          | sfence.vma zero, zero | tlbi vmalle1       | write to CR3 |

# Notes

- The 32 bit forms of registers refer to the lower 32 bits of their 64 bit counterparts.
  For example, EAX refers to the lower 32 bits of RAX.
- AARCH64 has multiple stack pointers, one for each exception level.
  Which one is being used is controlled by `SPSel`.
- All RISCV64 registers have multiple names. All registers have a number (eg. X0-X31),
  but can also be referred to by a human-readable name (eg. A0 means argument 0, and is the same as X10).
- We are using the UNIX-style System-V ABI for X86_64, not the Microsoft x64 convention.
- sstatus.spie is the saved value of sstatus.sie, which only controls interrupt delivery in S mode.
To control interrupt delivery in U mode, use the sie csr, which is not saved / restored during traps.

# References
- [RISC-V Calling Convention Chapter 18](https://riscv.org/wp-content/uploads/2024/12/riscv-calling.pdf)
- [System V Application Binary Interface AMD64 0.99.6](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
- [AARCH64 Procedure Call Standard](https://developer.arm.com/documentation/102374/0102/Procedure-Call-Standard)

