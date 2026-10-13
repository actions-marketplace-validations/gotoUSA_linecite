# How linecite works

This page is for contributors. Its citations are checked by linecite itself on every commit, so the
line numbers below are right for the version you are reading.

## One reader

Every command reads documents through one function, [`scan.py:313`](../src/linecite/scan.py#L313 "extract: def extract("). It finds anchors first —
everywhere, including code blocks, since an anchor in a code excerpt is a real citation — then links
and symbol references outside code, then number-only citations outside code. Regions between ignore
markers are blanked before any of that ([`scan.py:153`](../src/linecite/scan.py#L153 "ignore_regions: def ignore_regions(")); an unclosed marker ignores nothing and is
reported.

## Resolving a citation

[`spec.py:171`](../src/linecite/spec.py#L171 "resolve: def resolve(") splits a spec into a file suffix, a symbol and quotes. The suffix must match exactly one
tracked file; the symbol's span comes from Python's `ast` ([`repo.py:239`](../src/linecite/repo.py#L239 "Repo.python_symbols: def python_symbols(")) or from YAML indentation;
the quote must hit exactly one line of that span ([`spec.py:49`](../src/linecite/spec.py#L49 "Target.find: def find(")) unless `#n` picks a hit. `sync` then
rewrites the numbers, writing bytes so the file keeps its own line endings ([`scan.py:431`](../src/linecite/scan.py#L431 "scan_file: path.write_bytes(")).

## What changed since a revision

`affected` reads `git diff -U0` ([`affected.py:60`](../src/linecite/affected.py#L60 "changed_lines: def changed_lines(")) and flags a citation when a changed line falls
anywhere inside the cited symbol, not just on the quoted line ([`affected.py:124`](../src/linecite/affected.py#L124 "affected: a, b = target.span")): a change next to a
line can change what the prose says about it.

## Auditing number-only citations

A number like `orders.py:310`<!--@--> records nothing about its code, so `audit` asks git when the
doc line was written ([`history.py:57`](../src/linecite/history.py#L57 "DocHistory._blame: def _blame(")) and which code it described ([`audit.py:115`](../src/linecite/audit.py#L115 "Tracer.origin: def origin(")): the same commit
when docs and code share a repository, or the code's first-parent commit at that time when they do
not ([`repo.py:212`](../src/linecite/repo.py#L212 "Repo.commit_before: out = self.git(\"log\", \"--first-parent\", \"--format=%ct %H\", \"HEAD\")")). A diff from that code to today's ([`history.py:89`](../src/linecite/history.py#L89 "LineMap: class LineMap:")) says where the line went. A
rewritten line counts as moved only when the diff dropped exactly one copy of its text and added
exactly one ([`audit.py:305`](../src/linecite/audit.py#L305 "Tracer.trace_line: moved = hits[0] if len(hits) == 1 and len(lost) == 1 else None")).

`adopt` offers a conversion only after reading the document back with the replacement in place, as
`check` will ([`adopt.py:102`](../src/linecite/adopt.py#L102 "_check: def _check(")).

## Read-only git

Every git process runs with [`repo.py:19`](../src/linecite/repo.py#L19 "GIT_ENV: GIT_ENV = {**os.environ, \"GIT_OPTIONAL_LOCKS\": \"0\", \"GIT_NO_LAZY_FETCH\": \"1\"}"): no opportunistic index refresh, and no lazy fetches in a
partial clone.
