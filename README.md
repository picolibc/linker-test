# Getting lld to work with Picolibc

lld _almost_ works with picolibc, but the TLS section of the sample
picolibc linker script doesn't appear to generate the desired output.

This little project demonstrates the issue without depending on
picolibc at all, only clang and the two linkers.

## The test

The test code is a subset of the tls test case from picolibc and
includes six global variables and the required _start entry point.

	int data_var = 1;
	__thread int tdata_var = 2;
	_Alignas(128) __thread int tdata_align_var = 3;
	__thread int tbss_var;
	_Alignas(256) __thread int tbss_align_var;
	int bss_var;

The default picolibc linker script should arrange these packed
together as closely as possible in memory with extra spacing as
required to align the two over-aligned variables.

## The GNU linker

Running this with the GNU linker (ld), we see that the assigned
addresses for each of the variables makes sense:

	20000000 D data_var
	20000100 D __tls_base
	00000000 D tdata_var		0x20000100
	00000080 D tdata_align_var	0x20000180
	00000100 B tbss_var		0x20000200
	00000200 B tbss_align_var	0x20000300
	20000304 B bss_var

The addresses in the 'static' TLS segment are included in the right
hand colume for TLS variables and are the symbol value (which is the
TLS offset) added to the __tls_base value, which is the
linker-allocated static TLS block.

## LLD

Here's the same part of the symbol table from LLD:

	20000000 D data_var
	20000080 D __tls_base
	00000000 D tdata_var		0x20000080
	00000080 D tdata_align_var	0x20000100
	00000180 B tbss_var		0x20000200
	00000280 B tbss_align_var	0x20000300
	20000408 B bss_var

Running this with LLD, we see two problems:

1. `tbss_align_var` isn't assigned an offset which is correctly
    aligned -- the static TLS region allocated by the linker works,
    but any dynamically created TLS region (which would be allocated
    with 256-byte alignment) ends up with `tbss_align_var` misaligned

 2. `bss_var` 0x104 bytes higher in the address space, indicating that
    lld overallocated the TLS area (presumably due to the presence of
    the .tbss_space section).


Problem 2. is survivable, and would likely be fixed by eliding the
.tbss_space section. I'm not sure how to fix 1. though -- that would
make any dynamically allocated TLS section incorrectly aligned.
