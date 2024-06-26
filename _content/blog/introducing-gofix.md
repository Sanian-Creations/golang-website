---
title: Introducing Gofix
date: 2011-04-15
by:
- Russ Cox
tags:
- gofix
- technical
summary: How to use go fix to update your code with each new Go release.
---


The next Go release will include significant API changes in several fundamental Go packages.
Code that [implements an HTTP server handler](http://codereview.appspot.com/4239076),
[calls `net.Dial`](http://codereview.appspot.com/4244055),
[calls `os.Open`](http://codereview.appspot.com/4357052),
or [uses the reflect package](http://codereview.appspot.com/4281055) will
not build unless it is updated to use the new APIs.
Now that our releases are [more stable and less frequent](/blog/go-becomes-more-stable),
this will be a common situation.
Each of these API changes happened in a different weekly snapshot and might
have been manageable on its own;
together, however, they represent a significant amount of manual effort
to update existing code.

[Gofix](/cmd/fix/) is a new tool that reduces the amount
of effort it takes to update existing code.
It reads a program from a source file, looks for uses of old APIs,
rewrites them to use the current API, and writes the program back to the file.
Not all API changes preserve all the functionality of an old API,
so gofix cannot always do a perfect job.
When gofix cannot rewrite a use of an old API,
it prints a warning giving the file name and line number of the use,
so that a developer can examine and rewrite the code.
Gofix takes care of the easy, repetitive,
tedious changes, so that a developer can focus on the ones that truly merit attention.

Each time we make a significant API change we’ll add code to gofix to
take care of the conversion,
as much as mechanically possible.
When you update to a new Go release and your code no longer builds,
just run gofix on your source directory.

You can extend gofix to support changes to your own APIs.
The gofix program is a simple driver around plugins called fixes that each
handle a particular API change.
Right now, writing a new fix requires doing some scanning and rewriting
of the go/ast syntax tree,
usually in proportion to how complex the API changes are.
If you want to explore, the [`netdialFix`](https://go.googlesource.com/go/+/go1/src/cmd/fix/netdial.go),
[`osopenFix`](https://go.googlesource.com/go/+/go1/src/cmd/fix/osopen.go),
[`httpserverFix`](https://go.googlesource.com/go/+/go1/src/cmd/fix/httpserver.go),
and [`reflectFix`](https://go.googlesource.com/go/+/go1/src/cmd/fix/reflect.go)
are all illustrative examples,
in increasing order of complexity.

We write Go code too, of course, and our code is just as affected by these
API changes as yours.
Typically, we write the gofix support at the same time as the API change
and then use gofix to rewrite the uses in the main source tree.
We use gofix to update other Go code bases and our personal projects.
We even use gofix to update Google’s internal source tree when it is time
to build against a new Go release.

As an example, gofix can rewrite code like [this snippet from `fmt/print.go`](http://codereview.appspot.com/4353043/diff/10001/src/pkg/fmt/print.go#newcode657):

	switch f := value.(type) {
	case *reflect.BoolValue:
	    p.fmtBool(f.Get(), verb, field)
	case *reflect.IntValue:
	    p.fmtInt64(f.Get(), verb, field)
	// ...
	case reflect.ArrayOrSliceValue:
	    // Byte slices are special.
	    if f.Type().(reflect.ArrayOrSliceType).Elem().Kind() == reflect.Uint8 {
	        // ...
	    }
	// ...
	}

to adapt it to the new reflect API:

	switch f := value; f.Kind() {
	case reflect.Bool:
	    p.fmtBool(f.Bool(), verb, field)
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
	    p.fmtInt64(f.Int(), verb, field)
	// ...
	case reflect.Array, reflect.Slice:
	    // Byte slices are special.
	    if f.Type().Elem().Kind() == reflect.Uint8 {
	        // ...
	    }
	// ...
	}

Nearly every line above changed in some small way.
The changes involved in the rewrite are extensive but nearly entirely mechanical,
just the kind of thing that computers are great at doing.

Gofix is possible because Go has support in its standard libraries for [parsing Go source files into syntax trees](/pkg/go/parser)
and also for [printing those syntax trees back to Go source code](/pkg/go/printer).
Importantly, the Go printing library prints a program in the official format
(typically enforced via the gofmt tool),
allowing gofix to make mechanical changes to Go programs without causing
spurious formatting changes.
In fact, one of the key motivations for creating gofmt—perhaps second
only to avoiding debates about where a particular brace belongs—was to
simplify the creation of tools that rewrite Go programs, as gofix does.

Gofix has already made itself indispensable.
In particular, the recent reflect changes would have been unpalatable without
automated conversion,
and the reflect API badly needed to be redone.
Gofix gives us the ability to fix mistakes or completely rethink package
APIs without worrying about the cost of converting existing code.
We hope you find gofix as useful and convenient as we have.
