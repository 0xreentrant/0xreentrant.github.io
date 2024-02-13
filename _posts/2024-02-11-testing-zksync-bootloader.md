---
layout: distill
title: Testing the zkSync Era bootloader
date: 2024-02-08
description: What does it take to test the security of a blockchain's most important core smart-contract?
tags: 
categories: 
toc:
  - name: Remediations
  - name: Possible test suite
  - name: Logging capability
  - name: Exploring bootloader memory
  - name: Analyzing bootloader tooling
  - name: Assessing testing criteria
  - name: Criteria for testing the bootloader
  - name: Exploring tooling options
  - name: Roadblocks - single transaction mode
  - name: Logging woes
  - name: Results
---

> NOTE: This was performed during the the last week of the Code4rena contest. While the time constraints prevented us from fully completing the endeavor, we plan to follow up post-contest. 

Remediations are included at the start of this document for pragmatic readers, and what follows is an analysis of the tooling available for the bootloader to be tested during an audit. 

To motivate this exploration, testing the bootloader was done with the intent of embodying the persona of a new hire attempting to build a testing suite for a niche aspect of the zkSync system.   While the original intention was to build a fully functional test suite, the time allotted was too short to fully implement this vision.

## Remediations

At the end of all this time, several suggestions were considered for the Matter Labs team to address:
1. Provide an accessible, easy-to-update test suite, for the bootloader, to exercise both typical and edge-case transactions
2. Add easy-to-access logging capability for further development, and auditing, of the bootloader
3. Allow for accessible exploration of the bootloader memory throughout the execution of the bootloader

### Possible test suite 

For testing the bootloader, many kinds of scenarios can be formulated.  The general structure of the scenarios is the execution of batches.  In the the case that a batch contains a multitude of transactions, there is possibility of the state machine - that is implied by the bootloader's code graph and memory - could provide opportunities for the invariants of the system to be violated.  

It would be valuable to create tests that exercise the bootloader in manners that demonstrate the invariants hold. High-level descriptions of invariants to test against attacks would be:
- escalate transaction type to privileged upgrade
- premature batch closure
- skipped transactions
- update batch data after closing
- incorrect gas calculations
- wrong tx.origin, msg.sender generated
- priorityQueue reordering
- general overwrite transaction data within batch

### Logging capability

Put simply, providing easily accessible logging to the system through the test node would make exploration, development, and refactoring of the bootloader code a smoother process.  

### Exploring bootloader memory

Being able to peek into the memory of a running bootloader would be a powerful device.  Possible methods could be - with no attempt to rank how difficult it would be to implement:
1. Binary memory dumps: dump memory to files during execution so that a hex editor can be used to explore the data. Possible options could be to dump before or after each transaction, after a timeout, at every state change, etc.
2. Provide a hook/file/endpoint for an external debugger to connect to: the canonical example is the Chrome webtools, which famously can connect to an external host to debug process on other devices.  Likely there could be challenges in the control of the test node, pausing and resuming operation
3. Include an internal debugger

# Analyzing bootloader tooling

## Assessing testing criteria 

A survey of the provided codebase was made for any files related to the operation of the bootloader internals.  No testing suites were found to satisfy that criteria.

While examining the code of the bootloader itself, it was clear that the structure of this subsystem was unsuited to run-of-the-mill testing. This typically involves interacting with explicit interfaces defined in the code, including externally visible functions and constructors.  What was found is that the bootloader operates on memory loaded by the zkSync Era EVM (which we'll call "zkEVM" at times throughout this document). Additionally, there were no externally-visible functions, and no constructor.  

> NOTE: In the scope of the zkSync Era codebase, it is clear that the bootloader was coded in Yul to take advantage of the increased control over the resulting compiled code's performance.  The consequence here is that, for Yul, the limited tooling in the wider EVM-based ecosystem leaves a lot to be desired in terms of developer experience, including the ability to test and benchmark Yul codebases.

At this point, there were two major and relatively obvious paths that could be taken:
1. Existing tools could be used to load and manipulate the bootloader memory and its environment
2. The bootloader could be modified to provide external interfaces and a constructor to be used in popular testing frameworks

It was decided that controlling the execution environment would be the best bet. The main factors were:
- controlling the memory to the granularity required by the bootloader would require brittle and difficult-to-maintain, parallel data structures between the zkEVM and test suite
- modifying the code changed the attack surface compared to the original implementation
- any modifications would likely change the memory outlay and performance characteristics of the original implementation
### Criteria for testing the bootloader

During this time, the bootloader code itself was being analyzed for possible attack vectors. As a result, a handful of criteria were surfaced that could help explore these attacks:
1. Many transactions would need to be available to load into memory to explore execution paths in the case memory segregation was compromised
2. Full control over each transaction was necessary to explore cases where invariants across the zkEVM and bootloader execution were violated 
3. Logging and exploring the memory space was key
## Exploring tooling options

Initially the in-scope, zkSync Era EVM code was explored to be used for loading the bootloader and constructing its memory.  The understanding was that the zkEVM already had data structures and interfaces available that would make loading arbitrary transactions into the bootloader's memory simple.  

While the code was available to explore, it was determined that the necessary "plumbing" to quickly get a proof-of-concept would take more time to understand and modify than was available.  After discussing possibilities with the C4 sponsors, it was discovered that there was in fact a "test node" in development (era_test_node) that could provide a minimal environment for running the bootloader against individual transactions. This was quickly determined to be useful, and simple entry points and data structures were identified for controlling the bootloader memory:

```rust
// matter-labs/era-test-node, src/node.rs
324:    if !transactions_to_replay.is_empty() {
325:        let _ = node.apply_txs(transactions_to_replay);
326:    }
```

```rust
// matter-labs/era-test-node, src/node.rs
1204: pub fn run_l2_tx_inner(
... 
1245:         let tx: Transaction = l2_tx.clone().into();
1246: 
1247:         vm.push_transaction(tx);
```

```rust
// matter-labs/zksync-era, core/lib/types/src/l2/mod.rs
133: pub struct L2Tx {
```

### Roadblocks - single transaction mode

During discussions with one of the sponsors, it was made clear that the zkEVM used with the test node was configured to only run the bootloader with a single transaction at a time.  This would be a roadblock.  Scanning the code, it was determined that the solution was to include an existing VM setting:

```rust
// matter-labs/zksync-era, core/lib/multivm/src/interface/types/inputs/execution_mode.rs
8:    vm::VmExecutionMode::Bootloader
```

### Logging woes 

During the implementation of a rough proof-of-concept, it was made clear that there was no easy way to read the logs from the bootloader through the test node.  The two suggestions from a sponsor were:
1. Use the `log0` - `log4` events in Yul so logs could be caught by the test node. 
2. Send transactions to the `Console` system contract. 

For the first option, the output would be a series of hex-encoded values according to the event parameters.  For the second option, the test node would output the logged strings using when run with the correct command-line setting.  This would be the preferred option.  

Unfortunately, neither seemed to be a great option, since the existing `debugLog` calls in the existing bootloader would not work. A third option was to be forking the `zksync-era` crate that was imported into the project and adding `println!` calls for when the `debugLog` hook was triggered.
### Results

While there wasn't time to fully implement the plan, the intended plan of attack became:

1. Add a command line setting through `clap` to support loading a JSON file with multiple transactions and their fields
2. Parse the JSON with `serde` and load the data into `L2Tx` instances
3. Fill `transactions_to_replay` with the data
4. Switch the vm execution mode to `Bootloader` while this new command line setting is triggered
5. Fork and update the `zksync-era` crate to output logs in the `debugLog` hook