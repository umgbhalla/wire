# Scope

Specification only until the user explicitly authorizes implementation. Do not
create Rust/Python source, Cargo manifests, migrations, runnable examples, database
adapters or binding scaffolds.

Design language-neutral algebraic types and laws for storage, activity, state
control, history reconstruction and compatibility. Rust and PyO3 are future targets.
Codex, Claude Code and Pi will be separate adapter crates. Mindwire is a reference,
not the runtime architecture.

Keep reading cost low: explicit sums/products, few named operations, short traces
and decision tables. Avoid optional convenience layers, generic frameworks,
ambiguous capability booleans and untyped control bags.

Keep native identity, chat identity, stream epoch, command identity and storage
revision distinct. Unknown outcomes and history gaps stay visible. All database
backends obey the same observable commit laws.

Use CCCC on relevant reference source and, after authorization, changed source.
Repeated identical scans do not add evidence. Record source revision, exact scope,
tool version and parser failures. CCCC does not measure prose, type algebra,
correctness or developer reading effort directly.
