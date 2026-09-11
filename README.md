# dotspack for Home

This is a collection of .spack files for home

## Homebrew

### Installing Homebrew

For a mac with no admin rights, I often just [clone homebrew](https://docs.brew.sh/Installation#untar-anywhere-unsupported) into my home directory. This is not
the recommended way to install homebrew, but it works. 

```bash
mkdir -p $HOME/.homebrew
git clone https://github.com/Homebrew/brew $HOME/.homebrew
```

Then:

```bash
eval "$(homebrew/bin/brew shellenv)"
brew update --force --quiet
chmod -R go-w "$(brew --prefix)/share/zsh"
```

These are based on those from spack-stack

```bash
brew install coreutils
brew install gcc
brew install git
brew install lmod
brew install wget
brew install bash
brew install tcsh
brew install cmake
brew install openssl
brew install rust
```

NOTE 1: The install of gcc will be slow as they are built from source since we are using a non-standard location for homebrew.
NOTE 2: Yes, `rust` is there. Some Python projects need it

#### gawk from Homebrew

There is currently an issue between `gawk` from Homebrew and spack building `ncurses`. The issue
is it doesn't build `ncurses` which is a dependency of something in the chain. The workaround
is to *not* have `gawk` from Homebrew. So just to be safe:

```bash
brew uninstall gawk
```

### .zshenv

Add to .zshenv:

```bash 
eval "$(brew --prefix)/bin/brew shellenv"
. $(brew --prefix)/opt/lmod/init/zsh
```

## Clone spack

First, we need spack. Below we assume we clone spack into `$HOME/spack-mathomp4` as we 
are usually working on a fork of spack.

```bash
git clone -c feature.manyFiles=true git@github.com:mathomp4/spack.git spack-mathomp4
```

If you want to use the official spack, you can change the clone command to:
```bash
git clone -c feature.manyFiles=true git@github.com:spack/spack.git spack
```


## Environment

### .zshenv

```
export SPACK_ROOT=$HOME/spack-mathomp4
```

NOTE: If you are using the official spack, change this to:
```bash
export SPACK_ROOT=$HOME/spack
```

### .zshrc

```
# Only run these if SPACK_ROOT is defined
if [ ! -z ${SPACK_ROOT} ]
then
   export SPACK_SKIP_MODULES=1
   . ${SPACK_ROOT}/share/spack/setup-env.sh

   # Next, we need to determine our macOS by *name*. So, we need to have a
   # variable that resolves to "sonoma" or "sequoia"

   OS_VERSION=$(sw_vers --productVersion | cut -d. -f1)
   if [[ $OS_VERSION == 14 ]]
   then
      OS_NAME='sonoma'
   elif [[ $OS_VERSION == 15 ]]
   then
      OS_NAME='sequoia'
   elif [[ $OS_VERSION == 26 ]]
   then
      OS_NAME='tahoe'
   else
      OS_NAME='unknown'
   fi

   module use ${SPACK_ROOT}/share/spack/lmod/darwin-$OS_NAME-aarch64/Core
fi
```

We need the `OS_NAME` variable to determine which lmod files to use as a laptop might be on 
either `sonoma` or `sequoia` and the lmod files are different.

## Spack Configuration

### repos

Due to how spack is now split with the package repo being separate, we
now clone it ourselves and tell spack where to find it. This is done in the `repos.yaml` file.

#### Clone spack package repository

```bash
git clone -c feature.manyFiles=true git@github.com:mathomp4/spack-packages.git spack-packages-mathomp4
```

Again, this is because we are working on a fork of the spack packages. If you want to use the official spack packages, you can change the clone command to:
```bash
git clone -c feature.manyFiles=true git@github.com:spack/spack-packages.git spack-packages
```

#### Clone geosesm-spack repository

We rely on an extra repo for `geosgcm` and `geosfvdycore`. 
This is retreived by:
```
git clone git@github.com:GMAO-SI-Team/geosesm-spack.git
```

#### repos.yaml

Finally, we need to tell spack where to find the package repositories. This is done in the `repos.yaml` file.

```yaml
repos:
  builtin:
    git: git@github.com:mathomp4/spack-packages.git
    destination: /Users/fortran/spack-packages-mathomp4
  geosesm: /Users/fortran/geosesm-spack/spack_repo/geosesm
```

Again, change as needed if you are using the official spack packages and, of course, use your username
in the path (or wherever you cloned the repos to).

### config

Set the number of `build_jobs` to 6 (or whatever you want)

```bash
spack config add config:build_jobs:6
```

### compilers

Now we have Spack find the compilers on our system:
```bash
spack compiler find
```

For example, I got:
```bash
❯ spack compiler find
==> Added 4 new compilers to /Users/fortran/.spack/darwin/compilers.yaml
    gcc@16.2.0 gcc@15.2.0 gcc@13.3.0 gcc@12.5.0 apple-clang@21.0.0
==> Compilers are defined in the following files:
    /Users/fortran/.spack/packages.yaml
```

Note that in Spack 1.0.0 and later, the compilers.yaml file is not used. Instead, the compilers are
added to the `packages.yaml` file. So, you can ignore the compilers.yaml file. An example of
how this will look will be:
```yaml
  packages:
    - spec: gcc@16.2.0 languages:='c,c++,fortran'
      prefix: /opt/homebrew
      extra_attributes:
        compilers:
          c: /opt/homebrew/bin/gcc-16
          cxx: /opt/homebrew/bin/g++-16
          fortran: /opt/homebrew/bin/gfortran-16
```

### toolchains

For simplicity, we'll also setup a toolchain file. An example is:
```yaml

toolchains:
  apple-gfortran-16:
  - spec: "%c=apple-clang"
    when: "%c"
  - spec: "%cxx=apple-clang"
    when: "%cxx"
  - spec: "%fortran=gcc@16"
    when: "%fortran"
  apple-nag:
  - spec: "%c=apple-clang"
    when: "%c"
  - spec: "%cxx=apple-clang"
    when: "%cxx"
  - spec: "%fortran=nag"
    when: "%fortran"
```

Now when installing packages, instead of doing:

```bash
spack install mapl %[virtuals=c,cxx] apple-clang@21.0.0 %[virtuals=fortran] gcc@16.2.0
```

we can do:

```bash
spack install mapl %apple-gfortran-16
```

Much simpler!

### packages

Now we can use `spack external find` to find the packages we need already in homebrew. But,
we want to exclude some packages that experimentation has found should be built by spack.

```bash
spack external find --exclude bison --exclude openssl \
   --exclude gmake --exclude m4 --exclude curl --exclude python \
   --exclude gettext --exclude perl --exclude meson
spack external find bash
```

#### tcsh

For some reason, `tcsh` is not found by `spack external find`. So we add it manually:
```yaml
  tcsh:
    externals:
    - spec: tcsh@6.24.16
      prefix: /opt/homebrew
```

#### Additional settings

Now edit the packages.yaml file with `spack config edit packages` and add the following:

```yaml
packages:
  all:
    providers:
      mpi: [openmpi]
      blas: [openblas]
      lapack: [openblas]
  cdo:
    variants: ~proj ~fftw3
    # cdo wanted a lot of extra stuff for proj and fftw3. Turn off for now
  esmf:
    require:
    - spec: +debug
      when: platform=darwin
  mapl:
    variants: +pfunit
  netcdf-c:
    variants: +dap
```

These are based on how we expect libraries to be built for GEOS and MAPL.

### concretizer

We also want to set up the concretizer to not reuse builds. For this, edit
`concretizer.yaml` by running `spack config edit concretizer` and add:

```yaml
concretizer:
  reuse: false
```

This is less efficient, but it seems to make spack understand that different Fortran packages need
to be compiled with the same compiler.

### modules

Next setup the `modules.yaml` file to look like this by running `spack config edit modules`:

```yaml
modules:
  default:
    'enable:':
    - lmod
    lmod:
      core_compilers:
      - apple-clang@21.0.0
      hierarchy:
      - mpi
      hash_length: 0
      include:
      - apple-clang
      all:
        suffixes:
          +debug: 'debug'
          build_type=Debug: 'debug'
        environment:
          set:
            '{name}_ROOT': '{prefix}'
      hdf5:
        suffixes:
          ~shared: 'static'
      esmf:
        suffixes:
          ~shared: 'static'
          esmf_pio=OFF: 'nopio'
  prefix_inspections:
    lib64: [LD_LIBRARY_PATH]
    lib:
    - LD_LIBRARY_PATH
```

## Spack Environment

The best way to manage all the bits needed for GEOSgcm and MAPL is to use a Spack Environment.

In the example below, we'll work on one for GEOSgcm.

NOTE NOTE NOTE: At the moment, when you deactivate your spack environment, it will screw
up your shell:

https://github.com/spack/spack/issues/48391

it removes things like homebrew from your PATH. So, until that is fixed, you might want to
use the spack environment in a subshell, e.g.,

```bash
zsh
spack env activate geosgcm-gcc16
# do stuff
spack env deactivate
exit
```

or a new terminal window.

### Create environment

```bash
spack env create geosgcm-gcc16
```

### Activate environment

```bash
spack env activate geosgcm-gcc16
```

### Add packages

#### GEOSgcm

```bash
spack add geosgcm %apple-gfortran-16
```

#### GEOSgcm Dependencies

If you only want to install the dependencies of GEOSgcm, you can do:

```bash
spack add geosgcm-deps %apple-gfortran-16
```

### Concretize

```bash
spack concretize -Uf
```

### Spack Install

Now we install into the environment:

```bash
spack install
```

### Fix up the environment for CC/CXX/FC

At the moment, the environment will not have `CC`, `CXX` and `FC` set to *anything* which is
not what we want. Unfortunately, this is a spack bug ([spack/spack#51855](https://github.com/spack/spack/issues/51855)).

For now, you can manually set them by doing:

```bash
spack config add env_vars:set:CC:$(which clang)
spack config add env_vars:set:CXX:$(which clang++)
spack config add env_vars:set:FC:$(which gfortran-16)
```

NOTE: You probably need to make a new terminal/subshell and reactivate the environment for this to take effect.
If I find a spack way, I'll update this.

### Apple Silicon: hwloc OpenCL / Metal Crash (`SIGILL: Illegal instruction: 4`)

On macOS Apple Silicon, Open MPI's internal `hwloc` topology discovery compiles an OpenCL plugin (`hwloc_opencl.so`). During `MPI_Init()`, `hwloc` queries Apple's OpenCL framework to discover GPU devices. On Apple Silicon, Apple's OpenCL routes queries through Metal (`MTLCopyAllDevices`), which triggers an illegal instruction crash in the GPU driver (`AGXMetalG15G_C0`) resulting in:

```text
prterun noticed that process rank X exited on signal 4 (Illegal instruction: 4).
```

To prevent this crash, `hwloc` must be instructed to skip OpenCL probing via:

```bash
export HWLOC_COMPONENTS="-opencl"
```

This is permanently configured across our environments via `~/.spack/env_vars.yaml`:

```yaml
env_vars:
  set:
    HWLOC_COMPONENTS: "-opencl"
```

> [!WARNING]
> **Do NOT run `spack load geosgcm-deps` when using Spack Environments!**
> When using Spack Environments (`spack env activate`), the environment view automatically symlinks all binaries (`mpifort`, `mpicc`, `nc-config`, etc.) into `.spack-env/view/bin` and prepends it to `$PATH`. Running `spack load` inside an active environment can disrupt `$PATH` and fails to load link dependencies like Open MPI.

## Not using Spack Environments

### spack install

If you are not using spack environments, you can install GEOSgcm (or whatever) directly with:

```bash
spack install geosgcm %apple-gfortran-16
```

### spack load

If you do `spack load` you need to do:

```bash
spack load geosgcm-deps
```

This is true even if you installed `geosgcm` as `geosgcm-deps` is a dependency of `geosgcm`.

### Loading lmodules

If you are using the module way of loading spack, you need to do:

```bash
module load apple-clang openmpi esmf python py-pyyaml py-numpy pfunit pflogger fargparse zlib-ng mepo udunits
```

This might be too much, but it works.

### Build command

The usual CMake commmand can be used to build GEOS or MAPL:

```bash
cmake -B build -S . --install-prefix=$(pwd)/install --fresh
cmake --build build --target install -j 6
```

NOTE: If you used `spack load` you'll need to supply the compilers to the first command:
```
cmake -B build -S . --install-prefix=$(pwd)/install --fresh -DCMAKE_Fortran_COMPILER=gfortran-16 -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
```
as `spack load` does not populate `FC`, `CC` and `CXX`.

### MAPL

No changes were needed for MAPL

### GEOS

#### Running (with modulefiles)

You'll need to update the `gcm_run.j` to not use `g5_modules` and instead use the spack modules if you are using
the spack modulefiles. This isn't need with `spack load`, so that's usually the easier way.

So comment out:

```csh
#source $GEOSBIN/g5_modules
#setenv DYLD_LIBRARY_PATH ${LD_LIBRARY_PATH}:${BASEDIR}/${ARCH}/lib:${GEOSDIR}/lib
```
and add:

```csh
source $LMOD_PKG/init/csh
module use -a $SPACK_ROOT/share/spack/lmod/darwin-tahoe-aarch64/Core
module load apple-clang openmpi esmf python py-pyyaml py-numpy pfunit pflogger fargparse zlib-ng
module list
setenv DYLD_LIBRARY_PATH ${LD_LIBRARY_PATH}:${GEOSDIR}/lib
```

Note that `LMOD_PKG` seems to be defined by the brew lmod installation...though not sure.
You might just need to put in a full path or something.
