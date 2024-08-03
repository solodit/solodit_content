**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## High Risk

### Dependency: Loop With Unreachable Exit Condition

**Severity**: High

**Status**: Resolved

**Dependency**: github.com/regen-network/protobuf:v1.3.3-alpha.regen.1

**CWE**: 835

**CVE**: 2024-24786

**Details**: 

In the package `google.golang.org/protobuf` versions prior to 1.33.0, the "protojson.Unmarshal" function can enter an infinite loop when unmarshaling certain forms of invalid JSON. This condition can occur when unmarshaling into a message which contains a "google.protobuf.Any" value, or when the "UnmarshalOptions.DiscardUnknown" option is set. This package is an indirect dependency by the "github.com/regen-network/protobuf:v1.3.3-alpha.regen.1"

**Recommended strategy**: 

Update the dependency

### Dependency: Uncontrolled Resource Consumption

**Severity**: High

**Status**: Unresolved

**Dependency**: github.com/dvsekhvalnov/jose2go:v1.5.0

**CWE**: 400

**CVE**: 2023-50658

**Details**: 

The `jose2go` component prior 1.6.0 for Go allows attackers to cause a denial of service (CPU consumption) via a large p2c (aka PBES2 Count) value.

This package is an indirect dependency by multiple modules, including the "github.com/cosmos/keyring:v1.2.0"

**Recommended strategy**: 

Update the dependencies

**Client comment**: 

We'll have to wait for upstream libraries to bump this

## Medium Risk

### Path Traversal

**Severity**: Medium

**Status**: Acknowledged

**File**: client/cli/flags.go:557

**CWE**: 22

**Details**: 

The `ReadProposalFlag` function reads the file contents out of the filesystem, given by the variable propFN, into a `propFileContents` variable. However, this function doesn't check the path, and it's possible to point to any file in the filesystem.

**Recommended strategy**: 

Input Validation

**Client comment**: 

That function is supposed to be able to read any file that the file system allows the process to read. It's the OS's job to determine that and our stuff should not have any opinion on it

## Low Risk

### Code duplication

**Severity**: Low

**Status**: Acknowledged

**File**: client/cli/query_setup.go:293:371

**CWE**: 1041

**Details**:

Two functions, "SetupCmdQueryGetMarket" and "SetupCmdQueryValidateMarket," have an identical code.

**Recommendation**:

Externalize the implementation and call a separate function from the given ones.
**Client comment**:  The two functions serve different purposes. They have the same code right now, but the business logic of the two isn't related, so it's okay to have two similar code paths.
 
### Code duplication

**Severity**: Low

**Status**: Acknowledged

**File**: query.pb.gw.go:124:200

**CWE**: 1041

**Details**:

Two functions, "request_Query_GetOrderByExternalID_0" and "request_Query_GetOrderByExternalID_1," have an identical code.

**Recommendation**:

Externalize the implementation and call a separate function from the given ones.
**Client comment**:  there's nothing we can do about that. Variables are already used for any strings that are needed in two or more places.

###  Code duplication

**Severity**: Low

**Status**: Acknowledged

**File**: query.pb.gw.go:162:238

**CWE**: 1041

**Details**:

Two functions, "local_request_Query_GetOrderByExternalID_0" and "local_request_Query_GetOrderByExternalID_1," have an identical code.


**Recommendation**:

Externalize the implementation and call a separate function from the given ones.
**Client comment**:  there's nothing we can do about that. Variables are already used for any strings that are needed in two or more places.

### Code duplication

**Severity**: Low

**Status**: Acknowledged

**File**: query.pb.gw.go:744:798

**CWE**: 1041

**Details**:

Two functions, "request_Query_ValidateMarket_0" and "request_Query_ValidateMarket_1," have an identical code.

**Recommendation**:

Externalize the implementation and call a separate function from the given ones.
**Client comment**:  there's nothing we can do about that. Variables are already used for any strings that are needed in two or more places.

### Code duplication

**Severity**: Low

**Status**: Acknowledged

**File**: query.pb.gw.go:771:825

**CWE**: 1041

**Details**:

Two functions, "local_request_Query_ValidateMarket_0" and "local_request_Query_ValidateMarket_1," have an identical code.

**Recommendation**:

Externalize the implementation and call a separate function from the given ones.
**Client comment**:  there's nothing we can do about that. Variables are already used for any strings that are needed in two or more places.


## Informational

### Constant could be used

**Severity**: Informational

**Status**: Acknowledged

**File**: multiple

**CWE**: 400

**Details**:

Multiple text messages are using the same string. It is much more efficient to use constants to declare strings.

**Recommendation**:

Declare text messages in constants and then re-use them as needed.


If a comprehensive list of files and strings is needed, we'll provide it by request.

**Client comment**: 

I'm guessing it's referring to portions of error messages, which are better left as they are.

### Cognitive Complexity

**Severity**: Informational

**Status**: Acknowledged

**File**: multiple

**CWE**: 1120

**Details**:

Multiple functions have cognitive complexity up to 173, while the recommended value is 15.

**Recommendation**:

Decrease cognitive complexity to be at most 50.
If a comprehensive list of files and functions is needed, we'll provide it by request.

**Client comment**: 

isn't really addressable without major over-engineering.
