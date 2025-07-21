---
title: "PEEPanEIP-3540/3670: EVM Object Format (EOF) v1"
type: slide
slideOptions:
  theme: league
  allottedMinutes: 30
---

### EVM Object Format (EOF)

- [EIP-3541: Reject new contract code starting with the 0xEF byte](https://eips.ethereum.org/EIPS/eip-3541) 
- [EIP-3540: EVM Object Format (EOF) v1](https://eips.ethereum.org/EIPS/eip-3540)
- [EIP-3670: EOF - Code Validation](https://eips.ethereum.org/EIPS/eip-3670)
- [EIP-4200: Static relative jumps](https://eips.ethereum.org/EIPS/eip-4200)
- [EIP-4750: EOF - Functions](https://eips.ethereum.org/EIPS/eip-4750)

---

### EVM Object Format (EOF)

- Live since London:
  - [EIP-3541: Reject new contract code starting with the 0xEF byte](https://eips.ethereum.org/EIPS/eip-3541) 
- Shanghai:
  - [EIP-3540: EVM Object Format (EOF) v1](https://eips.ethereum.org/EIPS/eip-3540) 
  - [EIP-3670: EOF - Code Validation](https://eips.ethereum.org/EIPS/eip-3670)
- More:
  - [EIP-4200: Static relative jumps](https://eips.ethereum.org/EIPS/eip-4200)
  - [EIP-4750: EOF - Functions](https://eips.ethereum.org/EIPS/eip-4750)

---

### EVM Object Format (EOF)

Shanghai:
- [EIP-3540: EVM Object Format (EOF) v1](https://eips.ethereum.org/EIPS/eip-3540)
- [EIP-3670: EOF - Code Validation](https://eips.ethereum.org/EIPS/eip-3670)

---

### Motivation

Address current issues with EVM
- Introducing new instructions
- Instruction deprecation
- Data bytes and JUMPs
- Data sections

---

### Motivation

**Versioning**

- Provides **extensibility**: new features should be easy to introduce, obsolete features should be easy to remove, and distruption to existing code should be reasonable.
- Don't require changes to account structure.

---

### Motivation

**Deploy-time validation**

- Provides useful guarantee: only valid code is present on-chain.
 

---

### EIP-3540: Container specification

Layout:
```
magic, version                      \   EOF
(section_kind, section_size)+, 0,   /  header
<section contents>
```

---

### EIP-3540: Container specification

| description | length   | value      | |
|-------------|----------|------------|-|
| magic       | 2-bytes  | 0xEF00     | |
| version     | 1-byte   | 0x01–0xFF  | EOF version number |

---

### EIP-3540: Container specification

- Section header
    - Section kind (8-bit unsigned number)
    - Section size (16-bit unsigned number)
- Section contents

---

### EIP-3540: Validation rules

1. `version` MUST NOT be `0`.
2. `section_kind` MUST NOT be `0`. 
3. `section_size` MUST NOT be `0`.
4. There MUST be at least one section (and therefore section header).
5. Section data size MUST be equal to `section_size` declared in its header.
6. Stray bytes outside of sections MUST NOT be present.

---

### EIP-3540: EOF version 1

- Exactly one code section MUST be present.
- The code section MUST be the first section.
- A single data section MAY follow the code section.

> Note: this implies that data-only contracts are not allowed.

---

### EIP-3540: Example bytecode

Legacy code:
<pre>
600480600B6000396000F3600080FD
</pre>

EOF code:
<pre>
EF000101000B0200040060048060156000396000F3600080FD
</pre>

---

### EIP-3540: Example bytecode

Legacy code:
<pre>
600480600B6000396000F3600080FD
</pre>

EOF code:
<pre>
EF000101000B02000400 60048060156000396000F3 600080FD
</pre>

---

### EIP-3540: Example bytecode

<pre>
<span style="color:red">EF00</span>0101000B02000400 60048060156000396000F3 600080FD
</pre>

These bytes are the magic.

---

### EIP-3540: Example bytecode

<pre>
EF00<span style="color:red">01</span>01000B02000400 60048060156000396000F3 600080FD
</pre>

The version number, which is set at 1 currently.

---

### EIP-3540: Example bytecode

<pre>
EF0001<span style="color:red">01000B</span>02000400 60048060156000396000F3 600080FD
</pre>

The first section, with kind `01` meaning *code*, and length of `0xB` (11).

---

### EIP-3540: Example bytecode

<pre>
EF000101000B<span style="color:red">020004</span>00 60048060156000396000F3 600080FD
</pre>

The (optional) second section, with kind `02` meaning *data*, and length of `0x4`.

---

### EIP-3540: Example bytecode

<pre>
EF000101000B020004<span style="color:red">00</span> 60048060156000396000F3 600080FD
</pre>

The terminator for the header (i.e. section kind 0).

---

### EIP-3540: Example bytecode

<pre>
EF000101000B02000400 <span style="color:red">60048060156000396000F3</span> 600080FD
</pre>

This is the content of the first section (*the code*):

```
PUSH 4
DUP1
PUSH 21
PUSH 0
CODECOPY
PUSH 0
RETURN
```

---

### EIP-3540: Example bytecode

<pre>
EF000101000B02000400 60048060156000396000F3 <span style="color:red">600080FD</span>
</pre>

This is the content of the second section (*the data*):

```
PUSH 0
DUP1
REVERT
```

In our case we actually encoded the runtime code here.

---

### EIP-3670: Code Validation

Additional code validation rules:

1. Deploying undefined opcodes is forbidden.
2. Code must end with a terminator instruction (STOP, RETURN etc.)
> Implication: PUSH at the end of code without enough data is forbidden (no implicit zero bytes)

---

### EIP-3670: Code Validation

Easy to deprecate SELFDESTRUCT, CALLCODE etc.

---

### EIP-3670: Code Validation

Gas charge for code validation?
- Deployed code already covered with 200 gas per byte.
- [EIP-3860: Limit and meter initcode](https://eips.ethereum.org/EIPS/eip-3860) introduces gas charge for validation of initcode (2 gas per 32-byte word).

---

### EIP-3540/3670: Creation rules

- If initcode’s container has EOF1 prefix it must be valid EOF1 code.
- If code’s container has EOF1 prefix it must be valid EOF1 code.

---

### EIP-3540/3670: Creation rules

- For a create transaction, if initcode or code is invalid, the contract creation results in an exceptional abort. 
    - Such a transaction is valid and may be included in a block.
- For the `CREATE` and `CREATE2` instructions, if initcode or code is invalid, instructions’ execution ends with the result 0 pushed on stack.

---

### EIP-3540/3670: Creation rules

- EOF _initcode_ can create EOF _code_
<pre>
EF000101000B02000400 60048060156000396000F3 EF00010100600080FD
</pre>

- ~~EOF _initcode_ can create legacy _code_~~
<pre>
EF000101000B02000400 60048060156000396000F3 600080FD
</pre>

- Legacy _initcode_ can create EOF _code_

- Legacy _initcode_ can create legacy _code_


---

### EIP-3540: Execution rules

- `JUMP`s are allowed only inside code section
- `PC` is relative to code section (starts from 0)
- `CODECOPY`/`EXTCODECOPY` can be used to copy from data section

---

### Future EOF proposals

- [EIP-4200: Static relative jumps](https://eips.ethereum.org/EIPS/eip-4200)
- [EIP-4750: EOF - Functions](https://eips.ethereum.org/EIPS/eip-4750)
- [EIP-3779: Safer Control Flow for the EVM](https://eips.ethereum.org/EIPS/eip-3779)

---

### Learn more

[Everything about the EVM Object Format (EOF)](https://notes.ethereum.org/JLrlecMISYS7xT0JaNRNUw)
