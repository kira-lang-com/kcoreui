# KCoreUI

The compiled visual resource system underneath a Kira UI toolkit.

A toolkit names what something means. KCoreUI decides what that name means in the current environment. A renderer draws the result and never learns the name existed.

```text
Editor ─┐
        ├──▶ writer ──▶ .kcui ──▶ parser ──▶ catalog stack ──▶ resolver
.kira ──┘                                                          │
                                                                   ▼
                                               colour, material, typeface
```

## What It Is

One function, productized:

```text
(semantic name, kind, catalog stack, environment)  ->  rendition  ->  concrete value
```

`Color.label` is not a colour. It is an identity that a catalog answers for, differently in light and dark, differently again under increased contrast, and differently again on a display with a wider gamut. Nothing in a view changes when any of that changes.

## The Two Properties

**Round-trip is byte-exact.** The parser accepts exactly the canonical form the writer emits. Opening a catalog, changing one colour and saving cannot perturb anything else in it, including resources this build has no decoder for.

**Resolution is total.** The lowest catalog on the stack is complete, an integrity check proves it against the declared name surface, and a lookup therefore always answers. Nothing above has to carry a failure path for a colour.

## The API Is The Product

The writer and the parser are the only entry point. An editor and a program building records in Kira are peers above them, and neither is privileged: a blob one writes is a blob the other opens.

A producer supplies names, kinds, constraints and payload bytes. Everything else belongs to the framework: canonical order, section layout, the specificity rule.

```kira
import KCoreUI

var record = CatalogRecord()

var label = ResourceRecord()
label.name = "label"
label.kind = kindColor()

var light = RenditionRecord()
light.constraints.append(whenAppearance(.Light))
light.payload = encodeColor(Color(r: 0.08, g: 0.09, b: 0.12, a: 1.0))
label.renditions.append(light)

var dark = RenditionRecord()
dark.constraints.append(whenAppearance(.Dark))
dark.payload = encodeColor(Color(r: 0.95, g: 0.96, b: 0.99, a: 1.0))
label.renditions.append(dark)

record.resources.append(label)

let issues = validateRecord(record)
let bytes = writeCatalogRecord(record)
```

Reading it back:

```kira
var resources = makeResources(catalog, environment)
let ink = color(resources, colorLabel())
let sidebar = material(resources, materialRegular())
```

## Agnostic Payload, Typed Index

The container never inspects a payload. It is opinionated about exactly one thing: the trait vector a rendition is selected by.

That split is what lets a reader carry a resource kind it has never heard of through load, resolution and write-back unchanged, while still resolving it correctly. Resolution only ever reads traits.

## Resolution

Two rules, both total:

**Eligibility.** A rendition is eligible when every trait it pins equals the environment's value for that trait. A trait it leaves out matches anything. A trait the environment does not carry matches nothing.

**Specificity.** Among eligible renditions, walk trait identifiers ascending and prefer the one that pins the first identifier the other leaves out. Identifier order is priority order, which is why the numbering in `app/Core/Traits.kira` is a wire fact rather than a table that could disagree with it.

The comparison is a total order, and the parser refuses a catalog containing two renditions with the same constraint list. So "best match" means one rendition, deterministically, on the VM and under LLVM alike.

## Overrides Are Layering

A custom theme is a blob, not an API.

```text
application blob   (partial)
theme blob         (partial)
default catalog    (complete, the floor)
```

A catalog that names a resource but has no rendition eligible for this environment does not stop the search. Overriding a colour in dark appearance alone is safe: light still falls through to the floor.

## References

A material's tint names a colour instead of copying one:

```kira
tintLayer(referenceColor(colorSystemBackground()), 0.68, blendNormal())
```

The reference is resolved against the same environment the material was. So the recipe is authored once and is still correct in an appearance nobody had in mind when it was written. The validator rejects reference cycles before a blob is written; the resolver carries a depth limit as a backstop for a blob that arrived from somewhere else.

## No Floats On The Wire

Every scalar in the format is a little-endian unsigned integer, and every fractional quantity is 16.16 fixed point. An IEEE encoding would be a decision about NaN, negative zero and rounding that three backends could disagree about, and a resource database that resolves differently on the VM and under LLVM is not a resource database.

One channel unit is 1/65536, so a value authored in thousandths comes back within one part in 65536 of what was written.

## Names

`app/Names` carries Apple's vocabulary in SwiftUI's spelling: the hierarchical styles, the label and fill and background ladders, the grouped backgrounds, the system palette, the six-step grey, the six materials, and the eleven text styles.

Those are names, not values. Nothing in `app/Core` may import them, and the format, the parser and the resolver never learn a single one. Swapping the whole vocabulary is a different blob and no code change.

## Layout

```text
app/
├── KCoreUI.kira      the facade: UIResources, lookups, diagnostics
├── Core/             the framework; knows no names
│   ├── Traits.kira     the trait vector and the environment
│   ├── Model.kira      catalog shape and authoring records
│   ├── Blob.kira       the container's constants
│   ├── Bytes.kira      little-endian and fixed-point primitives
│   ├── Writer.kira     records and catalogs to bytes
│   ├── Parser.kira     bytes to a catalog, or a typed refusal
│   ├── Catalog.kira    loading, name lookup, stacks
│   ├── Resolver.kira   eligibility, specificity, tracing
│   ├── Resolve.kira    the typed surface, references followed
│   ├── Payload.kira    colours and gradients
│   ├── Recipes.kira    materials, effects, typography
│   ├── Cache.kira      the two-level resolution cache
│   └── Validate.kira   cycles, dangling references, integrity
└── Names/            Apple's vocabulary; knows no format
    ├── Colors.kira
    ├── Surfaces.kira
    └── Integrity.kira

Resources/Default.kcui   the catalog that ships, and the floor
tools/bootstrap/         what wrote the first one
examples/                the rendition debugger
tests/kcoreui_kik/       the suite
```

## Building

```sh
kira check .
kira lint .
kira test --backend vm tests/kcoreui_kik
kira test --backend llvm tests/kcoreui_kik
kira run examples/rendition-debugger
kira run tools/bootstrap          # rewrites Resources/Default.kcui
```

## Scope

Icons and vector artwork are not here. They belong to a later framework, and the resource kinds this one carries are all recipes and numbers rather than pixels.

## License

Apache 2.0.
