# CLAUDE.md

Context library: reusable, topic-organized Markdown fed to LLMs. See [README.md](README.md) for layout and usage.

## Documents are self-contained

Every document must stand completely on its own. **Do not link or refer to other documents in this repository** (no relative-path links, no "see X.md", no "developed further in Y", no "See also" sections pointing at sibling files). A reader must never have to open a second file to understand the one they're in.

When a document needs a concept that another document also covers, inline what this document needs: state the definition, idea, or example directly, in as much depth as this document's argument requires. Some overlap between documents is expected and fine; that redundancy is the price of self-containment, and it's a price we pay deliberately.

This applies to the body and to any "See also" / "Further reading" / footer sections, which should not exist. References to genuinely external material (papers, specs, source repos, URLs) are allowed and encouraged; the rule is only about cross-references within this repository.

## Writing style

Prose follows [writing/direct_prose.md](writing/direct_prose.md): no em dashes, no AI tells (empty significance claims, filler intensifiers, hollow transitions), specific and concrete, facts stated plainly and judgments appropriately hedged. (That file is itself a context document; reading it here is fine, the no-cross-reference rule governs what the documents say, not how you work on them.)
