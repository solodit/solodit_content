**Auditors**

[Zokyo](https://x.com/zokyo_io)

# Findings

## Informational

### Checks for nested if statements which can be collapsed by &&-combining their conditions.

**Description**

club-program/src/instructions/update_voter_weight.rs
Line. 80-84. Each if-statement adds one level of nesting, which makes code look more
complex than it really is.

**Recommendation**:

Should be written: if x && y { ...}

### Checks for usage of .clone() on a Copy type.

**Description**

club-program/src/instructions/create_club.rs:
Lines. 116, 117. The only reason Copy types implement Clone is for generics, not for using the
clone method on a concrete type.

**Recommendation**:

Better to use *ctx.program_id

### Checks for address of operations (&) that are going to be dereferenced immediately by the compiler.

**Description**

club-program/src/instructions/support_club.rs
Line. 101. Suggests that the receiver of the expression borrows the expression.

**Recommendation**:

Change this to ctx.program_id

### Checks for address of operations (&) that are going to be dereferenced immediately by the compiler.

**Description**

club-program/src/instructions/create_club.rs:
Line. 161. Suggests that the receiver of the expression borrows the expression.

**Recommendation**:

Change this to &ctx.accounts.payer.key()

### Checks for address of operations (&) that are going to be dereferenced immediately by the compiler.

**Description**

club-program/src/instructions/create_treasury.rs
Line. 74. Suggests that the receiver of the expression borrows the expression.

**Recommendation**:

Change this to ctx.program_id

### Checks for address of operations (&) that are going to be dereferenced immediately by the compiler.

**Description**

club-program/src/instructions/create_proposal.rs
Line. 123, 132. Suggests that the receiver of the expression borrows the expression.
club-program/src/instructions/allow_member.rs
Line. 66, 102. Suggests that the receiver of the expression borrows the expression.
club-program/src/instructions/accept_member.rs
Line. 41. Suggests that the receiver of the expression borrows the expression.
club-program/src/instructions/trunsfer_funds.rs
Line. 69, 114. Suggests that the receiver of the expression borrows the expression.
club-program/src/instructions/execute_transaction.rs
Line. 166, 178, 179. Suggests that the receiver of the expression borrows the expression.
club-program/src/state/treasury.rs
Line. 26. Suggests that the receiver of the expression borrows the expression.

**Recommendation**:

Remove (&)

### Checks for usage of .clone() on a Copy type.

**Description**

club-program/src/instructions/allow_member.rs, Line. 84, 87, 88, 89.
club-program/src/instructions/update_voter_weight.rs, Line. 123
The only reason Copy types implement Clone is for generics, not for using the clone method
on a concrete type.

**Recommendation**:

Remove clone()
