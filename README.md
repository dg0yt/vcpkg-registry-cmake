# vcpkg-registry-cmake

This repository contains a cmake project which helps to work with an
existing local checkout of custom git registries for vcpkg. Working with
this separate directory is entirely up to the user:
- editing
- authoring commits
- pushing and pulling changes.

The cmake build systems offers the following:
- Initial cloning and bootstrapping of the vcpkg repository into a
  subdirectory of the binary directory.
- Importing configuration from the custom registry.
- Creating a vcpkg configuration file which establish the custom registry
  as default registry, and enables selected packages from the vcpkg registry.
- Individual targets for installing (building) ports from the custom registry,
  taking care of
  - uninstalling the previous build of the port
  - formatting the manifest files
  - updating the version files.
- An `all-ports` target which passes all custom ports at once to
  `vcpkg remove`/`vcpkg install`. This target can make full use of 
  vcpkg's binary caching, so it doesn't always need to rebuild the world.

Important note: 
To actually use even uncommitted local changes to the custom registry,
its ports directory is normally also passed as overlay ports to vcpg.


## Configuration

### User-defined configuration

The user must set a single cmake variable:

#### `REGISTRY_CHECKOUT`
The directory of the custom vcpkg registry git checkout.


### Registry-defined configuration

The custom registry can provide configuration information in a top-level
file `registry-config.cmake` by setting the following variables:

#### `REGISTRY_NAME`
The named to be used as CMake project name.
This can help to identify the registry in IDE project trees.

#### `VCPKG_BASELINE`
The git commit to be used as vcpkg repository baseline.

#### `VCPKG_IMPORTS`
The list of ports to be used from the vcpkg repository.
