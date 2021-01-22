# Patches

Patches created on top of v5.10-rc6 (b65054597872ce3aefbc6a666385eabdf9e288da).

```
git checkout v5.10-rc6
git am path/to/patches/*.patch
```

# Configuration and build

```
make defconfig

# Processor type and features -> disable Symmetric multi-processing support
make menuconfig

make
```
