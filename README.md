### Some initial states

<details>
<summary>lscpu</summary>

```bash
~ » lscpu

Architecture:            aarch64
  CPU op-mode(s):        32-bit, 64-bit
  Byte Order:            Little Endian
CPU(s):                  4
  On-line CPU(s) list:   0-3
Vendor ID:               ARM
  Model name:            Neoverse-N1
    Model:               1
    Thread(s) per core:  1
    Core(s) per cluster: 4
    Socket(s):           -
    Cluster(s):          1
    Stepping:            r3p1
    BogoMIPS:            50.00
    Flags:               fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm lrcpc dcpop asimddp ssbs
NUMA:                    
  NUMA node(s):          1
  NUMA node0 CPU(s):     0-3
Vulnerabilities:         
  Gather data sampling:  Not affected
  Itlb multihit:         Not affected
  L1tf:                  Not affected
  Mds:                   Not affected
  Meltdown:              Not affected
  Mmio stale data:       Not affected
  Retbleed:              Not affected
  Spec store bypass:     Mitigation; Speculative Store Bypass disabled via prctl
  Spectre v1:            Mitigation; __user pointer sanitization
  Spectre v2:            Mitigation; CSV2, BHB
  Srbds:                 Not affected
  Tsx async abort:       Not affected
```
</details>
<details>
<summary>rustup</summary>

```bash
~ » rustup -V
rustup 1.26.0 (5af9b9484 2023-04-05)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: The currently active `rustc` version is `rustc 1.72.0 (5680fa18f 2023-08-23)`
```
</details>
<details>
<summary>gcc</summary>

```bash
~ » gcc -v

Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/aarch64-linux-gnu/11/lto-wrapper
Target: aarch64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 11.4.0-1ubuntu1~22.04' --with-bugurl=file:///usr/share/doc/gcc-11/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-11 --program-prefix=aarch64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-libquadmath --disable-libquadmath-support --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --enable-fix-cortex-a53-843419 --disable-werror --enable-checking=release --build=aarch64-linux-gnu --host=aarch64-linux-gnu --target=aarch64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=2
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)
```
</details>

### Let's build

```bash
git clone -b testnet-alpha-0 https://github.com/fleek-network/lightning.git ~/fleek-network/lightning
cd fleek-network/lightning
cargo clean && cargo update
cargo +stable build --release 
```
we see the following
```bash
error: linking with `cc` failed: exit status: 1
# ... wall of text
# ... the key error is ↴
= note: /usr/bin/ld: /home/ubuntu/fleek-network/lightning/target/release/deps/libblake3-a927e9b36d695ff0.rlib(blake3-a927e9b36d695ff0.blake3.91a53ea05847a7a5-cgu.0.rcgu.o): in function `blake3_compress_in_place_portable':
          /home/ubuntu/.cargo/registry/src/index.crates.io-6f17d22bba15001f/blake3-1.4.1/src/ffi_neon.rs:45: multiple definition of `blake3_compress_in_place_portable'; /home/ubuntu/fleek-network/lightning/target/release/deps/libfleek_blake3-990c4c0cfb4eaa87.rlib(fleek_blake3-990c4c0cfb4eaa87.fleek_blake3.4f11e9370af31773-cgu.0.rcgu.o):/home/ubuntu/.cargo/registry/src/index.crates.io-6f17d22bba15001f/fleek-blake3-1.4.1/src/ffi_neon.rs:45: first defined here
```

As a result of the research, after discarding the unrelated variants and testing we find out that gcc 11 does not allow multiple definition (or something like that), ok, let's allow it.

```bash
RUSTFLAGS="-Clink-arg=-Wl,--allow-multiple-definition" cargo +stable build --release
```
Now we get this:
```bash
warning: Linking globals named 'blake3_compress_in_place_portable': symbol multiply defined!

error: failed to load bitcode of module "fleek_blake3-990c4c0cfb4eaa87.fleek_blake3.4f11e9370af31773-cgu.0.rcgu.o":
```
Fixing `lto` in Cargo.toml (maybe it possible with `RUSTFLAGS` idk), change `lto = true` to `lto = "thin"`
```
vim Cargo.toml
...
 68 [profile.release]
 ...
 71 # currently enabled, may increase build time, but runtime faster, can set to `"thin"`.
 72 lto = "thin"
 ...
```
Rebuild it
```bash
RUSTFLAGS="-Clink-arg=-Wl,--allow-multiple-definition" cargo +stable build --release 
...
Finished release [optimized + debuginfo] target(s) in 19m 39s
...
lightning-node -V
lightning-node 0.1.0

lightning-node help
Usage: lightning-node [OPTIONS] <COMMAND>

Commands:
  run           Start the node
  keys          Handle keys
  print-config  Print the loaded configuration
  help          Print this message or the help of the given subcommand(s)

Options:
  -c, --config <CONFIG>      Path to the toml configuration file [default: ~/.lightning/config.toml]
      --with-mock-consensus  Determines that we should be using the mock consensus backend
  -v...                      Increases the level of verbosity (the max level is -vvv)
      --log-location         Print code location on console logs
  -h, --help                 Print help
  -V, --version              Print version
```
Done. 

Cross-compilation `x86-64` >> `aarch64` is also possible but has its own specific aspects.
