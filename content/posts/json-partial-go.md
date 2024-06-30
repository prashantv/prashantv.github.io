---
title: "Roundtrip JSON through a partial Go struct"
date: 2024-06-20T18:00:00+08:00
description: "Retain unknown fields when marshalling a Go struct"
tags: ["go", "json"]
type: post
weight: 25
showTableOfContents: true
---

Go's standard library json package makes it easy
to unmarshal JSON into a correspondingly typed struct, and vice-versa.
By default, `json.Unmarshal` ignores any fields in the JSON
that don’t match a struct field,
and `json.Marshal` will only marshal fields on the struct.
To fully roundtrip JSON from unmarshal to marshal,
the Go struct will need to represent the full schema of the JSON data.

```go
type Page struct {
	Title string `json:"title"`
	Slug  string `json:"slug"`
}

input := `{"title":"Contact Us","slug":"contact","icon":"email"}`
var p Page
if err := json.Unmarshal([]byte(input), &p); err != nil {
	return err
}

output, err := json.Marshal(p)
if err != nil {
	return err
}

fmt.Println(string(output)) // drops the "icon field.
// Output:
// {"title":"Contact Us","slug":"contact"}
```

To include the icon field in the output,
it needs to be added to the `Page` struct:

```go
type Page struct {
	Title string `json:"title"`
	Slug  string `json:"name"`
	Icon  string `json:"icon"`
}

[...]

// Output:
// {"title":"Contact Us","slug":"contact","icon":"email"}
```

This approach is reasonable when there's a small number of fields,
but it gets unwieldy for larger, complex JSON payloads.
If the Go code only needs to interact with a small subset of fields,
ideally only those would be defined in the struct.
This would make it easier to deal with schema changes,
as only changes impacting the fields used in Go
require a code change.

### Storing unknown fields in a `map[string]any`

Go can unmarshal JSON without a schema into a `map[string]any`,
which can be used to retain the original fields.
We can unmarshal the input twice:
first into a typed struct for fields that we need to interact with in Go
and again into a `map[string]any` to retain all other fields.

```go
type Page struct {
	Title string `json:"title"`
	Slug string `json:"slug"`

	raw map[string]any
}

func (p *Page) UnmarshalJSON(data []byte) error {
	if err := json.Unmarshal(data, p); err != nil {
		return err
	}
	if err := json.Unmarshal(data, &p.raw); err != nil {
		return err
	}
	return nil
}
```

However, this won't work since `json.Unmarshal(data, p)`
will end up calling the same `UnmarshalJSON` function.
We've inadvertently recursed into ourself, causing a stack overflow!

There is a simple workaround: we can create a new type,
which inherits all the same fields,
and can be casted to our original type.
But it won't inherit the `UnmarshalJSON` method
so it'll use the default behaviour of unmarshalling
into the exported fields:

```go
func (p *Page) UnmarshalJSON(data []byte) error {
	type pageNoJSON Page
	var copy pageNoJSON
	if err := json.Unmarshal(data, &copy); err != nil {
		return err
	}
	*p = Page(copy)

	if err := json.Unmarshal(data, &p.raw); err != nil {
		return err
	}

	return nil
}
```

This approach keeps the unknown fields in `Page.raw`,
but it doesn't use them for marshalling.
We can't marshal the map as-is,
as that wouldn't reflect changes made to the fields in `Page`.
Instead, we can replace the values in the map
with the field values as part of `MarshalJSON`:

```go
func (p Page) MarshalJSON() ([]byte, error) {
	p.raw["title"] = p.Title
	p.raw["slug"] = p.Slug
	return json.Marshal(p.raw)
}
```

This achieves our original goal
of only defining a schema
for fields that the Go code interacts with.
All fields are retained from unmarshal to marshal
while allowing the struct to be modified:

```go
type Page struct {
	Title string `json:"title"`
	Slug  string `json:"slug"`

	raw map[string]any
}

func (p *Page) UnmarshalJSON(data []byte) error {
	type pageNoJSON Page
	var copy pageNoJSON
	if err := json.Unmarshal(data, &copy); err != nil {
		return err
	}
	*p = Page(copy)

	if err := json.Unmarshal(data, &p.raw); err != nil {
		return err
	}

	return nil
}

func (p Page) MarshalJSON() ([]byte, error) {
	p.raw["title"] = p.Title
	p.raw["slug"] = p.Slug
	return json.Marshal(p.raw)
}

func example() error {
	input := `{"title":"Contact Us","slug":"contact","icon":"email"}`
	var p Page
	if err := json.Unmarshal([]byte(input), &p); err != nil {
		return err
	}

	p.Slug = "contact-us"

	output, err := json.Marshal(p)
	if err != nil {
		return err
	}

	fmt.Println(string(output)) // retains "icon", and has updated "slug"
	// Output:
	// {"icon":"email","slug":"contact-us","title":"Contact Us"}
}
```

This approach requires a lot of per-type noise
as every nested type will need the same
`UnmarshalJSON` and `MarshalJSON` methods.
The `UnmarshalJSON` looks similar for all types,
while the `MarshalJSON` requires duplicating the fields
defined in the schema.

Each type will need `UnmarshalJSON` and `MarshalJSON`
to override the default marshalling behavior
but the implements can be simplified with `reflect`.

### Simplifying integration with [jsonobj]

[jsonobj] builds on the above ideas
but simplifies integration for each type
by minimizing type code
and avoiding fields duplicated in marshalling.

For each type, integrating requires:

 * An unexported field, `raw jsonobj.Retain`
 * `UnmarshalJSON` method that calls `raw.UnmarshalJSON(data, obj)`
 * `MarshalJSON` method that returns `raw.ToJSON(obj)`

The above example is simplified to:

```go
type Page struct {
	raw jsonobj.Retain

	Title string `json:"title"`
	Slug  string `json:"slug"`
}

func (p *Page) UnmarshalJSON(data []byte) error {
	return p.raw.FromJSON(data, p)
}

func (p Page) MarshalJSON() ([]byte, error) {
	return p.raw.ToJSON(p)
}
```

The behaviour stays the same:
unknown fields are retained,
and changes to struct fields are reflected
in the marshalled JSON.

For more details, check out the package documentation
for [jsonobj].

### JSON v2 in the standard library

A new json v2 package in the standard library
is [planned](https://github.com/golang/go/discussions/63397)
and will include support for unknown fields,
simplifying the integration once it's released!

### Related

 * [go issue: proposal: encoding/json: preserve unknown fields](https://github.com/golang/go/issues/22533)
 * [go discussion: encoding/json/v2](https://github.com/golang/go/discussions/63397)
 * [reddit: Unmarshal some fields in struct and the rest in map](https://www.reddit.com/r/golang/comments/q4gyr9/unmarshal_some_fields_in_struct_and_the_rest_in/)
 * [stack overflow: Unmarshal JSON with some known, and some unknown field names](https://stackoverflow.com/questions/33436730/unmarshal-json-with-some-known-and-some-unknown-field-names)
 * [jsonobj]

[jsonobj]: https://pkg.go.dev/github.com/prashantv/pkg/jsonobj

