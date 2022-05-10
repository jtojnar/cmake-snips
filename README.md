# Please fix your CMake builds

As a very active downstream packager for a Linux distro, I often encounter packages that make assumptions which don’t hold for all systems they run on, or build tools they use. For projects using CMake especially, I find that many show at least one of the issues below.

I compiled a list of issues in this document that come up regularly in CMake projects. For most of them, there are simple best practices that can be followed to keep a project portable.

On a personal note, my recommendation is to move to the [Meson](https://mesonbuild.com/) build system where possible. It has a more modern design with better documentation, and does a good job avoiding common pitfalls. In my experience Meson makes life easier for both upstream developers and downstream packagers.

## Hardcoding the installation paths

This is not limited to CMake. Fortunately, it is rare.

The [Filesystem Hierarchy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) is a useful default but there are many reasonable points someone would want to install a project to a different path than `/usr`. The standard also has shortcomings leading some distributions like GoboLinux or NixOS to deviate from it.

Your project should allow changing the paths using something like [`GNUInstallDirs`](https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html). You also should not forget to update path references in the installed files.


## Creating custom install path options

This is more of a inconvenience but if you ask users to pass `-DMY_PROJECT_INSTALL_PREFIX=foo`, you should consider switching to [`GNUInstallDirs`](https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html). The module offers configurable variables for all common installation directories people are familiar with. No need to reinvent the wheel.


## Assuming `CMAKE_INSTALL_<dir>` is relative path

There is no guarantee that variables like `CMAKE_INSTALL_LIBDIR` or `CMAKE_INSTALL_INCLUDEDIR` are relative paths, or even located under `CMAKE_INSTALL_PREFIX`. In fact, the [documentation](https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html) explicitly mentions that they can be absolute several times.

If you need absolute paths, **never** concatenate `CMAKE_INSTALL_<dir>` to `CMAKE_INSTALL_PREFIX` manually. Use `CMAKE_INSTALL_FULL_<dir>` instead.


## Concatenating paths when building pkg-config files

This is actually the same issue as the previous one but with a slight twist. The variables in pkg-config files are expected to contain interpolations (e.g. `completiondir=${datadir}/bash-completion`) so that projects can install files relative their own directories instead of the ones of the project whose variables they use (see https://www.bassi.io/articles/2018/03/15/pkg-config-and-paths/). This means you cannot just use the `CMAKE_INSTALL_FULL_<dir>` variables and need to construct the paths yourself.

For CMake ≥ 3.20, you can use the [`cmake_path`](https://cmake.org/cmake/help/latest/command/cmake_path.html#append) function:

```cmake
cmake_minimum_required(VERSION 3.20) # For cmake_path function

# …

cmake_path(APPEND libdir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_LIBDIR}")

configure_file(
  "${PROJECT_SOURCE_DIR}/my-project.pc.in"
  "${PROJECT_BINARY_DIR}/my-project.pc"
  @ONLY)
```

Unfortunately, CMake < 3.20 does not provide a function for joining paths so you will need to add it yourself. I have created a [module](CMakeScripts/JoinPaths.cmake) you can copy to your code base and use as follows:

```cmake
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/CMakeScripts")

include(JoinPaths)

# …

join_paths(libdir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_LIBDIR}")

configure_file(
  "${PROJECT_SOURCE_DIR}/my-project.pc.in"
  "${PROJECT_BINARY_DIR}/my-project.pc"
  @ONLY)
```

See also a [real-life example](https://github.com/fmtlib/fmt/pull/1657) of the fix in {fmt} library.

## License

This text is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The source code is dual-licensed under [MIT](LICENSE.md) and [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
