# Experimental 8K JPEG 2000 MXF wrapping

Upstream baseline: cinecert/asdcplib, commit

`905d6047f67c8211c2d22f29fb0e2e44c142b59b` (2.13.3).

This extension produces a **picture Track File**, not a complete DCP.
It does not create a CPL/PKL/AssetMap, certify ST 429-2 compliance, or prove
that a cinema server/media block can play the result. No new SMPTE 8K
Cinema Profile, Rsiz value, or UL is invented.

## Build and use


```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j8
build/src/asdcp-wrap -8 -p 24 /path/to/j2c-sequence picture-8k.mxf
build/src/asdcp-info -d -c -r picture-8k.mxf
mkdir extracted
build/src/asdcp-unwrap -b 67108864 picture-8k.mxf extracted/frame
```

The directory contains alphabetically ordered **raw JPEG 2000 codestreams**
(e.g. 000001.j2c), not JP2 container files, RGB source pictures or video.
The library wraps without transcoding. Input component values must already
represent cinema X/Y/Z; the wrapper cannot infer or convert the source color
space from the three components alone.

`-8` is explicit opt-in, implies SMPTE labels (`-L`), and uses a 64 MiB
input frame buffer unless `-b bytes` is supplied. Larger input requires a
larger buffer. The unwrap tool still has the upstream 4 MiB default; use
`-b` for large frames. No system-wide installation is required.

Accepted by this packaging extension:

* Even dimensions, above the 4096x2160 bounds in at least one dimension,
  up to 8192x4320: includes Full 8192x4320, Flat 7992x4320,
  Scope 8192x3432 and UHD 7680x4320.
* Zero image/tile origin, one full-image tile.
* Three unsigned 12-bit, full-resolution components.
* Part 1 codestream with **Rsiz=0**, without CAP/PRF/CPF signaling.
* 24/25/30/48/50/60 fps; no fractional or arbitrary frame rates.
* Monoscopic only; `-3`, `-P`, and `-Z` cannot be combined with `-8`.

Accepting UHD and other bounded even rasters is a packaging feature, not a
change to the earlier proposed movie 8K raster table. This implementation
is **not a complete validator of that proposal**: it does not enforce
7 DWT levels, CPRL, code-block/precinct rules, image quality, or its
1/2 Gbit/s limits. It can package already-encoded inputs meeting the
checks above. HTJ2K and IMF-profile signaling are not included.

## Metadata and API

The registered **JPEG 2000 Undefined Digital Cinema Profile** label is used:

`urn:smpte:ul:060e2b34.0401010a.04010202.0301017f`

Source: [SMPTE published Labels Register](https://registry.smpte-ra.org/view/published/labels_view.html),
entry `JPEG2000UndefinedDigitalCinemaProfile` (LEAF, not deprecated).
This is an existing undefined-profile label, **not a standardized 8K label**.
The original SIZ Rsiz is retained in the JPEG 2000 MXF subdescriptor.
The File Package name explicitly describes experimental wrapping.

`ASDCP::JP2K::MXFWriter::OpenWrite8K(filename, info, descriptor, headerSize)`
is the new library entry point. Set `info.LabelSetType = LS_MXF_SMPTE`.
Obtain the descriptor from the codestream/sequence parser and set its
EditRate before opening. Continue with the existing WriteFrame/Finalize
API (including optional AES/HMAC contexts). Every written frame is checked
against the source essence descriptor before writing its KLV packet.
The public PictureDescriptor layout and existing OpenWrite signature are
unchanged. A separate ParseMetadataIntoDescWide API preserves 32-bit header
offsets for encrypted clear-header mode; the legacy byte-offset API remains
available and reports an error rather than truncating an unrepresentable offset. Existing binaries must relink to use the additional entry point.

The established OpenWrite path now rejects dimensions beyond 4K instead
of silently assigning the 4K UL and Rsiz=4. Existing valid 2K/4K paths
retain their behavior. The stereo path also rejects dimensions beyond 4K.

`asdcp-info` recognizes the undefined-profile label and warns that it does
not imply ST 429-2 conformity. It does not apply the standard 250 Mbit/s
cinema threshold to undefined-profile essence; an explicit `-t` remains
available. Its index-based bitrate statistic now handles one/two-frame
tracks and includes the first and final frame. For encrypted tracks,
the reported number includes encryption packet overhead and is not a
codestream-only conformance measurement.

## Reproducible tests

Requires Python 3 and OpenJPEG `opj_compress`/`opj_decompress` on PATH.

```sh
python3 tests/test_8k_mxf.py --bin-dir build/src
# Or configure the optional CTest integration:
cmake -S . -B build -DASDCP_BUILD_8K_TESTS=ON
cmake --build build -j8
ctest --test-dir build --output-on-failure
```

Tests encode real 12-bit codestreams locally, wrap and unwrap them, compare
SHA-256 hashes, decode an extracted full 8K image, inspect dimensions and
Rsiz/UL, check encrypted/HMAC roundtrip, and exercise negative inputs plus
2K/4K regression. The >4 MiB fixture uses legal COM segments in a real
codestream; it tests buffer/index correctness, not noisy-image decode
throughput or visual quality. Generated artifacts use a temporary directory
unless --work-dir is supplied. Use a new empty work directory for each run.

The optional CMake configuration also builds test-8k-api; run it with a
valid 8K codestream and a fresh disposable MXF output filename for direct
library API rejection and random-access readback checks. See
[8K-VALIDATION.md](8K-VALIDATION.md) for observed validation results.

Before production: test target playback hardware, non-black/noisy imagery,
long sequences, reel boundaries, independent MXF tooling, and any further
codec/profile signaling. This patch does not register or standardize an
8K DCP application profile.
