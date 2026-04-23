# RAM Component for Livt

This package provides a small byte-addressable RAM component in Livt. It uses a
Livt wrapper for the public API and a hand-written VHDL implementation for the
storage primitive.

The separate [DESIGN_NOTE.md](DESIGN_NOTE.md) records the Livt/VHDL boundary and
the current compiler observations that shape the implementation.

## 📋 Overview

The current package is organized around two implementation components:

- `Ram` exposes a compact Livt API for byte reads and writes.
- `InternalRam` defines the Livt-facing port contract for the hand-written VHDL
  RAM primitive.

The RAM is currently fixed to:

- 2048 addressable cells
- 8-bit `logic[8]` data values
- 11-bit internal address width

The public API is intentionally small and centered on byte-level random access:

- `WriteByte()` writes one byte to an address.
- `ReadByte()` reads one byte from an address.

## 📁 Project Structure

```text
.
├── src/
│   ├── InternalRam.lvt
│   ├── InternalRam.vhd
│   └── Ram.lvt
├── tests/
│   └── RamTest.lvt
├── DESIGN_NOTE.md
├── LICENSE
├── README.md
└── livt.toml
```

## 🔨 Building

Build the package with:

```bash
livt build
```

The package configuration is defined in [`livt.toml`](livt.toml). The current
project name there is `Ram`.

## 🧪 Running Tests

Run the full test suite with:

```bash
livt test
```

Configured test components:

- `RamTest`

The test suite may emit `NUMERIC_STD.TO_SIGNED: vector truncated` warnings for
addresses in the upper half of the 11-bit address range (≥ 1024). Those warnings
are documented in [DESIGN_NOTE.md](DESIGN_NOTE.md).

## 📚 Component Guide

### `Ram`

Small byte-addressable RAM wrapper in the `Livt.IO` namespace.

Features:

- byte-level random access
- synchronous writes through the VHDL storage primitive
- registered-address reads through the VHDL storage primitive
- automatic write-enable handling in the Livt wrapper

Public methods:

- `WriteByte(address: int, value: logic[8])`
- `ReadByte(address: int) logic[8]`

`address` is currently typed as `int` in the public Livt API as a compiler
workaround. The internal RAM port remains `logic[11]`.

### `InternalRam`

Opaque Livt component backed by [src/InternalRam.vhd](src/InternalRam.vhd).

Public ports:

- `write_enable: in logic[1]`
- `address: in logic[11]`
- `write_data: in logic[8]`
- `read_data: out logic[8]`

## 💡 Example

```livt
using Livt.IO

component Example
{
    ram: Ram

    new()
    {
        this.ram = new Ram()
    }

    public fn StoreAndLoad(address: int, value: logic[8]) logic[8]
    {
        this.ram.WriteByte(address, value)
        return this.ram.ReadByte(address)
    }
}
```

## 🔧 Configuration

This package does not currently expose configurable width or depth parameters.

To change the RAM shape today, update the fixed Livt interface in:

- [`src/InternalRam.lvt`](src/InternalRam.lvt)

The VHDL implementation derives its memory geometry from the port widths in:

- [`src/InternalRam.vhd`](src/InternalRam.vhd)

If the RAM contract changes, the expected behavior should also be updated in:

- [`tests/RamTest.lvt`](tests/RamTest.lvt)

## 📝 Notes

- The package intentionally demonstrates side-by-side Livt and VHDL: Livt owns
  the user-facing component API, while VHDL owns the low-level RAM primitive.
- `InternalRam` is marked `@Opaque`, so the compiler treats the Livt file as the
  component contract and uses the matching VHDL entity for implementation.
- The current address type is a deliberate workaround for compiler literal-width
  handling; see [DESIGN_NOTE.md](DESIGN_NOTE.md).

## 🤝 Contributing

Contributions are welcome. Areas that would be natural extensions for this
package include:

- configurable RAM depth
- configurable data width
- unsigned address parameters once compiler support is available
- word-level 16-bit or 32-bit read/write helpers
- dual-port RAM support
- memory initialization from a file

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
