# Mustache

[![Mustache](http://pkg.julialang.org/badges/Mustache_0.6.svg)](http://pkg.julialang.org/?pkg=Mustache)

Linux: [![Build Status](https://travis-ci.org/jverzani/Mustache.jl.svg?branch=master)](https://travis-ci.org/jverzani/Mustache.jl)
&nbsp;
Windows: [![Build Status](https://ci.appveyor.com/api/projects/status/github/jverzani/Mustache.jl?branch=master&svg=true)](https://ci.appveyor.com/project/jverzani/mustache-jl)

[Mustache](http://mustache.github.io/) is 

    ... a logic-less template syntax. It can be used for HTML,
    config files, source code - anything. It works by expanding tags in a
    template using values provided in a hash or object.

This package ports over most of the [mustache.js](https://github.com/janl/mustache.js) implementation for use in [Julia](http://julialang.org). All credit should go there. All bugs are my own.

## Examples

Following the main [documentation](http://mustache.github.io/mustache.5.html) for `Mustache.js` we have a "typical Mustache template" defined by:


```julia
using Mustache

tpl = mt"""
Hello {{name}}
You have just won {{value}} dollars!
{{#in_ca}}
Well, {{taxed_value}} dollars, after taxes.
{{/in_ca}}
"""
```

The values with braces (mustaches on their side) are looked up in a view, such as a dictionary or module. For example,

```julia
d = Dict()
d["name"] = "Chris"
d["value"] = 10000
d["taxed_value"] = 10000 - (10000 * 0.4)
d["in_ca"] = true

render(tpl, d)
```

Yielding

```
Hello Chris
You have just won 10000 dollars!

Well, 6000.0 dollars, after taxes.
```

The `render` function pieces things together. Like `print`, the first
argument is for an optional `IO` instance. In the above example, where
one is not provided, the `sprint` function is employed.


The second argument is a either a string or a mustache template. As
seen, templates can be made through the `mt` non-standard string
literal. The advantage of using `mt`, is the template will be
processed at compile time so its reuse will be faster.

The templates use tags comprised of matching mustaches (`{}`), either two or three, to
indicate a value to be substituted for.

The third argument is for a view to provide values to substitute into
the template. The above example used a dictionary. A Module may also
be used, such as `Main`:


```julia
name, value, taxed_value, in_ca = "Christine", 10000, 10000 - (10000 * 0.4), false
render(tpl, Main) |> print
```

Which yields:

```
Hello Christine
You have just won 10000 dollars!
```


Further, keyword
arguments can be used when the variables in the templates are symbols:

```julia
goes_together = mt"{{{:x}}} and {{{:y}}}."
render(goes_together, x="Salt", y="pepper")
render(goes_together, x="Bread", y="butter")
```

As well, one can use Composite Kinds. This may make writing `show` methods easier:

```julia
using Distributions
tpl = "Beta distribution with alpha={{α}}, beta={{β}}"
render(tpl, Beta(1, 2))
```

gives

```
"Beta distribution with alpha=1.0, beta=2.0"
```

### Variables

Tags representing variables have the form `{{varname}}`,
`{{:symbol}}`, or their triple-braced versions `{{{varname}}}` or
`{{{:symbol}}}`.  The triple brace prevents HTML substitution for
entities such as `<`. The following are escaped when only double
braces are used: "&", "<", ">", "'", "\", and "/".

### Sections

In the main example, the template included:

```
{{#in_ca}}
Well, {{taxed_value}} dollars, after taxes.
{{/in_ca}}
```

Tags beginning with `#varname` and closed with `/varname` create
sections. The part between them is used only if the variable is
defined. Related, if the tag begins with `^varname` and ends with
`/varname` the text between these tags is included only if the
variable is *not* defined.

### Iteration

If the section variable, `{{#VARNAME}}`, binds to an iterable
collection, then the text in the section is repeated for each item in
the collection with the view used for the context of the template
given by the item.

This is useful for collections of named objects, such as DataFrames
(where the collection is comprised of rows) or arrays of
dictionaries. The special variable `{{.}}` can be used to iterate over non-named collections.

For data frames the variable names are specified as
symbols or strings. Here is a template for making a web page:

```julia
tpl = mt"""
<html>
<head>
<title>{{:TITLE}}</title>
</head>
<body>
<table>
<tr><th>name</th><th>summary</th></tr>
{{#:D}}
<tr><td>{{:names}}</td><td>{{:summs}}</td></tr>
{{/:D}}
</body>
</html>
"""
```
This can be used to generate a web page for `whos`-like values:

```julia
_names = Array{String}(0)
_summaries = Array{String}(0)
m = Main
for s in sort(map(string, names(m)))
    v = Symbol(s)
    if isdefined(m,v)
        push!(_names, s)
        push!(_summaries, summary(eval(m,v)))
    end
end

using DataFrames
d = DataFrame(names=_names, summs=_summaries)

out = render(tpl, TITLE="A quick table", D=d)
print(out)
```


This can be compared to using an array of `Dict`s, convenient if you have data by the row:

```julia
A = [Dict("a" => "eh", "b" => "bee"),
     Dict("a" => "ah", "b" => "buh")]
tpl = mt"{{#:A}}Pronounce a as {{a}} and b as {{b}}. {{/:A}}"
render(tpl, A=A) |> print
```

yielding

```
Pronounce a as eh and b as bee. Pronounce a as ah and b as buh.
```

The same approach can be made to make a LaTeX table from a data frame:

```julia

function df_to_table(df, label="label", caption="caption")
	fmt = repeat("c", length(df))
    row = join(["{{$x}}" for x in map(string, names(df))], " & ")

tpl="""
\\begin{table}
  \\centering
  \\begin{tabular}{$fmt}
{{#:DF}}    $row\\\\
{{/:DF}}  \\end{tabular}
  \\caption{$caption}
  \\label{tab:$label}
\\end{table}
"""

render(tpl, DF=df)
end
```

(A string is used -- and not a `mt` macro above -- so that string interpolation can happen.)

### Iterating over vectors

Iterating over an unnamed vector uses `{{.}}` to refer to the item:

```julia
tpl = "{{#:vec}}{{.}} {{/:vec}}"
render(tpl, vec = ["A1", "B2", "C3"])  # "A1 B2 C3 "
```

Not the extra space after `C3`. There is *experimental* support for
indexing with the iteration of a vector that allows on to work around
this. The syntax `.[ind]` refers to the value `vec[ind]`.

To print commas one can use this pattern:

```julia
tpl = "{{#:vec}}{{.}}{{^.[end]}}, {{/.[end]}}{{/:vec}}"
render(tpl, vec = ["A1", "B2", "C3"])  # "A1, B2, C3"
```

To put the first value in bold, but no others, say:

```julia
tpl = """
{{#:vec}}
{{#.[1]}}<bold>{{.}}</bold>{{/.[1]}}
{{^.[1]}}{{.}}{{/.[1]}}
{{/:vec}}
"""
render(tpl, vec = ["A1", "B2", "C3"])  # basically "<bold>A1</bold>B2 C3"
```

This was inspired by
[this](http://stackoverflow.com/questions/11147373/only-show-the-first-item-in-list-using-mustache)
question, but the syntax chosen was more Julian. This syntax -- as
implemented for now -- does not allow for iteration. That is
constructs like `{{#.[1]}}` don't introduce iteration, but only offer
a conditional check.

### Partials

Partials are used to include partial templates into a template.

Partials begin with a greater than sign, like `{{> box.tpl }}`. In this example, the file `box.tpl` is opened and inserted into the template, then populated. A full path may be specified.

They also inherit the calling context. 

In this way you may want to think of partials as includes, imports,
template expansion, nested templates, or subtemplates, even though
those aren't literally the case here.

The partial specified by `{{< box.tpl }}` is not parsed, rather included as is into the file. This can be much faster.

## Alternatives

`Julia` provides some alternatives to this package which are better suited for many jobs:

* For simple substitution inside a string there is string
  [interpolation](http://julia.readthedocs.org/en/latest/manual/strings/).

* For piecing together pieces of text either the `string` function or
  string concatenation (the `*` operator) are useful.

* For formatting numbers and text, the
  [Formatting.jl](https://github.com/JuliaLang/Formatting.jl) package
  is available.


## Differences from Mustache.js

This project deviates from that of Mustache.js in a few significant ways:

* The tags are only demarcated with mustaches, this is not customizable
* Julian structures are used, not JavaScript objects. As illustrated,
  one can use Dicts, Modules, DataFrames
