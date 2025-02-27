
# Boost build by CMake

## Requirements:

- CMake >= 3.18
- Standard build tools needed for Boost build

## CMD Line Arguments

Specified in [CMakeLists.txt]

## Cross-compile

Activating cross-compile toolchain sets CMAKE_TOOLCHAIN_FILE, which then sets necessary CMAKE variables.
Important variables for Boost build are:
- **CMAKE_SYSTEM_NAME**: Target system name, if set, `CMAKE_CROSSCOMPILING` is set to `TRUE`.
- **CMAKE_SYSTEM_PROCESSOR**: Target system processor name.
- **CMAKE_CXX_COMPILER**: Target system CXX compiler.
- **CMAKE_SYSROOT**: Target system sysroot.

To configure compiler for `b2` command, configuring creates a `user-config.jam` file, defining CXX compiler for arm processor.

[CMakeLists.txt]: ./CMakeLists.txt