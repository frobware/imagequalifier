[![Travis CI](https://travis-ci.org/frobware/imagequalifier.svg?branch=master)](https://travis-ci.org/frobware/imagequalifier)
[![GoDoc](https://img.shields.io/badge/godoc-reference-blue.svg?style=flat-square)](https://godoc.org/github.com/frobware/imagequalifier)
[![Coverage Status](http://codecov.io/github/frobware/imagequalifier/coverage.svg?branch=master)](http://codecov.io/github/frobware/imagequalifier?branch=master)
[![Report Card](https://goreportcard.com/badge/github.com/frobware/imagequalifier)](https://goreportcard.com/report/github.com/frobware/imagequalifier)

This library and CLI tool is an experiment that, given a set of rules,
will qualify bare images names to include a domain (e.g.,
`registry.io`) component.

An unqualified image is something like `library/emacs` or `emacs`, or
`emacs:v26`. A qualified image has a domain component (e.g.,
`prep.ai.mit.edu/library/emacs`).

## Pattern Matching Rules

- Rules are new-line separated
- There are two fields per line, separated by one or more whitespace characters
- First field is the pattern
- Second field is the domain
- Blank/Empty lines are skipped
- Comment lines are skipped and begin with #.

### Rule Ordering (aka Partial Ordering)

(TBD) Should rules be automagically ordered when parsed?

- Currently no reordering of the rules is attempted
- Rules are attempted in the order presented
- _the first match wins_

## Pattern Syntax

Note: semantics stolen from `path.Match()`.

		pattern:
			{ term }
		term:
			'*'         matches any sequence of non-/ characters
			'?'         matches any single non-/ character
			'[' [ '^' ] { character-range } ']'
						character class (must be non-empty)
			c           matches character c (c != '*', '?', '\\', '[')
			'\\' c      matches character c

		character-range:
			c           matches character c (c != '\\', '-', ']')
			'\\' c      matches character c
			lo '-' hi   matches character c for lo <= c <= hi

Matching requires pattern to match all of name, not just a substring.

## Examples

The simplest pattern to pull images that do not contain a repository
component from `internal.io` is:

	* internal.io

This pattern will transform any bare image to `internal.io/<image>`.
Note: it will not transform `myrepo/nginx` to
`internal.io/myrepo/nginx` because the pattern needs to match all
segments of the [pattern] path.

To match any image and force it to pull from `internal.io` the
following rules are required:

	*/*   internal.io
	*     internal.io

Making all latest tags come from a dev registry:

	*/*:latest dev.io
	*/*        internal.io
	*          internal.io

Forcing nginx v1.2.3 to come from docker.io.

	*/nginx:v1.2.3   docker.io

	you/*            you.io
	me/*             me.io

	*/*              internal.io
	*                internal.io

### Sample Rules

	$ cat $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules

	busybox                    busybox.com
	*/emacs                    emacs.com
	myrepo/*                   myrepo.com
	nginx:v[1-9].[0-9]*        nginx.com
	*/*                        internal.com
	*                          external.com

### Sample Matches

| Image          | Qualified Image          |
|----------------|--------------------------|
| `emacs`        | `external.com/emacs`     |
| `busybox`      | `busybox.com/busybox`    |
| `myrepo/emacs` | `emacs.com/myrepo/emacs` |
| `nginx`        | `external.com/nginx`     |
| `nginx:v1.0`   | `nginx.com/nginx`        |

### Test by example

	$ go install github.com/frobware/imagequalifier/...

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules emacs
	internal.io/emacs

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules library/emacs
	emacs.com/library/emacs
