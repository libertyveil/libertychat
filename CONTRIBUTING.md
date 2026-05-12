# Contributing to LibertyChat

LibertyChat is in concept phase. Most contributions right now are to the design documents. Once implementation begins, this guide will expand to cover code contributions.

## How to participate

### Discuss the design

- Open a GitHub issue for any specific design question, ambiguity, or proposed change.
- Propose substantial changes via pull request: edit the relevant `.md` files and explain in the PR description.
- Tag issues with `design`, `crypto`, `protocol`, `mobile`, etc. as appropriate.

### Propose new ideas

- For any non-trivial design proposal, write a brief RFC and submit it as a pull request to a new file in `rfcs/` (we'll create this directory as needed).
- RFC format:
  - **Summary**: one paragraph explaining the proposal
  - **Motivation**: what problem does this solve
  - **Detailed design**: concrete spec changes
  - **Drawbacks**: what we lose by adopting this
  - **Alternatives considered**: other approaches and why they were rejected
  - **Open questions**: unresolved aspects

### Contribute code (later)

Once `libertychat-core` exists:

- Follow Rust idioms (clippy, rustfmt enforced via CI)
- All public APIs must have documentation
- Cryptographic code must have test vectors
- New features need design documents in this repo first

## Decision making

Until the project has a formal governance structure, decisions are made by:

1. Discussion in issues / PRs
2. Rough consensus among active contributors
3. The repo maintainer breaks ties when consensus doesn't form

Goal is to move toward more formal governance (technical steering committee) as the project grows.

## Code of conduct

Treat each other with respect. Disagree on technical merits, not on personalities. We don't have a formal CoC document yet; until then, use common sense and assume good faith.

## License of contributions

By submitting a PR, you agree that your contribution is licensed under the project's AGPL-3.0-or-later license.

No CLA required. Standard inbound = outbound model.

## Communication channels

- **GitHub issues / PRs**: design discussions, bug reports
- **Matrix room**: TBD
- **Email**: TBD

## Areas where help is most needed

In approximate order:

1. **Cryptographic protocol review**: Anyone with crypto background should poke holes in the design before we lock anything down.
2. **Threat model formalization**: Help write down the threat model in formal notation (TLA+, Tamarin, etc.) to enable automated checking.
3. **Design feedback**: Real-world use cases that the current design doesn't handle well.
4. **Standards expertise**: Anyone who knows MLS, Noise, libp2p, or related standards deeply.
5. **Mobile architecture**: How to structure mobile clients for battery life, push notifications, multi-device.

If your skills don't match these but you want to help, open an issue with what you can contribute and we'll figure it out.
