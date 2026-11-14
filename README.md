# pixtreme-transport-ndi

Full NDI® video and audio output for [pixtreme.transport](https://github.com/sync-dev-org/pixtreme-transport).
NDI® is a registered trademark of Vizrt NDI AB. See [NDI](https://ndi.video/).

Send processed GPU frames from a transport core `FrameBuffer` to NDI receivers.
The package provides the Python `Output` factory, a C++ shared library, public
headers, a CMake package, and a bundled libndi runtime. It provides output only.

## Installation

```bash
pip install pixtreme-transport-ndi
```

Supported wheels target Linux x86_64 (including WSL2), CPython 3.12 and 3.13,
and `manylinux_2_28_x86_64` (glibc 2.28 or newer). Processing requires an NVIDIA
GPU and a driver that supports CUDA 13. Pixel serialization calls pixtreme's
conversions, built for sm 86, sm 89, and sm 120. The wheel requires no CUDA Toolkit or NDI SDK
installation.

Avahi's `libavahi-client.so.3`, `libavahi-common.so.3`, and their dependencies
must be installed on the host. Install `libavahi-client3` on Debian/Ubuntu or
`avahi-libs` on RHEL-family systems. NDI mDNS discovery also needs a running
`avahi-daemon`. Import errors identify a missing Avahi library and the packages
to install.

The dependency `pixtreme-transport>=0.33.1` installs the transport core, which
supplies frame types, buffers, worker threads, audio processing, and pixel
serialization. This module directly depends on `pixtreme>=2.24.0` and uses
pixtreme's color definitions to match the primaries and white point of YCbCr
images to the NDI standard for their resolution. The core's native ABI must
match the ABI used to build this module;
import rejects an incompatible core before loading the NDI extension. The core
library stays in its own wheel. This package installs libndi under its own
`_ndi` directory and uses relative library paths, so `LD_LIBRARY_PATH` is not
needed to locate the installed NDI and core libraries.

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
There is no queue of encoded pictures before sending. The core requires a
positive `ExitSettings.encoded_depth` and preserves it in the settings returned
for inspection, but NDI output does not use it to drive sending.

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
8-bit 4:2:2, legal range, with byte order Cb0, Y0, Cr0, Y1. The output preserves
the input matrix, colorspace, and transfer function. Convert incompatible
images in your pipeline before submitting them.

NDI interprets UYVY colour according to the image resolution:

| Resolution | Required matrix | Required colorspace |
| --- | --- | --- |
| Width > 1920 or height > 1080 | BT.2020 | Rec.2020 |
| Otherwise, width > 720 or height > 576 | BT.709 | Rec.709 |
| Otherwise | BT.601 | Preserved from the input |

An incompatible YCbCr image stops the output before reaching the SDK. The error
identifies its matrix, colorspace, resolution, and the required NDI standard.
The core serializes pixels into page-locked host memory, which the output
passes directly to the SDK. That memory remains valid until serialization and
SDK reads finish, including during shutdown.

When audio is enabled, the core supplies interleaved `f32le` PCM at the
requested sample rate and channel layout. The output rearranges it into NDI's
planar float layout without changing sample values or counts. The core handles
dropped frames, repeats, padding, and trimming. At 48 kHz and 30000/1001 fps,
five periods contain 8008 samples per channel. Disabled audio is not sent.

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
Use your organization's shared Conan recipes to obtain the core 0.33.1 and
pixtreme 2.24.0 prebuilt SDKs and retain their Conan lock. The module's native SDK
archive is available alongside wheels in the public GitHub Release. Configure with that Conan toolchain, set
`pixtreme-transport-ndi_DIR` to the module's installed CMake directory, call
`find_package(pixtreme-transport-ndi CONFIG REQUIRED)`, and link `pixtreme::ndi`.
Include `<pixtreme/transport/ndi/output.hpp>` to use `Output`. Public headers
require the core SDK and do not require NDI SDK headers. The C session owns the
NDI sender; the C++ `Output` is a view of that session. Close the session explicitly
or release its last owning handle to stop it. Overriding the C++ view's processing
hooks to replace NDI sending is not supported.

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
values.encoded_depth = 1;
values.prefill_frames = 1;
pixtreme_transport_exit_settings* raw_settings = nullptr;
pixtreme_transport_error* error = nullptr;
auto status = pixtreme_transport_exit_settings_create(&values, &raw_settings, &error);
pt::check_status(status, error);
auto settings = pt::Handle::adopt(raw_settings);
const pixtreme_transport_module_exit_request request{
    sizeof(request), 0, post.native_handle(), settings.get()};
const pixtreme_transport_module_option options[]{
    {sizeof(pixtreme_transport_module_option), {"name", 4}, {"Processed output", 16}}};
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
`name` is the only NDI option and is required exactly once. Its value is the
source name as UTF-8 bytes without NUL. Unknown, duplicate, missing and invalid
options report the key path without including the value. Device, buffer and
core settings belong in the request. Each creation returns an independent
session and preserves the caller's current CUDA device and context.

The wheel places `libpixtreme_transport_ndi_entry.so` under
`pixtreme/transport/ndi/lib/`. Discovery returns CPU metadata; creation loads the
same output library used by Python and reports missing Avahi libraries with
the system packages to install. NDI reception remains unsupported.

## Building from source

Source builds use Conan 2 for the prebuilt core and pixtreme SDKs, the shared CUDA
toolkit recipe, Catch2, and a local description of the licensed NDI SDK. Public
Release recipe archives are verified against `SHA256SUMS`, and exported recipe
revisions must match `cpp/conan.lock`. Core and pixtreme are not compiled.
Install the pinned CLI with `uv tool install conan==2.34.0`,
set `NDI_SDK_ROOT`, and prepare test dependencies with
`cmake -P cpp/conan/prepare-tests.cmake`. Development builds do not need an OS
CUDA Toolkit. See [DEVELOPMENT.md](DEVELOPMENT.md) for Python builds,
standalone C++ tests, and manylinux wheels.

## License

This module's code is distributed under the
[MIT license](https://github.com/sync-dev-org/pixtreme-transport-ndi/blob/main/LICENSE).

The bundled libndi is subject to the NDI SDK License Agreement and the notices
in `libndi_licenses.txt`.
The wheel includes the complete nanobind and robin_map license texts and a
libndi notice in its `.dist-info/licenses/` directory.

## NDI terms and attribution

libndi is bundled with `libndi_licenses.txt`. NDI SDK redistribution remains
subject to its license conditions. Applications using NDI must supply the
applicable SDK license terms and notices to users, link to
[NDI](https://ndi.video/), and show the NDI® trademark and the attribution
“NDI® is a registered trademark of Vizrt NDI AB” near NDI selection or use,
in their About box, and in their trademark notices.
