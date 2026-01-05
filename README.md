[![Travis-CI Status](https://img.shields.io/travis/zeromq/cppzmq/master.svg?label=Linux%20|%20OSX)](https://travis-ci.org/josepholasoji/Geo-Location-Service)
[![Appveyor Status](https://img.shields.io/appveyor/ci/zeromq/cppzmq/master.svg?label=Windows)](https://ci.appveyor.com/project/zeromq/cppzmq/branch/master)
[![Coverage Status](https://coveralls.io/repos/github/zeromq/cppzmq/badge.svg?branch=master)](https://coveralls.io/github/zeromq/cppzmq?branch=master)
[![License](https://img.shields.io/github/license/zeromq/cppzmq.svg)](https://github.com/josepholasoji/Geo-Location-Service/LICENSE)

# Geo-Location Service (TK103 GPS Driver)

Geolocation service (GS) is a small C++ middleware that accepts GPS tracker device messages over TCP and processes them via a pluggable “device driver” model.

This repository ships with a TK103 driver plugin and a host service that:

- Loads device driver plugins from a `services/` folder at startup
- Opens one TCP listener per plugin on the port returned by the plugin (TK103: `2772`)
- Publishes parsed device feedback on ZeroMQ PUB (`tcp://*:5555`)
- Optionally persists feedback / device registry lookups via ArangoDB (HTTP)

License: MIT (see [LICENSE](LICENSE)).

## Solution layout

The main Visual Studio solution is [Geo-Location-Service.sln](Geo-Location-Service.sln).

- `geo_location` (project name: `geo_location_svc`): host service executable
- `tk103`: TK103 plugin (built as a DLL but output extension is `.gps`)
- `geo_location_svc_lib`: shared SDK implementation used by the host and plugins
- `geo_location_tests`, `tk103_tests`, `unittests`: test projects

## Build prerequisites (Windows)

This is a C++/MSBuild solution targeting the Visual Studio 2017 toolset (`v141`).

Dependencies used by the codebase:

- Boost (used for Asio, JSON/property_tree, string utils)
- ZeroMQ 4.0.x (used for PUB/SUB)
- cpprestsdk headers are vendored under `include/cpprest/` (no NuGet restore required for those headers)

Important note: some `.vcxproj` files contain machine-specific absolute include/library paths (for example `C:\SUPER\...` or `C:\boost_1_67_0\...`). You will likely need to retarget these paths to match your machine (Project Properties → C/C++ → Additional Include Directories; Linker → Additional Library Directories).

## Building

1. Open [Geo-Location-Service.sln](Geo-Location-Service.sln) in Visual Studio.
2. Select `x64` + `Debug` (or `Release`).
3. Build the solution.

Expected outputs (x64 configs):

- Host service exe goes to `bin/` (see `geo_location/geo_location.vcxproj` `OutDir`)
- Plugins go to `bin/services/` and are built with `.gps` extension (see `tk103/tk103.vcxproj` `TargetExt` + `OutDir`)

## Running the service

Run the host service from the `bin/` folder so the relative `services/` plugin directory resolves correctly.

At runtime the host service:

- Enumerates plugins from `services/*.gps`
- Loads each plugin and calls the exported `load(...)` function
- Starts a TCP server per plugin on `gps->serverPort()`

### Default ports

- TK103 device TCP listener: `2772` (see `tk103/tk103.cpp` `serverPort()`)
- ZeroMQ PUB/SUB: `5555` (see `geo_location_svc_lib/sdk.cpp`)

### Logs

The SDK initializes NanoLog with output directory `logs/`.

If you run from `bin/`, you should see logs under `bin/logs/` (for example `geolocation_service.log.*`).

## Device protocol notes (TK103)

The TCP session logic reads incoming bytes and treats `)` as the end-of-message delimiter.

The repository includes protocol reference PDFs under [protocols/](protocols/):

- `protocols/TK103.pdf`
- `protocols/GPS tracker Communication__ Protocol V1.51.pdf`

### Example messages

Examples used by tests (see `tk103_tests/modelutil.h`):

- Handshake request: `(040331141830BP00000013632782450HSO)`
- Handshake response: `(040331141830AP01HSO)`
- Login request: `(080524101241BP05000013632782450080524A2232.9806N11404.9355E000.1101241323.8700000000L000450AC)`
- Login response: `(080524101241AP05)`

## ArangoDB integration (optional but used by default code paths)

The SDK currently uses ArangoDB’s HTTP API for:

- Device registration lookup:

  - `GET http://127.0.0.1:8529/_db/geo_location/_api/document/devices/<deviceId>`
- Feedback log persistence:

  - `POST http://127.0.0.1:8529/_db/geo_location/_api/document/logs`

Credentials are currently hard-coded in `geo_location_svc_lib/sdk.cpp` (`imo` / `password`). If you use ArangoDB, update these in code before running.

For the device registry lookup to succeed, the collection `devices` must contain a document with `_key == <deviceId>`.

## Tests

There are GoogleTest-based projects under `geo_location_tests/`, `tk103_tests/`, and `unittests/`.

Notes:

- Some TK103 functional tests expect ArangoDB to be running and to contain a registered device document for device id `000013632782450` (see `tk103_tests/functional_tests.cpp`).

## Contact

For questions about this codebase, you can reach the original author at `josepholasoji@gmail.com`.
