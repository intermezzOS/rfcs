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
owner of the video memory. I have not yet found a working way to magically
assure Rust that the magic raw memory is the type we want -- I remember building
this out of `core::ptr::from_raw_parts_mut()` in the workshop but have not been
able to get it to work properly with any type other than `[u8; 4000]`.
Regardless of *how* a VGA-memory-owner gets its memory, though, we will have to
do magic to assure Rust that the raw memory at the VGA magic address is able to
be used within the type system.

I propose designing a VGA structure which owns only a single VGA-memory-sized
slice, rather than the current implementation which gives one VGA structure two
regions of memory. Since the magic address is the kind of implementation detail
which should be hidden from consumers, the VGA structure will implement multiple
constructors: the standard `new()` method will take a generic buffer, which can
be allocated anywhere in normal memory (but as intermezzOS does not yet have a
dynamic allocator, it will probably be in BSS), and an **unsafe** `new_at_vga()`
method will create a VGA structure that has taken ownership of the video memory
region.

In order to achive the pseudo-atomic functionality currently present in our
console implementation, the VGA structure will need to implement a method that
will permit one structure to borrow another and copy the borrowed buffer into
its own storage. When the VGA structure at video memory does this, we achieve
pseudo-atomic video update, and we will also be able to rapidly change display
contexts by switching between multiple different VGA structures in main memory.

Incidentally, since VGA-memory structures are 4000-byte slabs of memory with no
other special considerations, we can implement Clone and Copy for the atomic
units (each cell of the grid), and thus for each row and for the entire grid.
This means that such a borrow-for-copy method would simply look something like
this:

```rust
impl VgaMemory {
    //  ...
    fn copy_from_other(&mut self, other: &VgaMemory) {
        //  do whatever magic needs be done to access the actual buffer
        self.buffer = other.buffer;
        //  assignment uses Copy trait to memcpy 4000 bytes from other to self
    }
}
```

No loops or bounds checking need be done by us; everything can be verified at
compile-time thanks to Rust's intrinsic type- and bounds- checking. Valid
`VgaMemory` (or however this gets implemented) instances will be guaranteed to
own memory of the appropriate size and type, so this doesn't even need to use
unsafe code (at this level) in any way.

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

## Potential Pain Points

### How do we point at VGA memory?

Speaking entirely from personal experience, the Unique pointer mechanism is a
bit of an educational ride. When I first saw it, I assumed from the name that
Rust would verify at compile time that only one Unique pointer to a memory hunk
existed at a time. It turns out this doesn't happen; Unique is paired with a
Shared pointer type and simply means "this pointer has mutable access" while a
Shared pointer type will not allow mutable access. See its [documentation][docu]
and [issue][ghu]. Unique should really be named Owned, because it signifies that
it has full ownership of the memory to which it points, and a Mutex would be
used at runtime to guarantee synchronized access. IntermezzOS already has a
Mutex in use in the current kernel source, but this is not discussed in the book
yet.

Constructing a pointer from raw parts is also an option, but in my personal
experience and opinion much more prone to bugs. Whether we use Unique (an
unstable, slightly misleading, and not-yet-standardized feature) or construction
from raw parts (a manual, difficult, means of getting the compiler to whine at
us) is a question worth considering regardless of how else we structure the VGA
design.

### How do we handle safety and testing?

Giving Rust access to the magic memory address is always going to be (a) unsafe
and (b) impossible to test except in production.

- unsafe: No matter what method we choose for doing so, we will wind up having
to convince Rust to just give use the VGA magic memory, and damn the rules. This
is, obviously, going to require `unsafe` functions. In my experimental
implementations, I have elected to use unsafe functions, rather than unsafe
blocks inside functions, for the instantiation at magic memory, like so:

```rust
impl VgaScreen {
    unsafe fn new_at_vga() -> VgaScreen {
        VgaScreen {
            //  however we trick Rust into giving us 4000 bytes at 0xB800;
        }
    }
}
impl VgaWriter {
    unsafe fn new_at_vga() -> VgaWriter {
        VgaWriter {
            //  other fields omitted...
            screen: VgaScreen::new_at_vga(),
        }
    }
}

//  Safe -- uses non-special memory allocated by the compiler and linker
let some_writer: VgaWriter = VgaWriter::new();
//  Unsafe -- uses the magic memory location
let main_writer: VgaWriter = unsafe { VgaWriter::new_at_vga() };
```

This provides a clear visual cue when instantiating an owner of magic memory,
that they are using magic memory instead of general memory and that doing so can
come back to haunt them if they're not careful, such as by making two writers on
magic memory. Requiring explicit acknowledgement of unsafety by the consumer is,
in my opinion, both educational and desirable.

- segfaulting: A real operating system will never grant access to physical
memory at the VGA address, and the test suite can't guarantee that it will own
enough virtual memory for that address to function as a virtual address, or even
if that address is in scope, that it won't belong to some other part.

So `cargo test` can't test any code using `new_at_vga()`, because it will always
segfault. The only way to test that the implementation runs properly is by
ensuring `new_at_vga()` functions are the ONLY functions touching magic memory,
and that the structures we design function identically regardless of where it is
they own a buffer. Testing against `new()` and all high-level uses of the VGA
structs must work correctly, and then using `new_at_vga()` in the compiled VM
should successfully manipulate video memory.

This will, I hope, wind up as an encouragement of automated testing protocols.
There are only two ways to give confident assurances that literally-untestable
code will function correctly, and one of them requires a lot of formal math.
The other is using good design and automated tests wherever else possible.
IntermezzOS already encourages and uses such a plan, so this should be somewhat
painless.

## How do we clearly explain what's going on?

Rust has a great tradition of documentation already; once a settled design is
chosen, clean documentation and a chapter in the book should suffice just fine.

[docu]: https://doc.rust-lang.org/core/ptr/struct.Unique.html
[ghu]: https://github.com/rust-lang/rust/issues/27730
