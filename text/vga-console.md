# VGA Console

The VGA spec and the guaranteed presence of VGA hardware on any machine or VM
intermezzOS targets provide convenient methods of printing information for us to
observe the kernel in action.

## Details

The VGA spec says that the magic address `0xB8000` is the start of video memory,
and represents the top left cell on the screen. The screen is 80 columns wide,
25 rows high, and each column is two bytes. In total, VGA video memory is 4000
bytes (just under 4K).

### Overview

At present, intermezzOS' VGA implementation consists of a struct that owns a
VGA-sized buffer **not** at the magic location, the VGA memory that is at the
magic location, and some minor internal state.

The console structure uses the buffer to collect text until `flush()` is called,
at which point the buffer is copied en masse to the video memory. This provides
apparently-atomic updates to the screen, although this operation need not be
*actually* atomic as (a) good design and Rust checks should guarantee only one
VGA structure owns video memory, and (b) moving 4k of memory is a long time to
not have any other interrupts.

### Proposed Design

Philipp Oppermann suggests using a Unique pointer to wrap the magic memory
location so that the compiler can guarantee that there is ever only one mutable
owner of the video memory. This is a good plan, but in practice may be overkill
for intermezzOS' needs and goals. Unique is yet another feature of Rust nightly
that needs to be set as a crate-level attribute, and while intermezzOS already
makes plentiful use of such features, I feel that where some can be trimmed, we
may want to do so.

I propose designing a VGA structure which owns only a single VGA-memory-sized
slice, rather than the current implementation which gives one VGA structure two
regions of memory. Since the magic address is the kind of implementation detail
which should be hidden from consumers, the VGA structure will implement multiple
constructors: the standard `new()` method will take a generic buffer, which can
be allocated anywhere in normal memory (but as intermezzOS does not yet have a
dynamic allocator, it will probably be in BSS), and an `at_vga_memory()` method
will create a VGA structure that has taken ownership of the video memory region.

In order to achive the pseudo-atomic functionality currently present in our
console implementation, the VGA structure will need to implement a method that
will permit one structure to borrow another and copy the borrowed buffer into
its own storage. When the VGA structure at video memory does this, we achieve
pseudo-atomic video update, and we will also be able to rapidly change display
contexts by switching between multiple different VGA structures in main memory.

### Other Considerations

From the intermezzOS workshop and from personal experience, I noticed that
forgetting some details of the VGA buffer's size was a common source of bugs.
Mixing up rows vs columns, or forgetting that each cell was two bytes wide not
one, leads to some interesting learning opportunities in debugging.

Some of the problems with C that Rust aims to overcome are type safety and
buffer overruns. Working with video memory provides opporunities to showcase
Rust's power in both regards.

While we obviously can continue to treat VGA memory as a single chunk of 4000
`u8`s, I feel that doing so offers no benefit to writing intermezzOS in C. Mr.
Oppermann's tutorial defines a character struct two bytes wide that internally
handles ordering (little- vs big- endianness is a frequent bug) and sizing, so
the screen can be represented in terms of these two-byte character structs
rather than individual bytes, eliminating the need to remember to multiply the
memory size by two.

Furthermore, while memory itself is clearly one continuous stream of cells,
representing a 2D grid on that requires some mathematics. At present, the VGA
structure requires that the programmer remember to do division and modulo work
themselves to map between cells on screen and memory locations. However, if the
VGA structure were to represent video memory as slices of slices of cell
structs, then Rust could provide firm guarantees about not overflowing
individual rows or the buffer as a whole.

(A benefit to this approach I noted was that my tests for wrapping long lines
would fail when trying to overrun a single row, rather than letting me write
into the next row without taking appropriate action).

## Advantages

- Simplify the design of the VGA structure and implementation
- Isolate magic numbers such as the video memory address
- Use Rust type-checking to make screen cells, not bytes, the atomic unit of the
console
- Use Rust bounds-checking to ensure that the console is correctly manipulated,
both in entirety and also for individual rows
- Simplify future development plans, such as rapidly switching the display
between multiple display-sized buffers (multiple consoles, full-screen contexts)
- Demonstrate the power of Rust abstractions over hardware, without the cost
such abstractions usually impose
- More thorough unit testing

## Disadvantages

- Would require a nearly full rewrite of the VGA console
- Increased complexity causes confusion about different things
    - at present, confusion stems from the magic numbers and byte ordering
    - with this, confusion would stem from the complex structure and design
- Performance penalty? Memcpy is memcpy, regardless of ownership, but maybe
decoupling might cause some extra overhead
- More complex unit/integration testing
