# multiplatform plugin preparation

## context

aurora 0.1.0, created by [joe croskey](https://www.joecroskey.com/world.html?open=projects), is an audio metering and visualization plugin. this source came from `Aurora-0.1.0-source.zip`, provided by lucas on october 14, 2026.

- archive sha256: `0140245bad78765f581ef5dfdc7fc6be9fb53d5fc2d021ba3a1dddaf5e1f9234`.
- import commit: `9c51778`; all 4,340 files were compared byte for byte with the archive, including executable permissions.
- the outer `aurora-meters/` directory was omitted so that `CMakeLists.txt` sits at the repository root.
- c++17, cmake 3.22 or newer and juce 8.0.6, included in `third_party/JUCE/`.
- the original documentation identifies juce commit `51a8a6d7aeae7326956d747737ccf1575e61e209`; this import preserves the received copy without upgrading it.
- the original license is in [LICENSE](../LICENSE), with dependency notices in [THIRD_PARTY.md](../THIRD_PARTY.md).

the [author's installation guide](https://www.joecroskey.com/aurora/downloads/INSTALLATION.html) describes a macos release in vst3 and auv2 formats. it does not include windows or linux builds or system audio capture.

### findings from the release documentation

- [TECHNICAL-STACK.html](../release-docs/TECHNICAL-STACK.html) and its [pdf version](../release-docs/TECHNICAL-STACK.pdf) describe the native architecture. the interface uses juce and the effects use opengl/glsl; the diffusion studio project is optional and is not a runtime dependency.
- [RELEASE-NOTES.txt](../release-docs/RELEASE-NOTES.txt) records eight automated suites, native interface tests and audio unit validation in the author's environment. actual ableton playback, intel runtime and other gpu configurations remained unverified. these results are not tests of this adaptation.
- the original binaries require macos 13.3, although `CMakeLists.txt` declares 11.0 as the default. the technical document explains this difference using binary metadata; the intended configuration must not be treated as verified compatibility.
- [START-HERE.txt](../release-docs/START-HERE.txt) and [GITHUB-UPLOAD.txt](../release-docs/GITHUB-UPLOAD.txt) describe distribution of a larger kit. the referenced `downloads/`, binaries, `SHA256SUMS.txt` and `RELEASE-MANIFEST.json` are not included in this source archive. this import has no binary release to publish.
- [DOWNLOAD-README.txt](../release-docs/DOWNLOAD-README.txt) and [INSTALLATION.html](../release-docs/INSTALLATION.html) contain instructions for the macos package, which uses ad hoc signing without notarization. windows and linux distributions will need their own instructions, and future macos builds will need updated validation and signing information.
- [FLUID-LICENSE.txt](../release-docs/FLUID-LICENSE.txt) preserves the mit notice for the adaptation of pavel dobryakov's solver. this notice accompanies the aurora and juce licenses.

`scripts/package-release.py` is also tied to the original release: it checks hashes from `docs/inline-controls-validation.json`, runs `codesign`, `lipo` and `otool`, and requires both macos binaries. packaging a new version requires its own validation records, manifest and checksums.

## objective

develop aurora as a native plugin for macos, windows and linux while retaining its meters, scenes, presets and unchanged audio passthrough. keep one shared codebase and isolate platform-specific build, installation and packaging steps.

target formats and initial validation scope:

- macos: vst3 and audio unit. preserve the original arm64/x86_64 build intent and validate runtime on each architecture before claiming support.
- windows: vst3, initially targeting x64. compile with a native windows toolchain and validate in a compatible daw.
- linux: vst3, initially targeting x86_64. choose and record a baseline distribution, compiler, system libraries and compatible daw before claiming distribution support.

juce 8.0.6 supports these platforms and filters plugin formats according to the target system. the existing `FORMATS AU VST3` configuration therefore does not require windows or linux to build au. this makes a shared build a reasonable starting point, but framework support alone does not establish aurora compatibility. additional formats such as lv2 and standalone system audio capture are outside the current scope.

minimum operating-system versions, test hosts and the initial linux distribution still need to be agreed with lucas. the macos 13.3 requirement describes the original distributed binaries; it is not a newly verified support guarantee.

next steps:

1. configure and build with native toolchains on all three platforms, recording actual errors before changing the code. preserve the macos build while adding windows and linux support.
2. build `Aurora_VST3` on each platform and `Aurora_AU` on macos, then run the existing tests. add a native ci build and test job for each platform once its build procedure is established.
3. adapt development tools that assume macos and the `/tmp/` paths used by visual tests. use the temporary directory provided by juce and keep signing and installation steps specific to each platform.
4. test each plugin in a compatible daw, including audio passthrough, saved host state, preset import/export, reopening the editor, resizing, display scaling and all six scenes. check graphics drivers and the linux display environment separately from compiler success.
5. package each validated platform and architecture with its licenses, matching source, installation instructions, validation record and checksums. publish only the combinations actually tested.

## constraints

- all new repository content and commit messages must be written in english. keep the initial import identical to the archive contents and make additions in later commits.
- this preparation does not change `Source/`, `Tests/` or `CMakeLists.txt` and does not deliver new plugin binaries or a ci workflow.
- the import environment is linux. the five local test suites do not establish a complete linux plugin build, msvc or apple clang compilation, daw loading or gpu rendering on any platform.
- `scripts/build.sh` configures apple architectures and can install cmake/ninja in `.tools/`. the existing installation, signing and packaging scripts target macos; they need platform-specific counterparts or adaptation before serving all three systems.
- linux builds require the system development libraries used by the enabled juce modules, including audio, font, x11 and opengl dependencies. package names and runtime compatibility depend on the chosen distribution. consult the pinned [linux dependency guide](../third_party/JUCE/docs/Linux%20Dependencies.md) before proposing installations.
- `Source/VisualSurface.cpp` requests opengl 3.2. context creation, shaders and performance need testing in graphical sessions on each platform. an automated build does not validate graphics behavior.
- `AuroraPreview` uses silent synthetic audio for development. it does not capture audio from other applications and is not a standalone monitoring product.
- preserve the received juce source and license notices. dependency changes and additional installations must be agreed with lucas.
- the original results in `docs/VALIDATION.md` and `release-docs/` are historical records from the author's environment.

## examples

### proposed native builds

these commands have not been executed in this repository. they assume cmake 3.22 or newer and the relevant native compiler, sdk and system development libraries are already installed. run from the repository root, using a separate build directory on each system. the build commands include the test executables so that ctest can run afterward.

#### macos

requires apple's command line tools or xcode and ninja. this example uses the original release's macos 13.3 deployment baseline and requests both architectures; both still require runtime validation. it does not sign, notarize or install the output.

```sh
cmake -S . -B build-macos -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_DEPLOYMENT_TARGET=13.3 "-DCMAKE_OSX_ARCHITECTURES=arm64;x86_64"
cmake --build build-macos --parallel
ctest --test-dir build-macos --output-on-failure
```

expected plugin outputs: `build-macos/Aurora_artefacts/Release/VST3/Aurora.vst3` and `build-macos/Aurora_artefacts/Release/AU/Aurora.component`.

#### windows

requires visual studio 2022 with c++ tools and the windows sdk.

```powershell
cmake -S . -B build-windows -G "Visual Studio 17 2022" -A x64
cmake --build build-windows --config Release --parallel
ctest --test-dir build-windows -C Release --output-on-failure
```

expected plugin output: `build-windows/Aurora_artefacts/Release/VST3/Aurora.vst3`.

#### linux

requires a c++17 compiler such as gcc or clang, ninja, pkg-config and the development dependencies for the enabled juce modules. no packages are installed by these commands.

```sh
cmake -S . -B build-linux -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build-linux --parallel
ctest --test-dir build-linux --output-on-failure
```

expected plugin output: `build-linux/Aurora_artefacts/Release/VST3/Aurora.vst3`.

the expected output paths still need confirmation from successful builds. vst3 bundles contain platform-specific binaries; they are not interchangeable between operating systems. the graphical validation executables are separate from ctest and require a desktop session and portable temporary paths. installation and daw scanning must be verified separately for each platform.

references: [juce 8.0.6 platform requirements](https://github.com/juce-framework/JUCE/blob/8.0.6/README.md), [cmake api](https://github.com/juce-framework/JUCE/blob/8.0.6/docs/CMake%20API.md), [linux dependencies](https://github.com/juce-framework/JUCE/blob/8.0.6/docs/Linux%20Dependencies.md) and [visual studio 2022 generator](https://cmake.org/cmake/help/latest/generator/Visual%20Studio%2017%202022.html).

### local import verification

on october 14, 2026, shader synchronization was checked with:

```sh
python3 scripts/sync-pearl-cloud.py --native --check
```

the five tests independent of juce were compiled directly with g++ 16.2.1 using `-std=c++17 -O2 -pthread -I Source`, and all passed: `AnalysisTests`, `VisualFramingTests`, `VisualReactivityTests`, `WaveformClockTests` and `WaveformProjectionTests`. `AnalysisTests` and `VisualReactivityTests` also include `Source/Analysis.cpp` in the compilation. temporary executables were kept outside the repository.

the 43 source hashes in `docs/inline-controls-validation.json` also match the files received in the archive. this confirms correspondence with the author's record without repeating the graphical tests or validating the macos binaries.

these checks cover the analysis code and visualization calculations. complete plugin builds, the remaining tests and daw/gui validation on macos, windows and linux are still pending in this repository.
