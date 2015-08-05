# Proposal: handle code relocations more like a traditional linker

Author(s): Michael Hudson-Doyle

Last updated: 2015-08-05

## Abstract

The Go toolchain currently doesn't acknowledge the inherent fiddliness
of linking code on RISC platforms and we should stop trying to wish
the complexity away.

## Background

Architectures with fixed width instructions have the inherent issue
that a 32-bit (or 64-bit!) displacement cannot be stuffed into a
32-bit instruction, so an address needs to be spread across several
instructions, and in turn this requires a sequence of relocations,
each updating a part of an instruction. This leads to relocations that
are inherently processor specific.

For a motivating example, let's consider arm64 and the operation of
loading a value from memory relative to a global symbol.  Consider
this code:

```
package foo

var global struct {
	pad   [32]byte // Just to make the offset non-zero
	value int
}

func access() int {
	return global.value
}
```

The "global.value" access compiles to `MOVD "".global+32(SB), R0`, and
it's what this Prog becomes as machine code (and relocations) that is
interesting.  In Go 1.5 it assembles to:

```
    adrp x27, 0x00000000
    add x27, x27, #0x0
    ldr x0, [x27]
```

And a relocation of type `R_ADDRARM64` that adjusts the first two
instructions to compute the address `global + 32`.  `adrp` computes
the top 21 bits of the address and the low 12 bits are computed by the
`add`.  This is suboptimal: it's possible to do this in two
instructions, an `adrp` and then a `ldr` with immediate offset. But as
the instructions are different need to relocate this sequence slightly
differently: although the immediate happens to be in the same location
within the instruction word, the offset for a load or store
instruction is scaled by the size of the memory access being performed
(i.e. the immediate for loading a 4 byte quantity from x1+100 is
encoded as 25).  This means a linker need to convert the byte offset
to needs to be adjusted before being inserted into the instruction.

## Proposal

This proposal is to switch to a model similar to that used by the
[platform ELF toolchain]
(http://infocenter.arm.com/help/topic/com.arm.doc.ihi0056b/IHI0056B_aaelf64.pdf):
instead of generating one relocation that spans two instructions,
generate a relocation for each instruction that specifies how to
relocate that instruction. So instead of generating an `R_ADDRARM64`
that's only good for computing an address, we'd have relocations that
handle relocating the adrp instruction, the add instruction and also
each size of load and store.  The ELF processor supplement calls
these:

```
    R_AARCH64_ADR_PREL_PG_HI21
    R_AARCH64_ADD_ABS_LO12_NC
    R_AARCH64_LDST8_ABS_LO12_NC
    R_AARCH64_LDST16_ABS_LO12_NC
    R_AARCH64_LDST32_ABS_LO12_NC
    R_AARCH64_LDST64_ABS_LO12_NC
```

which look a bit intimidating at first but are not hard to read with practice:

  R_AARCH_{instruction kind}_{absolute/pc relative}_{which bits to extract}

The proposal is to use the same names (as far as I can tell, other
binary formats use similar names).  There is no proposal to make any
effort to make the values match those used in the processor
supplement.

A branch that implements the above change (accessing memory relative
to a symbol in two rather than three instructions) is available at:

    https://github.com/mwhudson/go/tree/risc-relocs-example

It passes ./all.bash and makes the .text section of godoc about 77k
smaller.

Taking a similar approach will make implementing other changes
(external linking on ppc64le, shared library support for RISC
platforms, straightening out the thread local storage implementation
on ARM and other platforms) more straightforward.

This proposal is to add support for more relocations strictly
as-needed: there's no need to support the full range of relocations.

## Rationale

There are obviously alternatives to any one change that might use this
scheme. For the example above, the obvious alternatives would be

 1. add a extra field to Reloc to identify somehow the second
    instruction in the sequence (I think this is roughly how cgo
    internal linking works on ppc64x today -- there is a Variant field
    on the linker's Reloc struct), or

 2. have the linker disassemble the second instruction
    to work out how to relocate it.

The first approach just seems slightly arbitrary to me, there doesn't
seem to be any value in specifying a relocation with two fields (type,
variant) rather than just type.  The second seems needlessly complex
and error prone.

The disadvantage of doing things the way proposed here is that it
either requires a certain amount of very repetitive code or intricate
table-driven code. For example, this is an implementaton of the
load/store relocations for arm64:

```
	case obj.R_AARCH64_LDST8_ABS_LO12_NC:
		t := ld.Symaddr(r.Sym) + r.Add
		*val |= (t & 0xfff) << 10
		return 0

	case obj.R_AARCH64_LDST16_ABS_LO12_NC:
		t := ld.Symaddr(r.Sym) + r.Add
		if t&0x1 != 0 {
			ld.Diag("offset for 16-byte load/store has unaligned value %d", t)
		}
		*val |= (t & 0xfff) << 9
		return 0

	case obj.R_AARCH64_LDST32_ABS_LO12_NC:
		t := ld.Symaddr(r.Sym) + r.Add
		if t&0x3 != 0 {
			ld.Diag("offset for 32-byte load/store has unaligned value %d", t)
		}
		*val |= (t & 0xfff) << 8
		return 0

	case obj.R_AARCH64_LDST64_ABS_LO12_NC:
		t := ld.Symaddr(r.Sym) + r.Add
		if t&0x7 != 0 {
			ld.Diag("offset for 64-byte load/store has unaligned value %d", t)
		}
		*val |= (t & 0xfff) << 7
		return 0
```

For all the repetition, I favor this approach: it's simple in some
sense, and it's easy to verify the correctness of each relocation
(with a copy of the architecture manual open).

The proposal use the ELF names for the relocations because they are
usually *reasonably* well documented.

## Compatibility

This is a toolchain only change, so there are no language-level
compatiblity concerns. There are slight concerns about interpreting
intermediate .a files: the values of the r.Type field will become more
volatile and version dependent. This doesn't seem a major problem to
me (they are already volatile and version depedendent, just not as
much as they are likely to be with this proposal).

## Implementation

This mostly a proposal for a shift in mindset and how a certain kind
of work *should be done*, rather than for a specific piece of work, so
in that sense there is no implementation part.  I (Michael
Hudson-Doyle) want to do several things in the 1.6 cycle that would
use this style, mostly related to extending shared library support to
other platforms. In fact I've already hacked up implementations of
shared library support for arm64, armhf and to some extent ppc64le, so
I know it works and isn't that much effort.
