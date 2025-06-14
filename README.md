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
brew install gcc@14
brew install git
brew install lmod
brew install wget
brew install bash
brew install curl
brew install cmake
brew install openssl
brew install rust
```

NOTE 1: The install of gcc will be slow as they are built from source since we are using a non-standard location for homebrew.
NOTE 2: We specify `gcc@14` as GEOS does not yet support GCC 14. But, it's possible something from brew will ask for GCC 14 and that
might be installed.
NOTE 3: Yes, `rust` is there. Some Python projects need it

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
  geosesm: /Users/fortran/geosesm-spack
```

Again, change as needed if you are using the official spack packages.

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
    gcc@14.2.0  gcc@13.3.0 gcc@12.4.0  apple-clang@17.0.0
==> Compilers are defined in the following files:
    /Users/fortran/.spack/packages.yaml
```

Note that in Spack 1.0.0 and later, the compilers.yaml file is not used. Instead, the compilers are
added to the `packages.yaml` file. So, you can ignore the compilers.yaml file.

### packages

Now we can use `spack external find` to find the packages we need already in homebrew. But,
we want to exclude some packages that experimentation has found should be built by spack.

```bash
spack external find --exclude bison --exclude openssl \
   --exclude gmake --exclude m4 --exclude curl --exclude python \
   --exclude gettext --exclude perl --exclude meson
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
  hdf5:
    variants: +fortran +szip +hl +threadsafe +mpi
    # Note that cdo requires threadsafe, but hdf5 doesn't
    # seem to want that with parallel. Hmm.
  netcdf-c:
    variants: +hdf4 +dap
  esmf:
    variants: ~pnetcdf ~xerces
  cdo:
    variants: ~proj ~fftw3
    # cdo wanted a lot of extra stuff for proj and fftw3. Turn off for now
  pflogger:
    variants: +mpi
  pfunit:
    variants: +mpi +fhamcrest
  fms:
    variants: precision=32,64 +quad_precision ~gfs_phys +openmp +pic constants=GEOS +deprecated_io
  mapl:
    variants: +extdata2g +fargparse +pflogger +pfunit ~pnetcdf
```

These are based on how we expect libraries to be built for GEOS and MAPL.

### modules

Next setup the `modules.yaml` file to look like this by running `spack config edit modules`:

```yaml
modules:
  default:
    'enable:':
    - lmod
    lmod:
      core_compilers:
      - apple-clang@17.0.0
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


## Spack Install

Now we install packages.

```bash
spack install python py-numpy py-pyyaml py-ruamel-yaml
spack install openmpi
spack install esmf
spack install gftl gftl-shared fargparse pfunit pflogger yafyaml
spack install mepo
spack install udunits
```

This could just as well be:
```bash
spack install --only dependents [geosgcm|mapl]
```

### Regenerate Modules

Sometimes spack needs a nudge to generate lmod files. This can be done (at any time) with:

```bash
spack module lmod refresh --delete-tree -y
```

### Extra apple-clang module

Spack is not able to create a modulefile for apple-clang since it is a
builtin compiler or something. But, we want to have a modulefile for it
so we can have `FC`, `CC` etc. set in the environment. So we make one. There
is a copy in the `extra_modulefiles` directory. Copy it to the right place:

```bash
cp -a extra_modulefiles/apple-clang $SPACK_ROOT/share/spack/lmod/darwin-sequoia-aarch64/Core/
```

Note that the Spack lmod directory won't be created until you run a first `spack install` command.


## Building GEOS and MAPL

### spack load

If you do `spack load` you need to do:

```bash
spack load openmpi esmf python py-pyyaml py-numpy pfunit pflogger fargparse zlib-ng mepo udunits
```

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
cmake -B build -S . --install-prefix=$(pwd)/install --fresh -DCMAKE_Fortran_COMPILER=gfortran-14 -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
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
module use -a $SPACK_ROOT/share/spack/lmod/darwin-sequoia-aarch64/Core
module load apple-clang openmpi esmf python py-pyyaml py-numpy pfunit pflogger fargparse zlib-ng
module list
setenv DYLD_LIBRARY_PATH ${LD_LIBRARY_PATH}:${GEOSDIR}/lib
```

Note that `LMOD_PKG` seems to be defined by the brew lmod installation...though not sure.
You might just need to put in a full path or something.
