# Interoperability Service

[Code & Installation Instructions](https://github.com/interoperability-control-modules/interoperability-service){ .md-button }

The Interoperability Service lets control applications address Distributed Energy Resource (DER) data by a stable,
protocol-neutral identifier and receive it in whatever data format they understand. It provides templates for
standards-based communication and reduces integration complexity:

* **Reduces vendor-specific integration effort and deployment risk.** Supports interoperability across protocols
  including SunSpec Modbus, IEEE 1815.2 (DNP3 / MESA-DER), IEEE 2030.5, IEC 61850-7-420, the IEEE 1547.1
  functional view, and OpenFMB.
* **Translates communication interfaces and data models** to enable seamless device-to-application interaction
  regardless of the native protocol of each device.
* **Enables protocol-agnostic control and reference implementations.** Control applications such as the
  [Real-Time Control Agent](rt-control.md) and [Scheduler](scheduler.md) are decoupled from the underlying
  communication protocols, ensuring reliable and consistent operation across multiple ESS and DER vendors.

It does this with two cooperating components: a **mapping engine** that resolves a Uniquely Addressable Identifier
(UAI) to the canonical resource that actually publishes the data, following aliases as needed, and a **transform
registry** that holds declarative field-mapping rules between data formats and finds a chain of transforms from the
format a resource is published in to the format a consumer asked for. Consumers call one RPC, `resolve`, and get back
the resource definition plus the transform needed to read it in their preferred format.

As illustrated in [](#interop-service-mapping), the service acts as a translator of communication interfaces and data
models between control stations or outstations that implement different protocols.

Figure: The Interoperability Service translates communication interfaces and data models between IEC 61850 IEDs,
SunSpec Modbus, IEEE 1815.2 (DNP3), IEEE 2030.5 (HTTP), and OpenFMB nodes on a message bus. {#interop-service-mapping}

![](images/interop-service-mapping.png)

## Graph- and Transform-Based Interoperability

Rather than maintaining a direct mapping between every pair of protocols, the service maps each protocol-specific
data model once onto a common data model, as shown in [](#interop-service-graph-transform). Transforms between data
formats are stored as edges in a directed graph keyed by format name. A conversion from one format to another is
found as a path through this graph, so multi-stage transformations are composed automatically from reusable steps.
For example, with the bundled IEEE 1815.2 to IEC 61850 and IEC 61850 to SunSpec transforms registered, a request for
a DNP3 device's data in SunSpec form can be served by chaining the two without any direct DNP3 to SunSpec rule set.
When several chains connect two formats, the registry picks the one that preserves the most information, as
described under [Choosing between chains](#choosing-between-chains). This approach:

* Extends interoperability using graph-based mappings and transformation pipelines.
* Expands mappings between protocol-specific and common data models.
* Enables multi-stage transformations rather than maintaining direct mappings between every protocol pair.
* Supports reusable transformation paths across SunSpec Modbus, IEEE 1815.2 / DNP3, IEEE 2030.5, IEEE 1547.1,
  OpenFMB, and other supported interfaces.
* Reduces protocol-specific custom integration.

Figure: Generalized graph- and transform-based interoperability. Source and target protocols are mapped once through a
common data model, and multi-stage transformations are derived from the transform graph.
{#interop-service-graph-transform}

![](images/interop-service-graph-transform.png)

## Concepts

### Uniquely Addressable Identifiers

A UAI is an ordered tuple of path segments, for example `("site1", "feeder2", "pv_inverter")`. Internally the
service stores all UAIs in a tree rooted at `uai`; each segment becomes a node. A UAI may be given as a tuple, a list,
or a JSON-encoded string of the tuple.

Resolution is **longest-prefix** by default. If `("site1", "feeder2", "pv_inverter", "W")` is requested and no node
exists for that full path, the engine walks up to `("site1", "feeder2", "pv_inverter")` and so on until it finds a
resource. Pass `strict=True` to require an exact match.

### Resources

A leaf in the UAI tree is a resource. There are two kinds:

| Type        | Fields                                                   | Meaning                                                                                                                                              |
|-------------|----------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `canonical` | `data_format`, `owner`, `publication_topic`, `rpc_topic` | The authoritative source of the data. `publication_topic` is the message bus topic the data appears on; `rpc_topic` is where writes go.              |
| `alias`     | `data_format`, `owner`, `references`                     | A name that points at another UAI (`references`). `data_format` is the format the alias presents itself in.                                          |

Aliases can chain. When an alias is resolved the engine follows `references` until it reaches a canonical resource.
If the caller did not name a target format, the alias's own `data_format` is used as the target, so an alias exposes
an existing device as viewed through a different standard, such as IEC 61850.

### Formats and transforms

A **transform** is a declarative rule set that converts a message in one `input_format` to one in an
`output_format`. Transforms are stored as edges in a directed graph keyed by format name. When a consumer asks for a
resource in a format other than the one it is published in, the registry finds a chain of transforms through the
graph and returns the ordered list of transform patterns along it. If the two formats are the same, the chain is
empty. If a format is unknown or the two formats are not connected, the lookup fails with a `TransformNotFoundError`
that says which of the two is the case.

Formats are either **hubs** or **leaves**. The bundled standard formats are hubs and may appear in the middle of a
chain. Any other format, such as a device's own flat point list, is a leaf by default: it only ever starts or ends a
chain, so adding many device formats does not multiply the number of candidate chains between the standards.

## Bundled transforms

The package ships with transform definitions among the following formats. They are loaded automatically at start-up
as configuration defaults, so a deployment only needs to supply its device mappings.

| Format name                    | Meaning                                                                                                                       |
|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| `61850`                        | IEC 61850-7-420 logical nodes and data objects (DGEN, DSTO, DECP, MMXU, ...)                                                   |
| `sunspec`                      | SunSpec Modbus models, keyed by model number (`1`, `701`, `702`, `705`, ...)                                                   |
| `1815.2.inputs`                | IEEE 1815.2 (MESA-DER / DNP3) input points, grouped as `AI` and `BI`                                                           |
| `1815.2.outputs`               | IEEE 1815.2 output points, grouped as `AO` and `BO`, plus an optional `sequence` of point batches for curve writes             |
| `2030.5`                       | IEEE 2030.5 resources keyed by resource then attribute (`DERCapability.rtgMaxW`)                                               |
| `1547`                         | IEEE 1547.1 function groups and parameter names (`Nameplate`, `Volt-VAR`, ...)                                                |
| `openfmb.ess`, `openfmb.solar` | OpenFMB v2.2 profiles for a storage or a solar device, in the protobuf JSON form that OpenFMB adapters publish                 |

Every pairing of the first six formats ships as its own file named `<input>_to_<output>.json`; the IEEE 1815.2 files
hold two definitions each, one for `1815.2.inputs` and one for `1815.2.outputs`. The two OpenFMB formats map to and
from `61850` only (`openfmb_to_61850.json` and `61850_to_openfmb.json`) and reach every other protocol through that
hub. The IEEE 1815.2 to IEC 61850 files are generated from the MESA-DER PICS; the IEEE 2030.5, IEEE 1547.1, and
direct SunSpec to IEEE 1815.2 files are derived from a single cross-reference table of the IEEE 1547.1 mapping tables
so that the two directions of each pair stay consistent.

Conventions worth knowing when reading or writing messages in these formats:

* **Curves are lists.** SunSpec curve points are the `Pt` list of the active stored curve, IEEE 2030.5 curves are
  `DERCurve.<curveType>.CurveData`, and IEC 61850 curves are the `crvPts` list of the curve object. IEEE 1815.2
  exposes curves through a single edit window (selector, type, point count, and up to 100 X,Y pairs); reading maps
  the window into a mode's curve only when the selector matches, and writing produces an ordered `sequence` of
  point batches for the driver to apply.
* **Schedules** are carried as `DERControlList` entries in `2030.5` and as `FSCH.SchdEntr` entries in `61850`, and
  map onto each other entry by entry. SunSpec and IEEE 1815.2 have no place for scheduled settings, so schedules do
  not cross into them and the field map reports the loss.
* **OpenFMB** messages are single profiles as an adapter publishes them (`essReading`, `essStatus`, `essControl`,
  `essCapability`, or the solar equivalents). Readings map to `DECP.MMXU`, ratings to `DGEN`, `DSTO`, and
  `DECP.VRef`, nameplate to `LPHD.PhyNam`, and control profiles onto `FSCH.SchdEntr`. Generated profiles lack the
  message header (`mRID`, `messageTimeStamp`), which the publishing side must add.

Anything supplied through the configuration store is applied on top of these defaults, so a site can add its own
formats or override individual rules.

## Transform expression language

Each value in a transform `pattern` is a compact expression that says where in the input message the output field
comes from and what functions to apply along the way. Patterns are parsed with pyparsing and compiled to convtools
pipelines, so they are evaluated as compiled Python rather than interpreted per message.

```
transform[<path segment>, <path segment>, ...](<function>(<args>), <function>(<args>), ...)
```

* The bracketed segments are the path into the input message. Segments may be dictionary keys or list indices. Bare
  identifiers may contain letters, digits, `_`, and `.`; a segment with any other character, such as `RegClas[1]`,
  must be quoted with single or double quotes. Unquoted digits are treated as integers.
* If the brackets are omitted, the path of output keys leading to the expression is used as the input path.
* The parenthesised functions are applied in order to the value found at that path. An empty pair of parentheses
  copies the value unchanged.
* Function arguments may be numbers (including signed, decimal, and scientific forms such as `0.001` or `-5`),
  quoted strings, bare identifiers, dotted paths (`phsA.mag`), or nested calls.

Patterns may be **nested**. A value that is an object is a group, and the output contains the same group structure.
This is how the bundled files organise fields under IEC 61850 logical nodes or SunSpec model numbers. A value of
`null` marks an output field with no known source; it is skipped.

Source fields are **optional**. If the input message does not contain an expression's source path, that field is
left out of the output rather than raising an error, and a group whose fields are all absent is left out too. A
source that is present with a value of `null` is copied through as `null`. This lets a transform written for a full
device message be applied to a partial update.

An excerpt from the bundled IEC 61850 to SunSpec transform shows the nested style:

```json
{
  "input_format": "61850",
  "output_format": "sunspec",
  "pattern": {
    "1": {
      "Mn": "transform[LPHD, PhyNam, vendor]()",
      "Md": "transform[LPHD, PhyNam, model]()",
      "SN": "transform[LPHD, PhyNam, serNum]()"
    },
    "702": {
      "WMaxRtg": "transform[DGEN, WMaxRtg]()",
      "VAMaxRtg": "transform[DGEN, VAMaxRtg]()",
      "VNomRtg": "transform[DECP, VRef]()",
      "WChaRteMaxRtg": "transform[DSTO, WChaUnPFRtg]()"
    }
  }
}
```

And an example that applies functions, reading a nested IEC 61850 style message, copying the total power through and
averaging the three phase voltage magnitudes while skipping any that are missing:

```json
{
  "W":   "transform[DECP, MMXU, TotW]()",
  "LNV": "transform[DECP, MMXU, PNV](mean(phsA.mag, phsB.mag, phsC.mag))"
}
```

### Repeated groups

An output key ending in `[#]` produces a list. The group beneath it is evaluated once per element of a source list.
Inside the group, a path segment written `Pt[#]` names the list and marks the iteration point; the segments after it
are looked up in each element. Every path in the group must name the same list.

```json
{
  "CurveData[#]": {
    "#": "transform(take('705.Crv.0.ActPt'))",
    "xvalue": "transform[705, Crv, 0, Pt[#], V]()",
    "yvalue": "transform[705, Crv, 0, Pt[#], Var]()"
  }
}
```

The reserved `"#"` entry adjusts the source. Written without a path, as above, its functions are applied to the list
the sibling paths name. Written with a path (`"#": "transform[DERCurve, opModVoltVar](as_list())"`) it supplies the
list explicitly, and sibling paths then start with the bare segment `#`, meaning the current element. Written with
functions only and no sibling naming a list (`"#": "transform(as_list())"`), it applies the functions to the
enclosing element itself, which is how a flat message is wrapped into a one-element SunSpec `Crv` list.

Paths without a marker inside a repeated group are resolved against the message root, so shared fields can be copied
into every element. Groups nest, and a path starting with `#` always refers to the element of the group it is written
in. A repeated group whose list is absent or empty is left out of the output. Several groups may feed one list by
adding a label after the marker, for example `"sequence[#] volt-var"` and `"sequence[#] volt-watt"`; their lists are
concatenated in pattern order. Mistakes such as a marker outside a repeated group or sibling paths naming different
lists are reported when the definition is compiled.

### Spread entries

A key starting with `*` (the rest of the key is a comment) holds an expression that produces a group; its fields are
merged into the enclosing group at that position. This turns a list of curve points into numbered DNP3 point indices:

```json
{
  "AO": {
    "246": "transform[DERCurve, opModVoltVar, CurveData](count())",
    "*points": "transform[DERCurve, opModVoltVar, CurveData](unpairs(249, xvalue, yvalue))"
  }
}
```

### Functions

Table: Transform functions {#transform-functions}

| Function                                                    | Effect                                                                                                                                   |
|-------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `multiple(n)`, `add(n)`                                     | Multiply by or add `n`. Have inverses.                                                                                                   |
| `scale(n)`                                                  | Multiply by `n`, rounding away floating point noise to the decimal places of the operands. Has an inverse.                               |
| `scale_int(n)`                                              | Multiply by `n` and cast to `int`. Has an inverse.                                                                                       |
| `scale_reg(path)`                                           | Divide by the register at the quoted dotted `path`; an absent register leaves the value unchanged. Has an inverse.                       |
| `scale_reg_pow_10(path)`                                    | Multiply by 10 to the power of the register at `path`, e.g. SunSpec scale factors: `scale_reg_pow_10('701.W_SF')`. Has an inverse.       |
| `no_op()`                                                   | Copy the value unchanged.                                                                                                                |
| `mod10k(reverse)`, `mod10k64(reverse)`, `mod10k48(reverse)` | Decode the ION and PM800 M10K register formats (four decimal digits per 16 bit register). Have inverses.                                 |
| `scale_decimal_int_signed(n)`                               | Scale a decimal-encoded signed register (PM800 power factor style). Has an inverse.                                                      |
| `cast_value(type_name)`                                     | Cast to `bool`, `str`, `int`, `float`, `list`, `tuple`, or `dict`. Boolean parsing accepts common truthy and falsy strings.              |
| `mean(path, ...)`                                           | Average of several dotted-path fields of the current value, ignoring missing or `null`.                                                  |
| `take(n)`                                                   | Keep the first `n` elements of a list. `n` may be a quoted dotted path to the field holding the count, e.g. `take('705.Crv.0.ActPt')`.   |
| `pairs(start, count, x, y)`, `unpairs(start, x, y)`         | Convert between a flat, position-indexed array of alternating X and Y values and a list of `{x, y}` points (DNP3 curve points).          |
| `series(count, start, spec, ...)`, `unseries(start, spec, ...)` | Convert between numbered flat driver points (`705_Crv1_Pt1_V`, ...) and a list of records.                                          |
| `scale_fields(spec, ...)`                                   | Multiply named fields of every record in a list by constants: `scale_fields('V:0.01', 'Var:0.01')`. Has an inverse.                      |
| `prefix_keys(prefix)`                                       | Rename every key of a group by prefixing it, e.g. `{"244": 1}` to `{"AO_244": 1}`. Has an inverse.                                       |
| `count()`                                                   | Number of elements in a list or group.                                                                                                   |
| `as_list()`                                                 | Wrap the value in a one-element list.                                                                                                    |
| `const(v)`                                                  | Replace the value with the literal `v` (only emitted when the source is present).                                                        |
| `when(path, v)`, `when_equal(path_a, path_b)`               | Pass the value through only when the field at `path` equals `v`, or when both fields are present and equal; otherwise treat it as missing. |
| `approx(f)`                                                 | Copy the value unchanged but record that only a fraction `f` of its meaning carries over (see [Choosing between chains](#choosing-between-chains)). |

Functions after a guard such as `when` are skipped once the value has become missing. Paths given to `take`, `when`,
`when_equal`, `scale_reg`, and `scale_reg_pow_10` are resolved against the element enclosing the expression; a path
starting with `/` is resolved from the message root wherever it appears. Functions that define an inverse are
intended to support automatic generation of reverse transforms in the future.

## Choosing between chains

When more than one chain of transforms connects two formats, the registry picks the one that preserves the most
information, not the shortest. Each registered pattern is analysed, without running it, into a **field map**: which
source fields reach which target fields, and with what **fidelity**. Fidelity is 1 for a plain copy or an invertible
function and lower for functions that lose information: `scale_int` and `take` 0.9, guards such as `when_equal` 0.7,
`mean` 0.5, `count` 0.3, and `const` 0. An author can mark an approximate mapping with `approx(f)`. Field maps compose
along a chain and fidelities multiply, so a field dropped at the first hop cannot reappear later.

A lookup enumerates every chain of up to four hops whose intermediate formats are hubs, scores each on the same set
of source fields (the fields a resource publishes if the caller supplies them, otherwise every source field any
candidate's first step reads), and keeps the chain with the highest **retention**, the fraction of source fields that
reach the target. Ties go to fewer hops. Each edge also carries an informational weight, `-ln(retention) + 0.01`,
measured against the format's full field universe, which comes from the generated model packages for SunSpec,
IEEE 1815.2, IEEE 2030.5, and OpenFMB, from the `formats` configuration for site-specific formats, or failing both
from the fields the registered transforms mention. The `score_transform` RPC exposes the chosen path and its
retention. A `lossiness` value on a transform definition overrides the measured retention, for example from an
empirical round trip.

## Device formats and discovery

A platform driver publishes a device as a flat `{point name: value}` message. That flat point list is a format of
its own: a leaf in the transform graph that maps onto one protocol and reaches the others through it. The
`interoperability.discovery` package generates the configuration for such a device format; nothing at runtime
depends on it.

**SunSpec devices.** When a registry names each point after the SunSpec path it holds (`701_W`, `704_PFWInj_PF`,
`705_Crv1_Pt3_Var`, with a 1-based index on repeating groups), the two transforms between the device format and
`sunspec` and the `formats` declaration are derived from the point names alone, validated against the SunSpec model
classes. Where the SunSpec scale factors are applied is a build-time choice, `scaling`:

| Mode        | Behavior                                                                                                                        |
|-------------|---------------------------------------------------------------------------------------------------------------------------------|
| `transform` | The driver publishes raw registers including the `_SF` points; reads apply `scale_reg_pow_10` and writes bake the discovered factor in as a constant. |
| `driver`    | The driver applies the factors through a `Transform` column in its registry; the `_SF` points are omitted and the transforms pass values through unchanged. |

The SunSpec discovery tool scans a device over Modbus TCP or RTU, or reads a pysunspec2 JSON device description
offline, and writes the whole configuration set: a registry for the `modbus` driver, a registry for the `fake`
driver with the discovered values as starting values, the device configurations for both, the service configuration,
and a `vctl` script that stores them. It requires the optional `discovery` extra, which installs pysunspec2.

```shell
pip install "interoperability-service[discovery] @ git+https://github.com/interoperability-control-modules/interoperability-service"
python -m interoperability.discovery.sunspec --host 10.0.0.21 --unit 1 --scaling driver --out build/
python -m interoperability.discovery.sunspec --file device.json --out build/   # offline
```

| Option            | Default                          | Description                                                        |
|-------------------|----------------------------------|--------------------------------------------------------------------|
| `--host`, `--port`, `--unit` | `502`, `1`            | Modbus TCP target. `--serial` selects Modbus RTU; `--file` reads a device description instead. |
| `--out`           | required                         | Directory for the generated files.                                  |
| `--scaling`       | `transform`                      | `transform` or `driver`, as above.                                  |
| `--format-name`   | `sunspec_device`                 | Name of the generated device format.                               |
| `--device-topic`  | `devices/site1/feeder1/inverter` | Topic the driver publishes the device on.                          |
| `--uai`           | `site1/inverter`                 | UAI of the canonical resource, `/`-separated.                      |

**IEEE 1815.2 devices.** The same machinery serves an outstation behind the `dnp3` driver, with point names
`<table>_<index>` (`AI_297`, `AO_217`, `BI_93`) validated against the service's IEEE 1815.2 point registry. The
device format maps onto `1815.2.inputs` and `1815.2.outputs`, and back from `1815.2.outputs` only, since inputs are
read-only on a device. DNP3 has no self-description, so the discovery tool takes the point set from a test-tool
profile file or any index listing and writes the same configuration set:

```shell
python -m interoperability.discovery.dnp3 --profile mandatory_1547.json --out build/ \
    --outstation-ip 10.0.0.22 --port 20000
```

It accepts `--scaling`, `--format-name` (default `dnp3_device`), `--device-topic`, and `--uai` as the SunSpec tool
does. Multipliers and offsets come from the profile, so both scaling modes work without device discovery; the current
`dnp3` driver applies neither, so `transform` is the mode that works today.

**IEEE 2030.5.** There is no register device to discover: the VOLTTRON 2030.5 agent mirrors a driver device to a
2030.5 server through a point map of `Object::property` rows. `interoperability.discovery.point_maps` generates that
map for any device format that reaches `2030.5` from the composed field map of the chosen chain. Messages from a
2030.5 source are already in the service's `2030.5` format and need no device format.

## Interface

The service runs under the VIP identity `platform.presentation` and exposes the following interface on the message bus.

| Method / topic                                                                           | Type   | Returns               | Description                                                                                                                                                                                                                                                                                           |
|------------------------------------------------------------------------------------------|--------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `resolve(uai, as_format=None, strict=False)`                                             | RPC    | dict                  | Resolves a UAI to its canonical resource. When `as_format` is given, the result also includes `target_format` and `transform`, the ordered list of transform patterns from the resource's `data_format` to `as_format` (empty when they are the same). Returns `{}` if nothing canonical is found and fails with `TransformNotFoundError` if the resource cannot be converted. |
| `lookup_transform(input_format, output_format)`                                          | RPC    | list of pattern dicts | The transform chain between two formats. An empty list means no transform is needed. When no chain exists the call fails with `TransformNotFoundError`, saying whether a format is unknown or the formats are not connected.                                                                        |
| `score_transform(input_format, output_format, fields=None)`                              | RPC    | dict                  | The chain `lookup_transform` would choose, as `path` (the formats passed through) and `retention` (the fraction of source fields that reach the end), scored over `fields` if given.                                                                                                                 |
| `register_transform(input_format, output_format, pattern, update=False, lossiness=None)` | RPC    | bool                  | Adds a transform edge at runtime. If the pair already has a different transform it is kept, with a warning, unless `update` is true; registering the same pattern again is a no-op. Malformed patterns are rejected. Returns whether the registry changed.                                            |
| `mapper/update`                                                                          | PubSub | none                  | Publish a list of mapping objects (same structure as the `mappings` configuration below) to add resources to the UAI tree at runtime.                                                                                                                                                                 |

## Configuration

The service reads a single `config` entry from the configuration store with two top-level lists, `mappings` and
`transforms`, and an optional `formats` object. Because the bundled transforms are loaded as defaults, a typical
configuration contains only mappings:

```json
{
  "mappings": [
    {
      "uai": ["site1", "feeder2", "pv_inverter"],
      "resource_type": "canonical",
      "resource": {
        "data_format": "sunspec",
        "owner": "platform.driver",
        "publication_topic": "devices/site1/feeder2/pv_inverter/all",
        "rpc_topic": "devices/site1/feeder2/pv_inverter"
      }
    },
    {
      "uai": ["site1", "pv_61850"],
      "resource_type": "alias",
      "resource": {
        "data_format": "61850",
        "owner": "platform.driver",
        "references": ["site1", "feeder2", "pv_inverter"]
      }
    }
  ],
  "formats": {
    "acme_inverter": { "hub": false, "fields": ["W", "V.PhaseA", "V.PhaseB", "V.PhaseC"] }
  },
  "transforms": []
}
```

In this example, an agent that resolves the alias `["site1", "pv_61850"]` receives the canonical SunSpec resource for
the inverter together with the bundled `sunspec` to `61850` transform, without needing to know which protocol the
device implements.

Each entry in `mappings`:

| Key             | Required | Type           | Description                                                                |
|-----------------|----------|----------------|----------------------------------------------------------------------------|
| `uai`           | yes      | list or string | The identifier being defined. A string is parsed as a JSON-encoded tuple.  |
| `resource_type` | yes      | string         | `canonical` (or `canon`) or `alias` (or `aliased`).                        |
| `resource`      | yes      | object         | The resource fields listed in the Resources table above.                   |

Each entry in `transforms`:

| Key             | Required | Type   | Description                                                                                                  |
|-----------------|----------|--------|--------------------------------------------------------------------------------------------------------------|
| `input_format`  | yes      | string | Format name of the incoming message.                                                                         |
| `output_format` | yes      | string | Format name of the produced message.                                                                         |
| `pattern`       | yes      | object | Output field name to transform expression, possibly nested.                                                  |
| `lossiness`     | no       | number | Overrides the measured loss of this transform, 0 (lossless) to 1, for example from an empirical round trip.  |

Each entry in `formats` is keyed by format name:

| Key      | Type            | Description                                                                                                        |
|----------|-----------------|--------------------------------------------------------------------------------------------------------------------|
| `hub`    | boolean         | Whether transform chains may pass through this format. Bundled standard formats are hubs; anything else is a leaf by default. |
| `fields` | list of strings | The fields the format can carry, as dotted paths with `*` for repeating groups. Used to measure how much of the format a transform covers. |

Configuration definitions are authoritative: a transform for a pair that already exists replaces the earlier one, and
a pair defined twice in one configuration keeps the first and logs a warning. Store the configuration with:

```shell
vctl config store platform.presentation config path/to/config.json
```

## Consuming a resource from another agent

`interoperability.resource.ResourceData` is a client-side helper used by the
[Message Bus Adapter](message-bus-adapter.md) and available to any agent. It resolves a UAI through the service, compiles the returned transform, subscribes to the
canonical publication topic, and delivers transformed payloads to a callback under the caller's own local topic:

```python
from interoperability.resource import ResourceData

def on_data(peer, sender, bus, topic, headers, message):
    ...  # message is already in the requested format

resource = ResourceData.lookup(self, ("site1", "feeder2", "pv_inverter"))
if resource:
    resource.subscribe(on_data)
```

`lookup` accepts a tuple, a list, or a delimited string (default delimiter `/`). It returns `None` when the UAI does
not resolve to a canonical resource, and also when it resolves but no transform can convert the resource into the
requested format; the reason is logged as a warning. A resource that is already in the requested format is delivered
unchanged.

## OpenFMB and standards data models

The package includes generated pydantic models for the OpenFMB v2.2 protobuf modules (breaker, capacitor bank,
circuit segment service, environment, ESS, EVSE, generation, interconnection, load, meter, recloser, regulator,
reserve, resource, solar, switch, and common types). A `PROFILES` dictionary maps the 67 OpenFMB profile names to
their classes. The classes follow protobuf's JSON mapping, which is what OpenFMB adapters publish, and reject unknown
fields so that a message of the wrong shape fails validation rather than silently matching nothing. Hand-written
profile builders construct complete DER profiles. These models back the `openfmb.ess` and `openfmb.solar` transform
formats and are used with the [Message Bus Adapters](message-bus-adapter.md) for [OpenFMB Integration](openfmb.md).

Sibling packages provide the same treatment for SunSpec (one module per model id), IEEE 1815.2 (point enumerations
and function-group profiles from the IEEE 1815.2 test tool), and IEEE 2030.5 (all schema types of `sep.xsd`). They
supply the field universes used when scoring transform chains and validate point names during device discovery.

## Integration with Control Applications and OpenFMB

Publish/subscribe messaging provides normalized device data to control applications, which consume the published
data structures and issue control commands back through the framework, enabling coordinated actuation of multiple
heterogeneous DER/ESS devices. Message bus adapters connect the framework bus to OpenFMB buses over NATS and MQTT,
while device drivers provide direct point-to-point communication with controllers and devices. See the
[OpenFMB Integration](openfmb.md) page for the integration architecture.

## Status and known limitations

The service is at an early stage. Current limitations:

* **Transform weighting is measured from the patterns alone.** It does not know when a mapping is semantically
  approximate unless the author marks it with `approx()`, and there is no empirical round-trip check yet; the
  `lossiness` override exists so that one can be applied when it is written.
* **Inverse transforms** are attached to most functions but reverse pipelines are not yet generated automatically.
* **No default mappings ship with the package**, so UAIs must be supplied through the configuration store, the
  `mapper/update` topic, or the discovery tools.
* **Agent-level tests** of the RPC and pubsub behavior are not yet in place.

## Requirements

* Python >= 3.10
* Modular Eclipse VOLTTRON (`volttron-core` >= 2.0.0rc30) or the
  [der-control-fastlib runtime](der-control-fastlib.md). The agent falls back to the monolithic `volttron.platform`
  imports if `volttron-core` is not installed.
* Runtime libraries, installed with the package: `convtools`, `networkx`, `pydantic` 2, `pyparsing`, `treelib`
* Optional `discovery` extra for the SunSpec discovery tool: `pysunspec2`

## Installation

Before installing, VOLTTRON should be installed and running and its virtual environment should be active.
Information on how to install the VOLTTRON platform can be found
[here](https://github.com/eclipse-volttron/volttron-core).

```shell
git clone https://github.com/interoperability-control-modules/interoperability-service
vctl install ./interoperability-service --vip-identity platform.presentation --tag interop --start
vctl config store platform.presentation config path/to/config.json
vctl status
```

Other agents look the service up by the identity `platform.presentation`, so keep that identity unless the callers are
also changed. For deployment on the der-control-fastlib runtime, including as a container, see
[Deployment](deployment.md).
