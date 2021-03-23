# PartiQL Language Specification

This is the LaTeX source for the [PartiQL] specification.

## Building

As a prerequisite, you need the following:

* A LaTeX distribution such as [TeX Live][texlive].  Debian based linux distributions should install the package `texlive-full`.
* GNU Make.

To build a PDF:

```
$ make
```

To clean up the various build files including the PDF:

```
$ make clean
```

This subdirectory contains documents that have been accepted by the PartiQL team as correct. The information here should reflect the current understanding of PartiQL.

### [`/drafts`](drafts)

This subdirectory contains working versions of documents in various stages of development. While these documents are probably useful, be aware the information might be outdated or wrong.

### [`/deprecated`](deprecated)

This subdirectory contains documents that have been abandoned. When `drafts` are not moved to `docs`, they should be moved here. The docs here may be useful for learning the thoughts and considerations of past attempts, but there is no guarantee on the correctness.

## License

This specification is licensed under the [PartiQL Specification License][license]. 

The sample code within this documentation is made available under the MIT-0 license. See the LICENSE-SAMPLECODE file.

[partiql]: https://partiql.org/
[texlive]: https://www.tug.org/texlive/
[license]: LICENSE
