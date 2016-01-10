# Licensing

intermezzOS will use “open source” licensing, rather than “free software”
licensing.

Specifically, the book will be under the CC0, and the code will be under
MIT/Apache 2.0.

## Details

There are two major forces: open source and free software. There are details
between the two, but the biggest comes from what ‘freedom’ means. The crux of
the choice is this:

> Should someone be allowed to create a closed-source version of intermezzOS?

Given the goals of intermezzOS, primarily, that learning resources should be
available as widely as possible, I believe the answer to this question is ‘yes’.

This is a slightly counter-intuitive conclusion. Wouldn’t forcing it to always
be open mean that things would be more open? In a certain sense, this might be
true, but it would restrict what students can do with this material, and that
makes me uneasy. Furthermore, the boot code is derived from @phil-opp’s, and
that’s currently licensed under MIT/Apache 2.0.

I personally tend to prefer free software, but am not religious about it. I
think that this project is significantly different from the projects that
make me prefer it, and so it’s worth using the more liberal license.

Furthermore, as a Rust project, most of the Rust world is using MIT/Apache 2.0.
Continuing in that tradition makes sense, and makes it easier to use IP across
the ecosystem.

## Drawbacks

If someone were to create a proprietary intermezzOS and make piles of cash off
of it, I would be sad.

## Alternatives

We could make the opposite choice and choose the GPL + GFDL. This would require
relicensing the code, as well as licensing the book. And it may drive off some
potential contributors.

We could make everything still be “all rights reserved”, as it is now. This makes
some things very murky; is a fork unacceptable then? It’s also a bit inconsistent
to have open source code, but not an open source book, though people do do it.
