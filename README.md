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
GPU and a driver that supports CUDA 13. The transport core supplies CUDA kernels
for sm 86, sm 89, and sm 120. The wheel requires no CUDA Toolkit or NDI SDK
installation.

Avahi's `libavahi-client.so.3`, `libavahi-common.so.3`, and their dependencies
must be installed on the host. Install `libavahi-client3` on Debian/Ubuntu or
`avahi-libs` on RHEL-family systems. NDI mDNS discovery also needs a running
`avahi-daemon`. Import errors identify a missing Avahi library and the packages
to install.

The dependency `pixtreme-transport>=0.28.0` installs the transport core, which
supplies frame types, buffers, worker threads, audio processing, and pixel
serialization. This module directly depends on `pixtreme>=2.14.0` and uses
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

## C++ integration

The wheel installs its CMake package under `pixtreme/transport/ndi/cmake`.
Set `pixtreme-transport-ndi_DIR` to that directory, `pixtreme_transport_DIR`
to the core's CMake directory, and `pixtreme_DIR` to pixtreme's CMake directory
(`pixtreme.core.get_cmake_dir()`), call
`find_package(pixtreme-transport-ndi CONFIG REQUIRED)`, and link `pixtreme::ndi`.
Include `<pixtreme/transport/ndi/output.hpp>` to use `Output`. Public headers
require the core SDK and do not require NDI SDK headers. The C session owns the
NDI sender; the C++ `Output` is a view of that session. Close the session explicitly
or release its last owning handle to stop it. Overriding the C++ view's processing
hooks to replace NDI sending is not supported.

## Building from source

Source builds use Conan 2 for CUDA headers, Catch2, and a local description of
the licensed NDI SDK. Install the pinned CLI with `uv tool install conan==2.34.0`,
set `NDI_SDK_ROOT`, and prepare test dependencies with
`cmake -P cpp/conan/prepare-tests.cmake`. Development builds do not need an OS
CUDA Toolkit. See [DEVELOPMENT.md](DEVELOPMENT.md) for editable builds,
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
