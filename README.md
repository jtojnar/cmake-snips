# Please fix your CMake builds

As a very active downstream packager for a fringe Linux distro, I encounter many packages that have incorrect assumptions about systems they will run on or even build tools they use. Fortunately, most projects just work; if I focus my attention on projects using CMake, thought, the situation is sadder. Rarely does a CMake project not show at least one of the issues below.

When a majority of projects make the same mistakes, one starts to wonder if CMake is perhaps too complex for human beings to use. But it would be unreasonable to expect projects to switch to something saner like Meson, so I will at least try to describe common issues and how to fix them.


## Hardcoding the installation paths

This is not limited to CMake. Fortunately, it is rare.

[Filesystem Hiearachy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) is useful default but there are many reasonable points someone would want to install a project to a different path than `/usr`. The standard also has shortcomings leading some distributions like GoboLinux or NixOS to deviate from it.

Your project should allow changing the paths using something like [`GNUInstallDirs`](https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html). You also should not forget to update path references in the installed files.


## Creating custom install path options

This is more of a inconvenience but if you ask users to pass `-DMY_PROJECT_INSTALL_PREFIX=foo`, you should consider switching to [`GNUInstallDirs`](https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html). The module offers configurable variables for all common installation directories people are familiar with. No need to reinvent the wheel.


## Assuming `CMAKE_INSTALL_<dir>` is relative path

There is no guarantee that variables like `CMAKE_INSTALL_LIBDIR` or `CMAKE_INSTALL_INCLUDEDIR` are relative paths, or even located under `CMAKE_INSTALL_PREFIX`. In fact, the [documentation](https://cmake.org/cmake/help/latest/module/GNUInstallDirs.html) explicitly mentions that they can be absolute several times.

If you need absolute paths, **never** concatenate `CMAKE_INSTALL_<dir>` to `CMAKE_INSTALL_PREFIX` manually. Use `CMAKE_INSTALL_FULL_<dir>` instead.


## Concatenating paths when building pkg-config files

This is actually the same issue as the previous one but with a slight twist. The variables in pkg-config files are expected to contain interpolations (e.g. `completiondir=${datadir}/bash-completion`) so that projects can install files relative their own directories instead of the ones of the project whose variables they use (see https://www.bassi.io/articles/2018/03/15/pkg-config-and-paths/). This means you cannot just use the `CMAKE_INSTALL_FULL_<dir>` variables and need to construct the paths yourself.

Unfortunately, CMake does not provide a function for joining paths so you will need to add it yourself. I have created a [module](CMakeScripts/JoinPaths.cmake) you can copy to your code base and use as follows:

```cmake
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/CMakeScripts")

include(JoinPaths)

join_paths(libdir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_LIBDIR}")

configure_file(
  "${PROJECT_SOURCE_DIR}/my-project.pc.in"
  "${PROJECT_BINARY_DIR}/my-project.pc"
  @ONLY)
```


## License

This text is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The source code is dual-licensed under [MIT](LICENSE.md) and [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
