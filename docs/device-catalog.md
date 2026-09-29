# Device-type identification

An access point learns a joining device's numeric identity from two
fields in the Inclusion Request frame (see `security-and-inclusion.md`):
a 2-byte **manufacturer code** and a 4-byte **manufacturer device type**.
On their own, these are just numbers — turning `(1, 297)` into a
human-readable model name like "wall thermostat" requires a lookup
table, and that table is not something this specification reproduces
wholesale (see below for why), but it is trivial for any implementer to
obtain their own copy of.

## Where the mapping data actually lives

The vendor's own host-side server software ships a device-specification
catalog as plain, uncompiled XML resource files bundled inside its
distributed `.jar` archives (a `.jar` is an ordinary ZIP archive — no
decompilation, disassembly, or reverse engineering of any executable
code is involved in reading a plain-text resource file out of one, any
more than it would be for any other file in any other ZIP archive).
Inside the server-side `.jar`, the relevant resource path is:

```
de/eq3/cbcs/devicedescription/devicespecification/
├── eQ-3/                 (one small XML file per device family, e.g. device_swdo.xml)
│   ├── device_swdo.xml
│   ├── device_bwth.xml
│   └── ... (300+ files)
└── HunterDouglas/        (third-party-branded shutter/blind devices, same file shape)
```

Each file contains one or more entries of roughly this shape:

```xml
<devType label="<model name>" id="<device type, decimal>"
         manufacturerCode="<manufacturer code, decimal>" ... />
```

Extracting it is a single command against your own, lawfully-obtained
copy of the `.jar` (any ZIP tool works; `jar` is the one that ships with
a JDK):

```sh
jar xf HMIPServer.jar de/eq3/cbcs/devicedescription/devicespecification
# or, equivalently, since a .jar is just a ZIP archive:
unzip HMIPServer.jar 'de/eq3/cbcs/devicedescription/devicespecification/*'
```

If you have lawful access to a copy of that software (for example, as
part of your own HomeMatic/HomematicIP system installation), you can
extract this catalog yourself with any standard ZIP/archive tool and
get the complete, authoritative, currently-up-to-date mapping directly
— which will always be more complete and more current than anything
reproduced here. This specification does not include a full copy of
that catalog: republishing a vendor's own compiled device/product
catalog wholesale is a different question from independently describing
observed protocol behavior (the latter is this project's actual scope;
data-compilation rights are a separate legal area this project makes no
claims about one way or the other). What follows instead is a small,
explicitly-scoped table of entries this project has itself verified
against real, physically-paired hardware — not an extract of the
vendor's catalog.

## Verified table

Confirmed by observing the actual `(manufacturerCode, deviceType)` pair
reported in a real Inclusion Request frame from real, physically paired
hardware during the development of this specification — **verified on
the wire**, not looked up from the catalog file described above:

| Manufacturer code | Device type | Model | Hardware designator |
|---|---|---|---|
| 1 | 258 | Window/door sensor | `HMIP-SWDO` |
| 1 | 295 | Radiator (valve) thermostat | `HmIP-eTRV-2` |
| 1 | 297 | Wall-mounted thermostat | `HmIP-WTH-2` |

The **hardware designator** is the exact model string printed on the
physical device — it matches the catalog XML's own `label` attribute
character-for-character, including eQ-3's own inconsistent capitalization
between products (note `HMIP-SWDO` vs. `HmIP-`-prefixed names above; this
isn't a typo in this table, it's a genuine inconsistency in the vendor's
own labeling).

This table will grow as more device types are verified against real
hardware by this project or its contributors — see `CONTRIBUTING.md`
if you can add to it from your own devices. It is deliberately not a
substitute for the full catalog described above.
