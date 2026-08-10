# SPT_LoadBundleEvenFaster (SPT 4.1 port)

Speeds up SPT's bundle loading by validating bundle checksums in parallel instead of one at a time.

This is a fork of [s8ga/SPT_LoadBundleEvenFaster](https://github.com/s8ga/SPT_LoadBundleEvenFaster),
ported to **SPT 4.1.2** and carrying one behaviour change described below. All of the original design
and code is s8ga's.

## What this fork changes

**1. CRC concurrency is configurable instead of capped at 8 threads.**

Upstream fixed concurrency with a hardcoded expression:

```csharp
private static readonly int MAX_CONCURRENT_CRC = Environment.ProcessorCount >= 8 ? 8 : Environment.ProcessorCount;
```

On a machine with more than 8 logical cores that leaves the rest idle. This fork exposes the value as
a setting, editable in the BepInEx F12 config manager:

| Setting | Section | Default | Range |
|---|---|---|---|
| `MaxConcurrentCrcThreads` | `Performance` | `0` | 0 - 256 |

`0` means "use every logical core". Any other value is clamped to the real core count, since asking
for more threads than cores only adds contention. Lower it if you get stuttering during load.

**2. `Crc32` is fully qualified at its call site.**

The EFT client assembly declares its own `Crc32` type in the global namespace. C# name lookup prefers
a global-namespace type over one brought in by a `using`, so `using SPT.Custom.Utils` silently lost
and the build failed with `CS0117: 'Crc32' does not contain a definition for 'Update'`. Qualifying
the call as `SPT.Custom.Utils.Crc32.Update(...)` resolves it.

## Requirements

- **SPT 4.1.2.** The plugin declares a hard dependency on `com.SPT.custom` (minimum 4.0.0), which
  SPT 4.1.2 satisfies.
- Optional: [SPT_PatchCrc32](https://github.com/Dildz/SPT_PatchCrc32) for hardware-accelerated CRC32.
  It is a **soft** dependency, detected at runtime. Without it the plugin falls back to SPT's managed
  CRC32 implementation and still works.

## Installation

Grab the zip from the [Releases page](https://github.com/Dildz/SPT_LoadBundleEvenFaster/releases)
and extract it into your game folder, or copy the plugin DLL there yourself, so that you end up with:

```
BepInEx/plugins/s8_SPT_LoadBundleEvenFaster/SPT_LoadBundleEvenFaster.Plugin.dll
```

That single DLL is the whole plugin. **Do not copy the rest of the build output** - MSBuild places a
copy of every reference assembly next to it, including the game's own `Assembly-CSharp.dll`, and
loading duplicates of those would break BepInEx.

On first run the plugin writes `BepInEx/config/com.s8.sptloadbundleevenfaster.cfg`. A successful
start looks like this in the BepInEx log, with your own core count:

```
MaxConcurrentCrc set to 10 (configured: 0, CPU cores: 10)
[Performance] Hooked into 'com.s8.sptpatchcrc32' native accelerator successfully!
ValidateBundlesStreamingAsync: Validation completed. Valid: 25/25
Init_Prefix: Validation succeeded! Fast path enabled. Bypassing SPT serial hash checks.
```

The second line only appears if [SPT_PatchCrc32](https://github.com/Dildz/SPT_PatchCrc32) is also
installed. Without it you get `[Performance Tip] Native CRC32 accelerator not found!` instead, and
validation still works using SPT's managed implementation.

## Building

`References/Client/` is gitignored, so you must populate it before building. It needs 20 assemblies
taken from three folders of an SPT install:

- `EscapeFromTarkov_Data/Managed` - the EFT and Unity assemblies
- `BepInEx/core` - `BepInEx.dll`, `0Harmony.dll`
- `BepInEx/plugins/spt` - `spt-common`, `spt-core`, `spt-custom`, `spt-reflection`

On Windows, upstream's `CopyReferences.ps1 -SptPath "C:\Path\To\SPTarkov"` does this for you. On
Linux, copy them by hand or adapt the script.

> **Watch the SPT version of the `spt-*` assemblies.** They are **not** interchangeable across patch
> releases: all six `spt-*` client DLLs differ between SPT 4.1.1 and 4.1.2. Take them from the same
> SPT release you intend to run. The EFT and Unity assemblies are unaffected, and `BepInEx/core` is
> identical between those two releases.

Then:

```bash
dotnet build SPT_LoadBundleEvenFaster.Plugin/SPT_LoadBundleEvenFaster.Plugin.csproj -c Release
```

`Package.ps1` builds a distributable `.7z` with the correct BepInEx folder layout, but it is
PowerShell and Windows only. For local testing, copying the single output DLL into place is enough.

## Credits

All original design and implementation by **s8ga**. See the
[upstream repository](https://github.com/s8ga/SPT_LoadBundleEvenFaster) for the demo video and the
original notes.

Built for [SPT](https://sp-tarkov.com), which turns Escape From Tarkov into an offline
single-player experience.

## License

MIT, as inherited from upstream. See [LICENSE](LICENSE).
