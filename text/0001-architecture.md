# Architecture

intermezzOS will support `x86_64` only.

## Details

There’s a basic tradeoff for any hobby kernel: write it for ‘the real world’
of computers you probably own, or write it in something that’s easier.
Unfortunately, `x86` as a family is full of all kinds of odd legacy bullshit,
and newer architectures are much nicer.

However, it’s magical to actually boot up your ‘real’ computer with an OS you
made. To me, this is more important than just being ‘easier’. We can learn
the hard stuff!

Finally, I have significantly more experience on `x86_64` than anything else.

## Drawbacks

Other architectures can be a lot easier, especially with the 64-bit variant of
`x86`.

I think that this drawback isn’t that big of a deal, and that the advantages
are larger.

## Alternatives

There are two broad categories here: pick an initial different architecture, or
support multiple architectures.

### Different?

I could pick something else. With the `x86` family, sticking to 32 bit would
be similar, but easier. I could also choose ARM, MIPS, or something else.

### Multiple

This is still an option in the future, and is easier to do then rather than
now. It’s possible that I might make a choice which makes porting difficult;
that should be a consideration for future RFCs.
