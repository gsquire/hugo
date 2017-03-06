+++
date = "2017-03-06T14:30:42-08:00"
title = "Cleaner Go"

+++

Large code bases can become increasingly complex to manage as you add more code while
also refactoring what is there already. Even with diligent code reviews, it is easy to gloss over
common programming mistakes. Sometimes code can be written in a simpler way. Or perhaps a recent
refactor left a few functions in that are no longer used.

Luckily the `go` tool includes a sub-command that will examine Go source and report suspicious lines
based on heuristics that the compiler will not catch. You can run `go vet` over you entire source
tree as such:

```sh
go vet ./...
```

The vet sub-command is limited by what it catches but is a good first pass that can catch errors
that a developer might not.

## Static Analysis
Another set of tools for analyzing your Go code is under the [go-tools](https://github.com/dominikh/go-tools)
repository. These commands are a great addition to `go vet` in the sense that they provide a much more
extensive review of your source. Each tool can be installed with `go get` if you wish to pick out
certain ones you find useful.

Below are some examples of the tools I find most valuable. Even though these
examples are contrived, they will compile and pass `go vet`.

#### Simplifying Code
`gosimple` analyzes your code to see if it can be written in a simpler manner. For example:

```go
package main

import "fmt"

func bad(b bool) {
	if b == true {
		fmt.Println("true")
	} else {
		fmt.Println("false")
	}
}

func main() {
	bad(true)
}
```

When running `gosimple main.go` you will see this output:

`main.go:6:5: should omit comparison to bool constant, can be simplified to b (S1002)`

#### Static Checks
`staticcheck` is a great lint because it can find subtle issues in your Go code. For example:

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	s := strings.Replace("bad-bad-bad", "-", "&", 0)
	fmt.Println(s)
}
```

When running `staticcheck main.go` you will see this output:

`main.go:9:48: calling strings.Replace with n == 0 will return no results, did you mean -1? (SA1018)`

There are many items that `staticcheck` can catch and they are listed in the
[README](https://github.com/dominikh/go-tools/tree/master/cmd/staticcheck). During a recent
refactor of some of my code I had silently omitted an error check in a function. There
were other checks throughout the function but if it had failed in the step without a check, the error variable
would have had the wrong value. This would have been a nightmare to debug.

#### Unused
When refactoring it is often easy to forget about certain bits of code. There may be functions or
variables that are defined but not used anywhere. This is often referred to as dead code. For example:

```go
package main

import "fmt"

const a = 5

func main() {
	fmt.Println("hello")
}
```

When running `unused main.go` you will see this output:

```sh
main.go:5:7: const a is unused (U1000)
```

#### Git Hooks
I found that these tools were important in my workflow and decided to add them as a pre-commit hook
in git. For those unfamiliar with git hooks, the documentation for them can be found [here](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks).
They essentially provide a way to customize your git workflow and enforce things such as commit
message format or linting.

In this case I want to run the tools above before a commit is made. Git's pre-commit hook is run
before a new commit is made in the current branch as implied by the name. It adds a little extra time to the process but
it is nice to have in the sense that it catches problems and allows you to adjust the working tree. This
ensures that all commits are up to your standard.

Here is the pre-commit hook I use for all my go projects:

```sh
#!/bin/bash

check_exit() {
    if [ $? -eq 1 ]; then
        printf "linting failed when running $1...\n"
        exit 1
    fi
}

go vet $(glide nv)
check_exit "go vet"

gosimple $(glide nv)
check_exit "gosimple"

unused $(glide nv)
check_exit "unused"

staticcheck $(glide nv)
check_exit "staticcheck"
```

The `glide nv` command lists all non-dependency code used in a project. It's akin to `./...` in the sense
that is runs over all packages. Glide is the dependency manager I use for my projects and the site
can be found [here](https://glide.sh) for those that are interested.

#### Summary
By adding static analysis tools to your Go code, you can help prevent common coding mistakes and
make your development environment better in the process. Having this be automatic is also a
catalyst for more iterative and effective coding.
