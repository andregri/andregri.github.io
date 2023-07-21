---
layout: single
title: Makefile tips
toc: true
tags: make
---

This post talks about three basic features that you need when writing basic makefiles:

- phony targets to write recipes that doesn't produce files.
- escape $ when you need to run sub commands in recipes.
- types of variable assignments.

# Phony targets

From ["Phony Targets" documentation of GNU Make](https://www.gnu.org/software/make/manual/html_node/Phony-Targets.html):

> A phony target is one that is not really the name of a file; rather it is just a name for a recipe to be executed when you make an explicit request. There are two reasons to use a phony target: to avoid a conflict with a file of the same name, and to improve performance.
> 

Use **phony targets** when the recipe name is not the name of the file to produce but just the name of the make rule. The recipe is executed every time the phony target is invoked.

For instance, the `clean` recipe usually doesn't produce any file but remove produced artefacts, so it should always run when invoked:

```makefile
.PHONY: clean
clean:
	rm bin/tool
```

Without `.PHONY: clean`, the recipe doesn't run if there is a file called `clean` in the same directory.

# Escape $

From [“Using Variables in Recipe” documentation of GNU Make](https://www.gnu.org/software/make/manual/html_node/Variables-in-Recipes.html):

> […] if you want a dollar sign to appear in your recipe, you must double it (‘$$’). […] whether the variable you want to reference is a `make` variable (use a single dollar sign) or a shell variable (use two dollar signs).
> 

There is no much to show in example below:

```makefile
all:
	curl localhost:$$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
```

You can use `—dry-run` flag if you are not sure how variables are expanded. Consider the following makefile that define the variable `bar` as the command `$(date)`:

```makefile
bar = $$(date)

.PHONY: all
all:
	@echo bar is $(bar).
```

Execute a dry run of the makefile to see that the bar variable is expanded to `$(date)`:

```console
$ make all --dry-run
echo bar is $(date).
```

# Variable assignment

From [“The Two Flavors of Variables” documentation of GNU Make](https://www.gnu.org/software/make/manual/html_node/Flavors.html):

- `=` for ***recursively expanded* variables** is evaluated during substitution and thus, each time, the variable is used.

```makefile
x := foo
y := $(x) bar
x := later

.PHONY: printx
printx:
	@echo x is $(x)

.PHONY: printy
printy:
	@echo y is $(y)

.PHONY: all
all: printx printy
```

If you run this makefile, `y` variable is evaluated after `x` variable has changed twice. When `printy` is called, `y` variable is evaluated with the last value of `x`:

```console
$ make all
x is later
y is later bar
```

- `:=` for **simply expanded variables***: o*nce that expansion is complete the value of the variable is **never expanded again**. When the variable is used the value is copied verbatim as the expansion.

```makefile
x := foo
y := $(x) bar # the only line we changed
x := later

.PHONY: printx
printx:
	@echo x is $(x)

.PHONY: printy
printy:
	@echo y is $(y)

.PHONY: all
all: printx printy
```

If you run this makefile, `y` is expanded when `x` is still `foo` even if it is changed to `later`:
```console
$ make all
x is later
y is foo bar
```
