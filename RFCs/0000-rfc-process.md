### Summary

This RFC (request for comment) outlines the process that PartiQL maintainers and contributors should follow in order to introduce new changes to the language.

This RFC process is inspired by [Rust RFCs.](https://github.com/rust-lang/rfcs)

### Motivation
The motivation is to provide governance to PartiQL changes by which these changes are reviewed in a consistent and controlled manner.

*Change to PartiQL* is any change to the PartiQL Specification, any of its implementations, this RFC, or changes affecting long term decision-making for maintaining and evolving the PartiQL open source project.

# Guide-level explanation
The changes that require RFC are but not limited to the following:

* Any semantic or syntactic changes to the language that are not bug fixes—e.g. addition of temporal table semantics.
* Any changes to the public APIs—e.g. addition of new planner API.
* Any backward incompatible changes—e.g. deprecating `ExprNode`.

The changes that don’t require RFC are but not limited to the following:

* Rephrasing or rewording the existing specification where the change does not affect the semantics—e.g. making ORDER BY section more clear.
* Bug fixes.
* Implementing features that already exist in PartiQL specification—e.g. implementing ORDER BY.
* Performance improvements—e.g. improve the PartiQL’s CLI performance.
* Changes to that back-end that don't break consumers.

## The Process

Before submitting the RFC, discuss your idea using a pre-RFC document with PartiQL maintainers and community via [PartiQL Forum](https://community.partiql.org/faq)or [GitHub](https://github.com/partiql/). This helps to solicit early feedback and gather a sense of how the proposed change will be seen by PartiQL maintainers and community.

1. Fork this repository at https://github.com/partiql/partiql-doc
2. Copy `RFCs/templates/RFC-template.adoc` to `RFCs/0000-RFC-NAME.adoc`; do not assign a number yet.
3. Complete the RFC text using the template.
4. Create a pull request to this repository.
5. Add the pull request number to the RFC filename and pull request address to the placeholder in the template.
6. Submit a new commit.
7. The request enters the review process using the pull request.
8. PR gets closed upon the outcome of the review.

### Approved proposals

After the proposal is approved by a member of the PartiQL committee:

1. The PR gets merged with the pull request number as its Id.
2. The document is added to the Active RFCs list under `RFCs/active-rfcs.md`.
3. The proposed change may be implemented.
4. Modifications to the RFC can be done in follow-up pull requests; the aim should be to have changes to the RFC as final, but sometimes changes are inevitable.

### Rejected proposals

The proposal is discarded without further action. This does not necessarily mean its ideas are never going to make it into PartiQL, just that at this point it was decided not to continue the idea further.
