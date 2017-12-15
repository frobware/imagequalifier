[![Travis CI](https://travis-ci.org/frobware/imagequalifier.svg?branch=master)](https://travis-ci.org/frobware/imagequalifier)
[![GoDoc](https://img.shields.io/badge/godoc-reference-blue.svg?style=flat-square)](https://godoc.org/github.com/frobware/imagequalifier)
[![Coverage Status](http://codecov.io/github/frobware/imagequalifier/coverage.svg?branch=master)](http://codecov.io/github/frobware/imagequalifier?branch=master)
[![Report Card](https://goreportcard.com/badge/github.com/frobware/imagequalifier)](https://goreportcard.com/report/github.com/frobware/imagequalifier)

# A library that qualifies unqualified Docker images.

This library and CLI tool is an experiment that, given a set of rules,
will qualify bare Docker images names to include a domain component.
An unqualified image is something like `library/emacs` or `emacs`, or
`emacs:v26`. A qualified image has a domain component (e.g.,
`prep.ai.mit.edu/library/emacs`).

## Pattern Matching Rules

Rules are very simple and are encoded in a text file on a line-by-line
basis. There are two fields per line, separated by one or more
whitespace characters. The first field is a pattern to match against;
the second field is the domain to be used when a match occurs. Empty
lines are skipped, as are [comment] lines that being with '#'. The
rules should be listed from most specific to least specific. No
reordering is attempted (I tried this and was unhappy with the
results). *The first pattern match wins*.

### Pattern Matching Implementation

This implementation currently relies on the semantics of
`path.Match()`:

	$ go doc path Match
	func Match(pattern, name string) (matched bool, err error)

		Match reports whether name matches the shell file name pattern. The pattern
		syntax is:

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

		Match requires pattern to match all of name, not just a substring. The only
		possible returned error is ErrBadPattern, when pattern is malformed.

## Examples

The simplest (redirect) case is:

    * corp.io

This will transform a `nginx` image reference to `corp.io/nginx`. It
will not, however, transform `library/foo` to `corp.io` because it
uses path matching and the image reference contains a `/` but there is
no corresponding pattern that will be matched.

## Sample Rules

	$ cat $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules

	busybox                    busybox.com
	*/emacs                    emacs.com
	myrepo/*                   myrepo.com
	nginx:v[1-9].[0-9]*        nginx.com
	*/*                        corp.io
	*                          corp.io

## Test by example

	$ go install github.com/frobware/imagequalifier/...

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules emacs
	corp.io/emacs

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules library/emacs
	emacs.com/library/emacs

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules foo/emacs
	emacs.com/foo/emacs

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules nginx
	corp.io/nginx

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules nginx:latest
	corp.io/nginx:latest

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules nginx:v0.0
	corp.io/nginx:latest

	$ imagequalifier $GOPATH/src/github.com/frobware/imagequalifier/testdata/sample-rules nginx:v1.0
	nginx.com/nginx:v1.0
