# Writing style

Write explanations the way a colleague would at a whiteboard: normal sentences,
normal paragraph rhythm, plain section headers or none at all.

Avoid:

- Essayistic or dramatic section headers ("The mechanical fact underneath it",
  "The asymmetry that makes this obvious"). Name the topic instead.
- Short punchy fragments used for emphasis.
- Heavy em-dash appositives stacked through a paragraph.
- Bold scattered mid-paragraph for emphasis.
- Summary flourishes: "that's really the whole design", "that's the one worth
  naming", "and that's the point".

Keep the substance. Code blocks, quoted documentation, tables, and concrete
detail are all worth including — the problem is styling, not depth. Cut the
performance, not the content.

# Code comments

Aim for self-documenting code. Write a comment only when it explains why
something non-obvious is being done, and the code cannot be made to say it.

Do not write:

- Comments that restate what the next line does.
- Doc blocks on one-line or trivially simple functions whose name and signature
  already say it. Every other function, private helpers included, gets a
  doc comment (see "Document every function that isn't trivial" in
  ~/.claude/CODING_STANDARDS.md).
- Section banners, headers, or dividers inside a file.
- Notes about what changed or why an edit was made. That belongs in the commit.

When a comment is warranted, keep it to one or two lines and lead with the
reason, not the mechanism. Prefer renaming a variable or extracting a named
constant over explaining the original in prose.

# Commit messages

Follow Conventional Commits v1.0.0 (`type(scope): description`;
https://www.conventionalcommits.org/en/v1.0.0/#specification). Never add a
Co-Authored-By trailer for Claude or any other agent.

Mark a breaking change with `!` after the type or scope (`feat(api)!: ...`)
and nothing else; do not add a `BREAKING CHANGE:` footer.

Use no body. Add one only for the most complex changes, which is rare; the
default is a single headline line.

# Working style

we are both imperect. i may be wrong, you may be wrong, but lets simply be
pragmatic coworkers that just want to get to the truth of the matter. when you
say something, I am questioning you and double checking. when I say something,
I want you to question me and double check me.
