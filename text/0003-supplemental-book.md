# Computer Architecture Book

Operating Systems Development requires knowledge of low level computer concepts
such as how CPUs, RAM, and IO devices work. The intermezzOS book touches on
these concepts as needed and encourages and motivates the reader by teaching
these concepts only as they are needed and without flooding the student with
too much theoretical or tangential information. The book reassures the student
that all the information they need to follow along is contained within the book,
but that, of course, learning more will only aid in their understanding. But if
the reader wants to learn more, where should they go?

This is a proposal for a supplemental book on the topics of computers at the
lowest level. The book aims to teach the student the wide range of topics
associated with computer architecture that are essential for being able to
build operating systems but are also tangential to actually doing so.

These topics include:
  * Assembly Programming
  * Von Neumann Architecture
  * CPU Architecture and ISA
  * Stack and Heap memory
  * Virtual Memory
  * etc.

The book should not be thought of as a "learn to program assembly language"
guide although assembly will be used as the vehicle for introducing new topics
and "getting the reader's hands dirty".

## Drawbacks

The IntermezzOS project already has one book. Adding another book not only adds
more responsibility for the community to handle but also adds the challenge of
figuring out how certain topics should be covered in each of the books.

For instance, virtual memory is both fundamental to understanding how computers
work and operating systems development. Where does the coverage of this topic end
in one book and begin in the other?

## Alternatives

Of course, we can choose not write another book. Any topics that would have gone
into this book are instead covered in appendix chapters, covered as asides, addressed
through references to external blog posts or left out entirely. Of course, those
who are thirsty for more will have to quench their thirst elsewhere.
