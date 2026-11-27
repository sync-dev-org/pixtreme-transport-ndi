# pixtreme-transport-ndi

NDI® video and audio input and output for [pixtreme.transport](https://github.com/sync-dev-org/pixtreme-transport).
NDI® is a registered trademark of Vizrt NDI AB. See [NDI](https://ndi.video/).

Receive NDI sources into GPU frames and send processed frames from a transport
core `FrameBuffer` to NDI receivers. The package provides Python `Input` and
`Output` factories, a C++ shared library, public headers, a CMake package, and
a bundled libndi runtime.

## Installation

```bash
pip install pixtreme-transport-ndi
```

Supported wheels target Linux x86_64 (including WSL2) and native Windows x86_64,
with CPython 3.12 and 3.13. Linux wheels use `manylinux_2_28_x86_64` (glibc 2.28
or newer); Windows wheels use `win_amd64`. Processing requires an NVIDIA
GPU and a driver that supports CUDA 13. Pixel serialization calls pixtreme's
conversions, built for sm 86, sm 89, and sm 120. The wheel requires no CUDA Toolkit or NDI SDK
installation.

On Linux, Avahi's `libavahi-client.so.3`, `libavahi-common.so.3`, and their dependencies
must be installed on the host. Install `libavahi-client3` on Debian/Ubuntu or
`avahi-libs` on RHEL-family systems. NDI mDNS discovery also needs a running
`avahi-daemon`. Import errors identify a missing Avahi library and the packages
to install.

Windows requires Microsoft Visual C++ v14 Redistributable x64 for MSVC toolset
14.44 or later. The wheel bundles `Processing.NDI.Lib.x64.dll` and its
`Processing.NDI.Lib.Licenses.txt` notice under the package's `_ndi/` directory.
The package registers its own `bin/` and `_ndi/` with `os.add_dll_directory`
before importing the native binding; pixtreme and the core register their own
DLL directories. Their DLLs, CUDA runtime, NVIDIA driver and C runtime DLLs stay
outside this wheel. Use Linux wheels inside WSL2.

The dependency `pixtreme-transport>=0.53.0` installs the transport core, which
supplies frame types, buffers, worker threads, audio processing, and pixel
serialization. This module directly depends on `pixtreme>=2.31.1` and uses
the core's serialization to convert picture colours to the NDI standard
for their resolution. The core's native ABI must
match the ABI used to build this module;
import rejects an incompatible core before loading the NDI extension. The core
library stays in its own wheel. This package installs libndi under its own
`_ndi` directory and uses relative library paths on Linux, so `LD_LIBRARY_PATH` is not
needed to locate the installed NDI and core libraries.

## Using the input

```python
from pixtreme.transport.ndi import Input
from pixtreme.transport.session import FrameBuffer

pre = FrameBuffer(8)
source = Input(0, pre, "STUDIO (Camera)", delay_ms=0)
source.open()
try:
    while (item := source.next()) is not None:
        # Check before selecting a model that needs motion vectors or residuals.
        assert item.snapshot.info.side_data_status == "unavailable"
        process(item.frame, item.snapshot)  # Your pipeline receives TransportFrame values.
finally:
    source.close()
```

`Input(device, buffer, name, delay_ms=0)` returns the core's `Entrance`. Use the
source's NDI name, including its host component as advertised by NDI discovery.
The caller owns the pre buffer and chooses its capacity and overflow behaviour.
`open()` creates the receiver, applies its receive delay, and returns without
waiting for the sender. `snapshot` remains `None` until the first successful
`next()`, which returns the first picture with its capture number and actual
frame period. `request_stop()` interrupts reception even before a sender appears.
`close()` joins reception and destroys the receiver; the caller's buffer and
already returned frames retain their contents.

The native receive thread captures each SDK picture once, in arrival order.
It does not use frame sync, repeat pictures, or discard pictures to match a
receiver clock. Each frame has a consecutive capture number and a
`received_at_ns` observed once when the SDK returns that picture, before image
conversion. The frame period and image description follow source changes.
Observed SDK video loss produces a source `unknown` discontinuity; missing
source positions are not guessed. Caller-buffer overflow remains visible
through the core's drop records.

`delay_ms` is an optional nonnegative integer in the SDK `int` range
(0 through 2147483647). The same value is passed to the SDK video and audio
receive delays. Positive values smooth network arrival by waiting inside the
SDK before capture returns. That wait precedes `received_at_ns` and is separate
from an output's delay budget. The default is 0.

Decoded pictures are converted using pixtreme's public operations. Received
UYVY is legal-range YCbCr: BT.601 at up to 720x576, BT.709 above that through
1920x1080, and BT.2020 when either dimension exceeds 1920x1080. HD and UHD use
Rec.709 and Rec.2020 colorspaces respectively; unspecified colour axes use
the module's explicit Rec.709 gamma and left chroma siting. SD primaries default
to Rec.709. Received RGB has no YCbCr matrix, uses Rec.709 colorspace and gamma
when unspecified, and RGBA retains its alpha. The receiver requests 8-bit
UYVY/RGBA from the SDK.

The returned float32 Frame uses full scale: legal Y codes 16 and 235 become
0 and 1, and Cb/Cr code 128 becomes zero. `item.snapshot.info.colour` retains
the source interpretation, including `resolved["range"] == "legal"` for UYVY.
Its observations contain only source signalling; resolution-based module
fallbacks stay absent in `observed`. An `ndi_color_info` metadata element can
override matrix, primaries and transfer. Resolved matrix, colorspace and gamma
match the Frame, and `defaults_used` is empty because the module specifies the
PixelBuffer declarations explicitly. Unrepresentable colour declarations fail
with a reason.

No audio format is declared until audio arrives. The input uses the core's
`PcmNormalizer` to produce one common format: 48000 Hz, stereo, interleaved
`s16le`. It preserves the normalizer's phase across packets, including when a
mono or different-rate source is converted. Source timestamps establish the
initial correspondence to video; subsequent PCM positions advance by the
normalized sample counts. SDK submission jitter and local receipt intervals
never create gaps or overlaps in a continuous PCM stream. Observed SDK audio
loss and source format changes begin a new core audio origin. Later missing
PCM remains pending in the core; the input does not hold pictures for audio
or generate silence. `audio_synchronized` is true only with usable source
timestamp correspondence. Without such correspondence, synchronization is
not claimed. The core output controls bounded waiting and silence for missing
PCM.

PCM captured before the first picture is retained by the module until that
picture initializes the core's audio origin. The next receive call submits it
at its source-derived position before handling a stop request. A returned
picture's audio reference can remain pending until that submission completes.

NDI's 100 ns video timecode is carried as a core source timecode only when it
can be represented by a displayed hours/minutes/seconds/frame label at the
received cadence without loss. NDI provides no drop-frame flag. Other values,
including timestamps outside a displayed day, have no source timecode.
The standard SDK supplies decoded pictures without motion vectors or
residuals; each picture's snapshot reports `side_data_status` as `unavailable`.
Decoded NDI HX is supported through that SDK path. Compressed NDI HX access
units require the Advanced SDK and are outside this input.

## Using the output

```python
from fractions import Fraction
from pixtreme.transport.ndi import Output
from pixtreme.transport.session import ExitSettings, FrameBuffer

post = FrameBuffer(2)
settings = ExitSettings(Fraction(1001, 30000))  # 30000/1001 fps
settings.audio.enabled = True
settings.audio.accepted_formats = [
    (48000, 2, "stereo", "s16le"),
    (48000, 2, "stereo", "f32le"),
]
settings.audio.output_format = (48000, 2, "stereo", "f32le")
settings.audio.target_format = "FLTP"
settings.ancillary_kinds = ["caption"]

output = Output(0, post, settings, "Processed output")
output.open()
try:
    for frame in processed_frames:  # TransportFrame values from your pipeline.
        post.put(frame)
    post.close()
    output.join()
finally:
    output.close()
```

`Output(device, buffer, settings, name)` requires all four arguments. Device
numbers follow the core's CUDA device selection. `Output` returns the core's
`Exit`, with `open`, `request_stop`, `join`, `close`, `check`, and
`inspection`. `open()` creates the NDI source; video begins after the first
frame is acquired. The output never closes the caller's buffer.
`Output` is a factory function;
use `isinstance(output, Exit)` to check the returned session, with `Exit` imported
from `pixtreme.transport.session`.

`ExitSettings.prefill_frames` defaults to 2. Set it to 1 to start with one
buffered frame. The value must be positive and fit the buffer's capacity.
Closing the buffer drains its remaining frames even below that count.

On each output period, the core's sending thread takes one picture from the
buffer, serializes it into the module's host memory, and passes it to the SDK.
There is no queue of encoded pictures before sending.

While an open buffer has no new frame, the output repeats the latest picture
with silent audio. `request_stop()` stops acquisition and sends only pictures
already accepted by the core; pictures left in the buffer are not sent.
`close()` joins the workers and destroys the source.

## Video, audio, and timecode

The core's sending thread measures each frame period on a monotonic clock and
controls the sending interval. The output uses `Pacing.CORE_CLOCK` with
`clock_video` and `clock_audio` disabled, so the SDK sends each picture as soon
as it receives it.
Resolution follows each input image, and frame rate follows
`ExitSettings.frame_period`. Image widths must be divisible by two.

RGB images (without a matrix) are serialized as RGBA; images without alpha
receive opaque alpha. YCbCr images (with a matrix) are serialized as UYVY:
8-bit 4:2:2, legal range, with byte order Cb0, Y0, Cr0, Y1. The core converts
matrix, primaries, and transfer to the output's colour values before packing.

The output uses NDI's resolution rule for the UYVY matrix and the module's
shared input/output defaults for primaries:

| Resolution | Output matrix | Output primaries |
| --- | --- | --- |
| Width > 1920 or height > 1080 | BT.2020 | Rec.2020 |
| Otherwise, width > 720 or height > 576 | BT.709 | Rec.709 |
| Otherwise | BT.601 | Rec.709 |

UYVY uses Rec.709 transfer and left chroma siting. RGBA uses Rec.709 primaries
and transfer, full range, and preserves each pixel's alpha when present.
These explicit defaults also apply to received pictures without colour metadata.
The output sends no `ndi_color_info`; external receivers may interpret undeclared
colour differently. Conversion is always allowed and requires no configuration key.
The core serializes pixels into page-locked host memory, which the output
passes directly to the SDK. That memory remains valid until serialization and
SDK reads finish, including during shutdown.

When audio is enabled, the core supplies interleaved `f32le` PCM at the
requested sample rate and channel layout. The output rearranges it into NDI's
planar float layout without changing sample values or counts. The core handles
dropped frames, repeats, padding, and trimming. At 48 kHz and 30000/1001 fps,
five periods contain 8008 samples per channel. Disabled audio is not sent.

With core 0.36.0 or later, a frame can carry an `AudioReference` into the core's
`PcmStore`. The core resolves that range immediately before sending, so PCM
arriving while the picture is prepared can still reach the output. Set
`settings.audio.max_audio_wait_ms` to limit the wait for missing PCM; `None`
uses one nominal frame period and `0` does not wait. The budget starts when the
core selects the picture and includes preparation and pacing. Repeated pictures
and pictures without audio send nominal silence without waiting. The output's
`inspection.audio_filled_samples` counts unread source samples sent as silence;
`audio_added_samples` and `audio_trimmed_samples` count output sample adjustments
separately. NDI preserves the PCM that the core supplies.

An input `FrameTiming.source_timecode` is converted to the NDI video frame's
100 ns timecode units. Its displayed hours, minutes, seconds, and nominal-rate
frame fraction are preserved, including for drop-frame labels. Fractional ticks
round to the nearest integer, with halves rounding up. Repeated pictures retain
that input timecode. Pictures without an input timecode receive the current
system time on every send, including repeats.

## Ancillary metadata

Declare accepted kinds in `settings.ancillary_kinds`; the core rejects
undeclared kinds. Each new picture's `AncillaryData.items` becomes XML in the
NDI video frame's metadata:

```xml
<pixtreme_ancillary encoding="base64"><item kind="Y2FwdGlvbg==">PCY=</item></pixtreme_ancillary>
```

The example carries kind `caption` and payload bytes `<&`. Each `AncillaryItem`
becomes one `item`; its kind and payload use standard padded base64. Order,
duplicate kinds, opaque bytes, and empty payloads are preserved. Empty
collections produce no metadata. Repeats omit the original ancillary metadata.

The input restores these container items without changing their opaque bytes.
Other video metadata becomes one item with kind `ndi-metadata`, holding the
original UTF-8 XML bytes. Declare `ndi-metadata` in an output's accepted kinds
to forward that payload directly as video metadata. The output does not
reformat the XML or wrap it in base64. When a picture also has other kinds,
raw metadata payloads appear first in their original relative order, followed
by one `pixtreme_ancillary` container for the other items, inside an
`ndi_metadata_group` root. The input extracts container items and preserves
the other child elements as `ndi-metadata` without reformatting their bytes.
`ndi_color_info` describes colour and goes into the Frame's colour record,
so it is excluded from ancillary data forwarded to an output.

## Send timing trace

Before `open()`, call the core session's `output.set_trace(trace)` with a
`pixtreme.trace.Trace` containing eight integer values per record. NDI records
kind 1001 immediately before the synchronous SDK video send, after audio has
been rearranged, and 1002 when that send returns. Kinds 1003 and 1004 surround
the SDK audio send and are omitted when audio is disabled. These rows join the
core's send rows (13 and 14) by session ID and work ID, with the core's input
number, epoch, sequence, ticket, and repeat flag. Reading and exporting the
trace belongs to the caller after the session has stopped. Without a trace,
the module makes no measurement clock or recording calls; SDK inputs and the
core's send cadence are unchanged.

## C++ integration

The wheel installs its CMake package under `pixtreme/transport/ndi/cmake`.
Use your organization's shared Conan recipes to obtain the core 0.53.0 and
pixtreme 2.31.1 prebuilt SDKs and retain their Conan lock. The module's native SDK
archive is available alongside wheels in the public GitHub Release. Configure with that Conan toolchain, set
`pixtreme-transport-ndi_DIR` to the module's installed CMake directory, call
`find_package(pixtreme-transport-ndi CONFIG REQUIRED)`, and link `pixtreme::ndi`.
Include `<pixtreme/transport/ndi/output.hpp>` to use `Output`. Public headers
require the core SDK and do not require NDI SDK headers. The C session owns the
NDI sender; the C++ `Output` is a view of that session. Close the session explicitly
or release its last owning handle to stop it. Overriding the C++ view's processing
hooks to replace NDI sending is not supported.

Include `<pixtreme/transport/ndi/input.hpp>` to construct
`pixtreme::transport::ndi::Input(device, pre_buffer, name, delay_ms)`; the last
argument defaults to 0. It uses the same owning core session model as `Output`.
The named module descriptor `ndi` provides both `create_entrance` and
`create_exit` in shared and source-built static modules. Input options are
`source_name` (required) and `receive_delay_ms` (optional); invalid, unknown,
repeated, or missing options report their key path without exposing option values.

To select the output by name, link only `pixtreme::transport` and include
`<pixtreme/transport/module.h>`. The core finds the installed NDI wheel without
importing Python or linking the NDI C++ library:

```cpp
// post is a caller-owned session::FrameBuffer. Build core settings via its C API.
namespace pt = pixtreme::transport;
pixtreme_transport_exit_settings_options values{};
values.struct_size = sizeof(values);
values.period_numerator = 1;
values.period_denominator = 25;
values.prefill_frames = 1;
pixtreme_transport_exit_settings* raw_settings = nullptr;
pixtreme_transport_error* error = nullptr;
auto status = pixtreme_transport_exit_settings_create(&values, &raw_settings, &error);
pt::check_status(status, error);
auto settings = pt::Handle::adopt(raw_settings);
const pixtreme_transport_module_exit_request request{
    sizeof(request), 0, post.native_handle(), settings.get()};
const pixtreme_transport_module_option options[]{
    {sizeof(pixtreme_transport_module_option), {"source_name", 11}, {"Processed output", 16}}};
auto output = pt::module::create_exit("ndi", request, options);
// output owns one unopened core exit reference. Use the core C session API.
status = pixtreme_transport_exit_open(output.get(), &error);
pt::check_status(status, error);
// Feed post, then close the session before releasing output.
```

The C equivalent is `pixtreme_transport_module_create_exit` from
`<pixtreme/transport/c/module.h>`. Supply the module name `ndi`, the request,
and the options array with its count and byte stride. A successful result owns
one unopened exit reference; close it and release it with the core handle API.
`source_name` is the only NDI output option and is required exactly once. Its
value is the source name as UTF-8 bytes without NUL. Unknown, duplicate, missing and invalid
options report the key path without including the value. Device, buffer and
core settings belong in the request. Each creation returns an independent
session and preserves the caller's current CUDA device and context.

The wheel places `libpixtreme_transport_ndi_entry.so` under
`pixtreme/transport/ndi/lib/`. Discovery returns CPU metadata; creating the
output or the input loads the same module library used by Python and reports
missing Avahi libraries with the system packages to install.

On Windows the entry is `pixtreme/transport/ndi/bin/pixtreme_transport_ndi_entry.dll`.
The body is `bin/pixtreme_ndi-<core ABI>.dll`, using the ABI of the core SDK
selected at build time; its import library is
`lib/pixtreme_ndi.lib`. The core and module must share one installation tree.
Native applications choose their own DLL search directories and load each
pixtreme and core DLL once. Link through `pixtreme::ndi` when using C++ `Output`.

## Building from source

Source builds use Conan 2 for the prebuilt core and pixtreme SDKs, the shared CUDA
toolkit recipe, Catch2, and a local description of the licensed NDI SDK.
The shared recipe archive is verified against `SHA256SUMS`, and exported recipe
revisions must match `cpp/conan.lock`. Core and pixtreme are not compiled.
Install the pinned CLI with `uv tool install conan==2.34.0`,
set `NDI_SDK_ROOT`, and prepare test dependencies with
`python3 cpp/conan/prepare-tests.py`. `cpp/conan/prepare.py` prepares dependencies
before CMake runs; the shared tools profile supplies locked CMake and Ninja
binaries from ConanCenter. Development builds do not need an OS
CUDA Toolkit. See [DEVELOPMENT.md](DEVELOPMENT.md) for Python builds,
standalone C++ tests, and manylinux wheels.

## License

This module's code is distributed under the
[MIT license](https://github.com/sync-dev-org/pixtreme-transport-ndi/blob/main/LICENSE).

The bundled libndi is subject to the NDI SDK License Agreement and the notices
in `libndi_licenses.txt` on Linux or `Processing.NDI.Lib.Licenses.txt` on Windows.
The wheel includes the complete nanobind and robin_map license texts and a
libndi notice in its `.dist-info/licenses/` directory.

## NDI terms and attribution

libndi is bundled with `libndi_licenses.txt`. NDI SDK redistribution remains
subject to its license conditions. Applications using NDI must supply the
applicable SDK license terms and notices to users, link to
[NDI](https://ndi.video/), and show the NDI® trademark and the attribution
“NDI® is a registered trademark of Vizrt NDI AB” near NDI selection or use,
in their About box, and in their trademark notices.
