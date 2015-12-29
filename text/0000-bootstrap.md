# An RFC process for intermezzOS

Every design choice in intermezzOS must be justified through writing an RFC,
before it’s accepted that this is the path that we’ll take.

“Design choice” is a bit vague, but it’s basically any high-level choice. This
isn’t “every PR must be justified” but any sort of “Why is intermezzOS like
$x?” question should probably have an answer here.

## Details

To initially discuss a topic, open an issue. We’ll discuss things in those
issues.

When a proposal is ready to be made, submit a pull request adding a new file in
the `text` directory. We’ll discuss things in the PR, and then either merge or
not.

## Drawbacks

This is a bit more heavyweight than just doing whatever I feel like doing.

## Alternatives

Not have an RFC process. This is a pretty valid alternative, given that I’m the
only one working on this project for now. However:

1. I don’t have a good place to collect research on various topics as I make my
   decisions.
2. It might be easier to not have one, but I’ve found RFCs so useful in Rust that
   I am willing to bet that I wish I had done this previously.
3. RFCs allow for newcomers to learn about what design decisions have been made, and why. 
