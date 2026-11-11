# 8K MXF validation record

Date: 2026-10-31. Host: Linux x86_64, GCC 13.3.0, OpenSSL 3.0.13.
Upstream: https://github.com/cinecert/asdcplib.git
Baseline: 905d6047f67c8211c2d22f29fb0e2e44c142b59b (2.13.3).
Branch: feature/8k-jpeg2000-mxf. Existing workspace clones were not modified.

## Observed results

- Full Release build: passed, including the existing command-line tools.
- Fresh CTest run: 1/1 suite passed in 30.12 s; suite contains 81 checks.
- Full 8192x4320: 24/25/30/48/50/60 fps checked.
- Flat 7992x4320, Scope 8192x3432, UHD 7680x4320: checked.
- Three unsigned 12-bit components; seven DWT decompositions in generated 8K samples.
- Two-frame sequence with a final frame above the legacy 4 MiB buffer size: checked.
- Stored dimensions, Rsiz=0 and registered Undefined Cinema UL: checked.
- Plaintext wrap/unwrap: SHA-256 identity of every codestream checked.
- Encrypted essence: decrypt + HMAC verify + SHA-256 identity checked.
- Encryption with clear codestream headers (-E), including >255-byte headers: checked.
- Extracted full 8K image decoded by OpenJPEG: passed.
- Missing opt-in, stereo, conflicting UL override, disabled parameter checking,
  fractional/unknown rate, false 4K profile, oversized raster and bad bit depth: rejected.
- Actual 2K and 4K cinema-profile samples: established profile metadata and
  byte-exact wrap/unwrap regressions passed.
- Direct C++ API: legacy over-4K rejection, Interop rejection, valid 8K writing,
  mismatched frame rejection without SequenceParser, two-frame final duration,
  reverse-order random-access byte identity, and legacy parser literal-zero
  source compatibility: passed.
- git diff --check: passed before final packaging.

## Reproduce

See [8K-JPEG2000-MXF.md](8K-JPEG2000-MXF.md) for accepted inputs and limitations.


```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DASDCP_BUILD_8K_TESTS=ON
cmake --build build -j8
ctest --test-dir build --output-on-failure
# Independent API test (use a fresh disposable output filename):
build/test-8k-api /path/to/8192x4320-Rsiz0-12bit.j2c /tmp/api-8k-output.mxf
```

Local evidence: build/ctest-8k-final.log, build/test-8k-v3.log,
build/test-8k-v3/results.json, and the generated MXF/extracted codestreams.
These generated test files are not committed.

## Limits of this evidence

Generated sources are black pictures. The >4 MiB sample uses legal COM markers
after SIZ; this stresses buffers, wide plaintext offsets, encryption, and index
handling, not high-entropy decoder throughput. No projector, cinema server,
independent MXF implementation, complete DCP, or DCI certification was tested.
No normative 8K profile or new UL/Rsiz allocation is claimed.
