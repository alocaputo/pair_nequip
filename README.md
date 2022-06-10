# LAMMPS pair style for NequIP

This pair style allows you to use NequIP models in LAMMPS simulations.

*Note: MPI is not supported due to the message-passing nature of the network.*
## Changes

* Extended compatibility to float64 models
* Added virial computation


## Pre-requisites

* PyTorch or LibTorch >= 1.10.0 

## Usage in LAMMPS

```
pair_style	nequip
pair_coeff	* * deployed.pth <type name 1> <type name 2> ...
```
where `deployed.pth` is the filename of your trained model.

The names after the model path `deployed.pth` indicate, in order, the names of the NequIP model's atom types to use for LAMMPS atom types 1, 2, and so on. The number of names given must be equal to the number of atom types in the LAMMPS configuration (not the NequIP model!). 
The given names must be consistent with the names specified in the NequIP training YAML in `chemical_symbol_to_type` or `type_names`.

## Building LAMMPS with this pair style

### Download LAMMPS
```bash
git clone -b stable_29Sep2021_update2 --depth 1 https://github.com/lammps/lammps.git
```
or your preferred method.
(`--depth 1` prevents the entire history of the LAMMPS repository from being downloaded.)

### Download this repository
```bash
git clone git@github.com:alocaputo/pair_nequip
```

### Patch LAMMPS
#### Automatically
From the `pair_nequip` directory, run:
```bash
./patch_lammps.sh /path/to/lammps/
```

#### Manually
First copy the source files of the pair style:
```bash
cp /path/to/pair_nequip/*.cpp /path/to/lammps/src/
cp /path/to/pair_nequip/*.h /path/to/lammps/src/
```
Then make the following modifications to `lammps/cmake/CMakeLists.txt`:
- Change `set(CMAKE_CXX_STANDARD 11)` to `set(CMAKE_CXX_STANDARD 14)`
- Append the following lines:
```cmake
find_package(Torch REQUIRED)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
target_link_libraries(lammps PUBLIC "${TORCH_LIBRARIES}")
```

### Configure LAMMPS
If you have PyTorch installed:
```bash
cd lammps
mkdir build
cd build
cmake ../cmake -DCMAKE_PREFIX_PATH=`python -c 'import torch;print(torch.utils.cmake_prefix_path)'`
```
If you don't have PyTorch installed, you need to download LibTorch from the [PyTorch download page](https://pytorch.org/get-started/locally/). Unzip the downloaded file, then configure LAMMPS:
```bash
cd lammps
mkdir build
cd build
cmake ../cmake -DCMAKE_PREFIX_PATH=/path/to/libtorch
```
CMake will look for MKL and, optionally, CUDA and cuDNN. You may have to explicitly provide the path for your CUDA installation (e.g. `-DCUDA_TOOLKIT_ROOT_DIR=/usr/lib/cuda/`) and your MKL installation (e.g. `-DMKL_INCLUDE_DIR=/usr/include/`).

Pay attention to warnings and error messages.

**MKL:** If `MKL_INCLUDE_DIR` is not found and you are using a Python environment, a simple solution is to run `conda install mkl-include` or `pip install mkl-include` and append:
```
-DMKL_INCLUDE_DIR="$CONDA_PREFIX/include"
```
to the `cmake` command if using a `conda` environment, or
```
-DMKL_INCLUDE_DIR=`python -c "import sysconfig;from pathlib import Path;print(Path(sysconfig.get_paths()[\"include\"]).parent)"`
```
if using plain Python and `pip`.

**CUDA:** Note that the CUDA that comes with PyTorch when installed with `conda` (the `cudatoolkit` package) is usually insufficient (see [here](https://github.com/pytorch/extension-cpp/issues/26), for example) and you may have to install full CUDA seperately. A minor version mismatch between the available full CUDA version and the version of `cudatoolkit` is usually *not* a problem, as long as the system CUDA is equal or newer. (For example, PyTorch's requested `cudatoolkit==11.3` with a system CUDA of 11.4 works, but a system CUDA 11.1 will likely fail.)

### Build LAMMPS
```bash
make -j$(nproc)
```
This gives `lammps/build/lmp`, which can be run as usual with `/path/to/lmp -in in.script`. If you specify `-DCMAKE_INSTALL_PREFIX=/somewhere/in/$PATH` (the default is `$HOME/.local`), you can do `make install` and just run `lmp -in in.script`.
