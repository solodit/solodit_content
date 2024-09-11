**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk


### Double Public Key Signing Function Oracle Attack on ed25519-dale

**Severity**: High

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

Versions of ed25519-dalek prior to v2.0 model private and public keys as separate types which can be assembled into a Keypair, and also provide APIs for serializing and deserializing 64-byte private/public keypairs.
Such APIs and serializations are inherently unsafe as the public key is one of the inputs used in the deterministic computation of the S part of the signature, but not in the R value. An adversary could somehow use the signing function as an oracle that allows arbitrary public keys as input can obtain two signatures for the same message sharing the same R and only differ on the S part.
Unfortunately, when this happens, one can easily extract the private key.
Revised public APIs in v2.0 of ed25519-dalek do NOT allow a decoupled private/public keypair as signing input, except as part of specially labeled ""hazmat"" APIs which are clearly labeled as being dangerous if misused.

**Recommendation**: 

Update ed25519-dalek library to the version >=2		

### libsecp256k1 allows overflowing signatures

**Severity**: High

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

libsecp256k1 accepts signatures whose R or S parameter is larger than the secp256k1 curve order, which differs from other implementations. This could lead to invalid signatures being verified.
The error is resolved in 0.5.0 by adding a check_overflow flag.


**Recommendation**: 

Update libsecp256k1 library to the version >=0.5.0	


### Multiple soundness issues in `owning_ref`

**Severity**: High

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:
- OwningRef::map_with_owner is unsound and may result in a use-after-free.
- OwningRef::map is unsound and may result in a use-after-free.
- OwningRefMut::as_owner and OwningRefMut::as_owner_mut are unsound and may result in a use-after-free.
- The crate violates Rust's aliasing rules, which may cause miscompilations on recent compilers that emit the LLVM noalias attribute.
safer_owning_ref is a replacement crate which fixes these issues. No patched versions of the original crate are available, and the maintainer is unresponsive.

**Recommendation**: 

No fixed upgrade is available! Avoid using of the lib. 

## Informational

### `aes-soft` has been merged into the `aes` crate

**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

Please use the aes crate going forward. The new repository location is at:
https://github.com/RustCrypto/block-ciphers/tree/master/aes
AES-NI is now autodetected at runtime on i686/x86-64 platforms. If AES-NI is not present, the aes crate will fallback to a constant-time portable software implementation.
To force the use of a constant-time portable implementation on these platforms, even if AES-NI is available, use the new force-soft feature of the aes crate to disable autodetection.

**Recommendation**:
Use `aes` crate.	

 
 
### `aesni` has been merged into the `aes` crate

**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

Please use the aes crate going forward. The new repository location is at:
https://github.com/RustCrypto/block-ciphers/tree/master/aes
AES-NI is now autodetected at runtime on i686/x86-64 platforms. If AES-NI is not present, the aes crate will fallback to a constant-time portable software implementation.
To prevent this fallback (and have absence of AES-NI result in an illegal instruction crash instead), continue to pass the same RUSTFLAGS which were previously required for the aesni crate to compile:
RUSTFLAGS=-Ctarget-feature=+aes,+ssse3

**Recommendation**: 

Use `aes` crate	

### ansi_term is Unmaintained

**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

The maintainer has advised that this crate is deprecated and will not receive any maintenance.
The crate does not seem to have much dependencies and may or may not be ok to use as-is.
Last release seems to have been three years ago.

Recomm****endation: 

Here is the list of possible alternatives: ansiterm, anstyle, console, nu-ansi-term, owo-colors, stylish, yansi	

### `cpuid-bool` has been renamed to `cpufeatures`

**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

Please use the `cpufeatures` crate going forward:
https://github.com/RustCrypto/utils/tree/master/cpufeatures
There will be no further releases of cpuid-bool.

**Recommendation**: 

Use the `cpufeatures` crate.	

### mach is unmaintained

**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

Last release was almost 4 years ago.
Maintainer(s) seem to be completely unreachable.

**Recommendation**: 

Possible alternative: mach2

### Crate `parity-wasm` deprecated by the author

**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

This PR explicitly deprecates parity-wasm. The author recommends switching to wasm-tools.

**Recommendation**: 

Use the `wasm-tools` crate		

### stdweb is unmaintained
**Severity**: Informational

**Status**: Acknowledged

**File**: Cargo.lock

**Details**:

The author of the stdweb crate is unresponsive.

**Recommendation**: 

Possible alternatives: wasm-bindgen, js-sys, web-sys	
