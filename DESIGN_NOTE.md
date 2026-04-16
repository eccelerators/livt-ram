# Design Note

This note records the project architecture and the compiler observations
discovered while building the RAM component. It has two purposes:

- explain why the Livt source and VHDL source live side by side
- provide concrete feedback for compiler development so generated VHDL/HDL can
  become more correct, predictable, and efficient over time

The compiler observations below are based on the generated files under `out/`
and on behavior observed while running the project test suite.

## Side-by-Side Livt and VHDL

This project intentionally keeps the high-level RAM API in Livt and the storage
primitive in VHDL.

`Ram` in [src/Ram.lvt](src/Ram.lvt) is the user-facing component. It exposes
`ReadByte` and `WriteByte`, manages the write-enable signal, and keeps callers
away from the lower-level RAM ports.

`InternalRam` in [src/InternalRam.lvt](src/InternalRam.lvt) is the Livt-facing
contract for the VHDL primitive. It is annotated with `@Opaque`, so Livt
generates the package/interface information but does not generate the component
body.

The implementation body lives in [src/InternalRam.vhd](src/InternalRam.vhd).
This is useful for a sample project because it shows how Livt can wrap an
existing HDL block while still presenting a compact, idiomatic Livt API to the
rest of the design.

The hand-written VHDL currently follows two Livt integration conventions:

- `use work.livt_lang_icontext_package.t_icontext_in;` must be imported manually.
  The surrounding `---` marker lines are human-facing markers showing where the
  VHDL was manually adjusted to fit Livt-generated VHDL.
- VHDL ports that correspond to Livt opaque-component fields currently need the
  `ctor_` prefix, for example `ctor_address` and `ctor_read_data`.

## RAM Geometry

`InternalRam` derives its size at VHDL elaboration time from the port widths:

```vhdl
type T_Ram is array (0 to 2 ** ctor_address'length - 1)
    of std_logic_vector(ctor_write_data'range);
```

With `address: logic[8]` and `write_data: logic[8]`, the result is 256 cells of
8-bit data. Changing either port width changes the memory geometry without
rewriting the VHDL array type.

## Read and Write Timing

Writes happen on the rising clock edge when the generated `ctor_write_enable`
port is asserted. The Livt wrapper sets the target address, write data, and
write enable together, then clears write enable after the write state.

Reads use a registered address inside `InternalRam` and then expose the selected
cell combinatorially:

```vhdl
read_address <= ctor_address;
ctor_read_data <= internal_ram(to_integer(unsigned(read_address)));
```

This gives the VHDL primitive one-cycle read latency. The generated `ReadByte`
state machine currently waits additional cycles before sampling `read_data`;
those cycles are conservative and harmless for the tests.

## Generated Files to Inspect

| File | Role |
| ---- | ---- |
| `out/debug/main/Livt.IO.Ram.vhd` | Generated component VHDL; contains address conversion and `ReadByte`/`WriteByte` state machines |
| `out/debug/main/InternalRam.vhd` | Copied hand-written VHDL RAM implementation |
| `out/debug/main/Livt.IO.Ram.Package.vhd` | Record types for `ReadByte`/`WriteByte` function I/O |
| `out/debug/tests/Livt.IO.Tests.RamTest.vhd` | Generated test harness |

## Current Compiler Observations

The implementation currently keeps `ReadByte` and `WriteByte` address parameters
as `int`, even though the `InternalRam` port is `logic[8]`.

This is a deliberate workaround for a compiler issue with `logic[8]` parameters
at integer literal call sites. When the public parameter is declared as
`address: logic[8]`, the generated VHDL in the test harness does not always
preserve the target width:

| Livt call site | Generated VHDL | Expected VHDL |
| -------------- | -------------- | ------------- |
| `ram.ReadByte(0)` | `... .address <= '0';` | `... .address <= x"00";` |
| `ram.ReadByte(64)` | `... .address <= "1000000";` | `... .address <= x"40";` |

For `0`, the compiler emits a one-bit character literal. For `64`, it emits a
7-bit binary string without the leading zero. Both cases fail VHDL analysis due
to width mismatches.

Keeping `address: int` avoids that code-generation issue. The generated RAM
component then converts the integer to the internal `logic[8]` address via
`to_signed`:

```vhdl
this_readbyte_ram_address <= std_logic_vector(
    to_signed(this_readbyte_in.address, this_readbyte_ram_address'LENGTH));
```

For addresses greater than 127, this conversion emits
`NUMERIC_STD.TO_SIGNED: vector truncated` warnings during simulation. The
truncated bit patterns still address the intended memory cells because
`InternalRam` indexes with `unsigned(ctor_address)`. The current tests therefore
pass, but the warnings are expected until the compiler can generate better
address conversions.

## Compiler Improvement Opportunities

1. When an integer literal is assigned to a `logic[N]` parameter, emit an
   `N`-bit vector literal such as `x"00"` or `x"40"` rather than a character
   literal or an unsized binary string.
2. When an `int` is converted to an unsigned bit vector such as an address,
   prefer `to_unsigned` over `to_signed` to avoid truncation warnings for values
   in the upper half of the unsigned range.
