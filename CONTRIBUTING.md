# Contributing to Cybersecurity 101

Thanks for your interest in improving these lab guides! This repository is
educational material, so clarity, accuracy and safety matter more than volume.

## Ways to contribute

- **Fix errors** — typos, broken commands, out-of-date tool output.
- **Improve explanations** — make a step clearer without removing the technical detail.
- **Translate** — keep the Greek (`_el`) and English (`docs/`) versions in sync.
- **Add a lab guide** — a new walkthrough for an intentionally vulnerable target.

## Guide conventions

To keep the collection consistent, please follow these rules:

1. **Authorised labs only.** Every offensive technique must target a system the reader owns or is explicitly authorised to test. Include the standard safety callout at the top of the guide.
2. **Console transcripts, not screenshots.** Show commands and their output as fenced code blocks, followed by a short explanation.
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
