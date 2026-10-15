# Contributing to Cybersecurity 101

Thanks for your interest in improving these lab guides! This repository is
educational material, so clarity, accuracy and safety matter more than volume.

## Ways to contribute

- **Fix errors** — typos, broken commands, out-of-date tool output.
- **Improve explanations** — make a step clearer without removing the technical detail.
- **Translate** — keep the Greek ([`guides/el/`](guides/el)) and English ([`guides/en/`](guides/en)) versions in sync.
- **Add a lab guide** — a new walkthrough for an intentionally vulnerable target.

## Guide conventions

To keep the collection consistent, please follow these rules:

1. **Authorised labs only.** Every offensive technique must target a system the reader owns or is explicitly authorised to test. Include the standard safety callout at the top of the guide.
2. **Console transcripts, not screenshots.** Show commands and their output as fenced code blocks, followed by a short explanation.

## Writing style

The guides are meant to be *read*, not just followed, so the house style is:

1. **One command per box, its output in a separate box.** A command goes in its own ` ```bash ` block; the result goes in its own ` ```text ` block, introduced by a label on a line of its own (`**Output:**` in English, `**Έξοδος:**` in Greek) with a blank line before the next fence. Never put a command and its output in the same block.
2. **Short paragraphs in the main flow.** Keep the main path to one to three sentences per paragraph. Say what the command does and how to read its output.
3. **Deep material goes in a collapsible block.** Extra flags, pitfalls, background theory, alternative commands and methodology belong in a dropdown so the main path stays readable:

   ````
   <details>
   <summary><strong>Read more</strong> — a short description</summary>

   ...text, tables and code blocks...

   </details>
   ````

   In Greek guides use `<summary><strong>Περισσότερα</strong> — …</summary>`. Always leave one blank line after the `</summary>` tag so the Markdown inside renders.
4. **Explain, don't just instruct.** After each output, say what happened and why it matters — the point is to understand the tool and the method, not to copy a command.
5. **Representative output.** Console text is illustrative; state that versions, sizes and timestamps will differ on the reader's machine.
6. **Ground each section in a concept.** Open every numbered section with a short `> **Concept.** …` block (Greek: `> **Έννοια.** …`) that gives the underlying principle — the protocol, the kernel mechanism, the model — and cite the standard inline where one exists (RFC number, POSIX, `man` section, DMTF, OWASP). Prefer precise terminology over vague wording, and keep it to two to four sentences.
7. **Place the technique in the methodology.** Name where a step sits in the assessment workflow (e.g. PTES phase) and, where it applies, the MITRE ATT&CK technique ID. Add one sentence on the defensive control that would stop it.
8. **Verify before you claim.** State how a result can be confirmed with an independent tool rather than asserting it.
3. **Numbering and language.** Keep the shared numbering — a continuous `00`–`10` sequence used by both languages. English guides live in [`guides/en/`](guides/en); Greek guides live in [`guides/el/`](guides/el), with matching filenames.
4. **Mirror changes across languages** where a translated version already exists.
5. **Reproducibility.** State the exact tool versions, target VM and network layout where relevant.

## Submitting changes

1. Fork the repository and create a topic branch (for example `fix/ufw-typos`).
2. Make your changes and update the relevant index table (`guides/en/README.md` or `guides/el/README.md`) if you add or rename a guide.
3. Open a pull request with a clear description of what changed and why.

> **Do not** submit real-world target data, credentials, or anything that targets
> systems other than an isolated lab.

## Code of conduct

Be respectful and constructive. This is a learning-oriented project, and
questions are always welcome.

## License

By contributing, you agree that your contributions are licensed under the
[MIT License](LICENSE).
