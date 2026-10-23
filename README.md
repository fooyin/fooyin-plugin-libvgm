# fooyin libvgm input plugin

An input plugin for [fooyin](https://www.fooyin.org/) that uses [libvgm](https://github.com/ValleyBell/libvgm)
to play VGM (`.vgm` and `.vgz`), S98 (`.s98`), GYM (`.gym`), and DOSBox Raw OPL (`.dro`) files

## Installation

Pre-built bundles for Linux x64 and Windows x64 are available from the project's
[GitHub releases](https://github.com/fooyin/fooyin-plugin-libvgm/releases).
Download `fooyin-libvgm.fyplugin` for both platforms or choose the smaller Linux or Windows bundle.
Install it from fooyin's `Plugins` settings page, then restart fooyin.

## Building from source

You will need:

- A C++ compiler with C++23 support
- CMake 3.19 or later
- fooyin, including its CMake development files
- libvgm (`vgm-player`)

The repository includes libvgm as a submodule and uses it automatically when a system installation cannot be found. 
Clone recursively, configure, and build:

```
git clone --recurse-submodules https://github.com/fooyin/fooyin-plugin-libvgm.git
cd fooyin-plugin-libvgm
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

The built plugin is `build/fyplugin_vgminput.so`. To install it using CMake:

```
cmake --install build
```

## License

fooyin-plugin-libvgm is licensed under the [GNU General Public License, version 3 or later](COPYING).
