

# Game Reverse Engineering Research Notes: IL2CPP, Memory Analysis, and ACTk Protection Research (TaskBarHero)

> Game Reverse Engineering Study Notes: IL2CPP, Memory Analysis, and ACTk Research

This project outlines a reverse engineering research workflow targeting Unity IL2CPP games, covering static analysis, dynamic memory observation, symbol restoration, data structure deduction, and behavioral analysis of the Anti-Cheat Toolkit (ACTk) protection mechanisms.

The focus of these notes is not on creating modifiers, but on consolidating a complete binary security research workflow into a reviewable, verifiable, and extensible technical document. The research perspective leans towards a Purple Team approach: understanding how attackers observe client-side states, while also discussing server authority, anti-debugging, integrity verification, and data trust boundaries from a defensive standpoint.

> Disclaimer: This project is intended solely for information security research, reverse engineering education, and defensive design discussions. Do not use this content to compromise game fairness, bypass commercial service restrictions, infringe on third-party rights, or engage in any unauthorized activities.
## Conclusion First

> This project successfully bypassed the Anti-Cheat Toolkit protection mechanisms and achieved tamper verification of game coin data. However, this research is strictly for binary security and reverse engineering purposes. Therefore, the following sections will only focus on technical consolidation of low-level concepts and framework architectures. This project does not provide complete Cheat Tables, fixed memory addresses, reproducible bypass scripts, or automated exploitation workflows that can be directly applied to specific commercial games.
> The following text information is generated and refined by AI.

<img width="461" height="152" alt="image" src="https://github.com/user-attachments/assets/62085a03-c6bb-45dd-b049-4eb51b04ff4b" />


**What can the Anti-Cheat Toolkit be used for?**

* Protect variables in memory.
* Protect and extend PlayerPrefs and binary files.
* Generate build code signatures for tamper checks.
* Detect non-Play Store installations on Android.
* Detect speed modifiers/accelerators.
* Detect time manipulation cheats.
* Detect 3 common cheat defense barriers.
* Detect external unknown managed components (code injection).
* Includes an ObscuredPrefs / PlayerPrefs editor.
## Research Scope

This research focuses on the following topics:

- Observing program structure under the Unity IL2CPP architecture
- The relationship between `GameAssembly.dll` and `global-metadata.dat`
- Interpreting classes, field offsets, and function addresses in `dump.cs`
- The role of Cheat Engine in dynamic analysis
- The data protection approach of ACTk `ObscuredTypes`
- Limitations of client-side data protection when facing dynamic analysis
- Deriving a more reasonable game security architecture from both offensive and defensive perspectives


## Background

If a Unity project uses the IL2CPP backend, the original C# IL is converted to C++ and then compiled into platform-native machine code. For example, core files commonly found on the Windows platform include:

- `GameAssembly.dll`: Native binary corresponding to the main logic
- `[GameName]_Data/Metadata/global-metadata.dat`: Classes, methods, strings, and metadata

This means traditional .NET tools cannot directly retrieve the complete C# logic as they would with a Mono build. Researchers typically need to first restore the structure via metadata, then combine it with disassembly and dynamic observation to understand the actual execution flow.

## Toolkit

| Tool | Purpose |
| --- | --- |
| Il2CppDumper | Restores class, method, and field structures from IL2CPP binaries and metadata |
| Cheat Engine | Dynamic memory observation, breakpoint tracking, disassembly, and register state analysis |
| Ghidra / IDA Free | Static disassembly, cross-reference tracking, and control flow analysis |
| Wireshark / Burp Suite | Can be used later for network layer behavior observation and packet boundary analysis |


## Phase 1: Static Analysis with IL2CPP Dump

> Tool (https://github.com/Perfare/Il2CppDumper)

The first step in static analysis is to understand what information can still be restored after IL2CPP converts the original C# project into a native binary.

### Key Files

`GameAssembly.dll` typically contains the machine code for the main game logic. It is not a standard .NET assembly, so one cannot expect tools like dnSpy to directly restore readable C#.

`global-metadata.dat` stores the metadata required by the IL2CPP runtime, such as class names, method names, field information, string references, and type descriptions. Reverse engineering tools utilize this data to assist in symbol restoration.

### Output: dump.cs

One common output from Il2CppDumper is `dump.cs`. This file looks like C#, but it is not source code and typically does not contain actual function implementations.

Its value lies in providing:

- Class / Struct names
- Namespace and inheritance relationships
- Field offsets
- Method names
- Method RVA / VA / Offset
- Partial generic type and nested type information

When reading `dump.cs`, it should be viewed as an index and a map, rather than compilable code. The actual logic still needs to be verified in native disassembly or runtime behavior.
<img width="718" height="372" alt="image" src="https://github.com/user-attachments/assets/bd8860a1-184a-4598-9544-bd8d482c0538" />


### Static Analysis Goals

The main goals of this stage:

- Find classes related to research targets, such as player data, currency, inventory, drop tables, battle states
- Establish a mapping between field names and memory offsets
- Flag suspicious methods, such as setters, constructors, serializers, validation routines
- Associate high-level semantics with low-level addresses

Example observation direction:

```text
Class: PlayerResource
Field: coins
Possible meaning: currency-like value
Next step: observe writes during runtime
```

These notes help narrow down the scope for subsequent dynamic analysis.

## Phase 2: Dynamic Memory Analysis

The core of dynamic analysis is observing how the program reads, writes, and transforms data during execution.

Here, Cheat Engine's role is not merely searching for values, but serving as a runtime debugger. Common analysis tasks include:

- Searching for candidate values
- Observing which instructions read or write to target addresses
- Checking registers and the call stack
- Cross-referencing method offsets in `dump.cs`
- Determining whether the data is plaintext, encrypted, cached, or a UI display value

### Typical Workflow

1. First, create value changes through controllable in-game actions.
2. Use appropriate data types to narrow down candidate addresses, such as 4-byte integers or floats.
3. Observe access/write instructions for candidate addresses.
4. Trace triggered instruction addresses back to modules and functions.
5. Cross-reference `dump.cs`, disassembly tools, and the runtime call stack.
6. Determine whether the address represents a real state, temporary state, display state, or encrypted state.

The key to this workflow is cross-validation. Single-value search results are easily misinterpreted; they must be viewed in conjunction with structure, instructions, call paths, and actual game behavior.

## Applied Case Study: Tactical Attack Workflow

Previous sections lean towards methodology; this section adds the actual progression route from this lab environment. It does not record fixed addresses, complete injection scripts, or specific modification parameters that can be directly applied to a specific game, but instead preserves the decision sequence of the attack chain, bottlenecks, pivot strategies, and defensive conclusions.

```text
[Attack Path Overview]

Chain A: protected numeric value analysis
memory scan
  -> access breakpoint
  -> observe packed 16-byte movement
  -> identify decode / validation lifecycle
  -> avoid pre-decode mutation
  -> observe safer post-validation state

Chain B: local data structure analysis
dump.cs symbol review
  -> identify container-returning routine
  -> inspect return-time registers
  -> dereference List<T> / array structure
  -> map object field layout
  -> evaluate client-side authority weakness
```

### Chain A: ACTk Protected Value Analysis

The first attack chain starts from "observable values on the screen," but the real breakthrough is not the numbers themselves, but the protection processes behind them.

#### Step 1: Trigger Controlled Value Changes

First, create repeatable value changes in-game, such as resource gains, consumption, or settlement. The goal of this step is not immediate modification, but to obtain stable observation samples.

Research records should include:

- Which actions trigger value changes
- The range before and after value changes
- Whether UI displays update in real-time
- Whether there is delayed synchronization or refresh behavior

#### Step 2: Scan and Classify Candidate Addresses

After scanning for candidate addresses with Cheat Engine, one should not immediately assume the candidate value is the real state. In experiments, directly modifying exposed values may cause states to be reset, reverted, or zeroed, which usually indicates that the value is merely a cache, display value, or inconsistent with internal validation data.

Key points for judgment in this step:

- Whether candidate addresses exist stably
- Whether modifications are overwritten by the next logic update
- Whether multiple similar values exist simultaneously
- Whether there are signs of encrypted values, fake values, or runtime keys

#### Step 3: Use Access Breakpoints to Find the Real Flow

Set access/write breakpoints on candidate addresses to observe which instruction block reads or writes to it. If 16-byte moves, XMM registers, stack temporary areas, and multiple fields appear together, it usually indicates the program is moving an entire block of protected state, rather than simply handling a single plaintext number.

```text
[Observation]
single displayed value
      │
      ▼
multiple memory fields move together
      │
      ▼
protected state is likely involved
```

In this experiment, the truly valuable discovery was not "which address can be modified," but "which location has not yet completed decryption and validation." This difference directly determines whether subsequent attempts will succeed, fail, zero out, or produce strange anomalous numbers.

#### Step 4: Avoid the Pre-Decode Trap

Initially, attempting to modify data right when it's loaded into a register easily forces plaintext into a process expecting ciphertext. Subsequent decode, XOR, bit shifting, or integrity checks will continue to process the incorrectly formatted data, ultimately leading to unpredictable results.

```text
[Failed Attempt Pattern]
protected block loaded
  -> premature mutation
  -> decode routine treats mutated bytes as protected data
  -> corrupted runtime value or validation failure
```

This pitfall is important because it illustrates that dynamic analysis cannot just look at "data being read"; it must also judge the data's stage in its lifecycle.

#### Step 5: Move Observation Toward Post-Validation State

The subsequent strategy shifts to observing the state after the decode/validation routine, such as before function return, before data is prepared to be written back, before UI rendering, or at locations where logic is about to use the runtime value.

In an authorized lab environment, such observation points can be used to verify:

- When protected data converts to a runtime value
- Whether integrity checks have been completed
- Which data the UI is using
- Which data the gameplay logic is using
- Whether the client holds excessive authority over that state

### Chain B: Local Rate / Drop Table Structure Analysis

The second attack chain does not start from a single value, but shifts to data structures. When certain results appear to be composed of local data tables, drop pools, lists, or weighting models rather than simple numbers, the analysis focus shifts from "searching values" to "understanding containers."

#### Step 1: Search Symbols in dump.cs

First, search for classes and methods related to data tables, drops, lists, rewards, weights, or configuration files in `dump.cs`. The goal is not just to find a specific field, but to establish a path from high-level semantics to low-level addresses.

Research records can include:

```text
Candidate class:
Candidate method:
Return type:
Related field:
RVA / Offset:
Reason for interest:
```

If a method returns `List<T>`, an array, or a custom data collection, it means subsequent analysis may need to examine container layouts in IL2CPP.

#### Step 2: Break Near Function Return

For methods suspected of returning data collections, observe the register states before and after function return. At this point, you can typically see:

- Returned object pointers
- Data collections about to be used by the caller
- Internal array or items pointers within the container
- Element object addresses

```text
[Return-Time Observation]
target routine
  -> builds or fetches data list
  -> prepares return object
  -> caller consumes list
```

This observation method is more robust than directly scanning weight numbers, as it connects "where the data comes from" with "who will use it."

#### Step 3: Dereference List<T> and Array Layers

In IL2CPP, `List<T>` is usually not directly equal to the first element. It has a multi-layer structure including container objects, internal arrays, array headers, element storage areas, or element pointers.

The conceptual path is as follows:

```text
List<T>
  -> internal items / array reference
  -> array header
  -> element storage
  -> object instance
  -> target field
```

A key takeaway from this experiment is: seeing an address in a register does not mean it is the target field. It must be verified layer by layer whether it is a container, array, element, or an internal field of an element.

#### Step 4: Map Field Offsets Back to dump.cs

Once the element object is found, go back to `dump.cs` to compare field order, types, and field offsets. This step prevents misidentifying adjacent fields as target fields and verifies whether the current dereference chain is logical.

Suggested verification questions:

- Does the current address fall within a reasonable object range?
- Does the field type match expectations?
- Do adjacent fields also match the structure in `dump.cs`?
- Do modification tests only affect expected behavior?
- Will the results be overwritten or rejected by the server?

#### Step 5: Derive the Security Finding

If local data tables or weight models can influence results, it means those results rely at least partially on client-side state. From a defensive perspective, this is not a problem of "which offset was found," but a design issue regarding authority boundaries.

```text
[Security Finding]
client-side configurable model
  -> local data structure can be observed
  -> local fields can influence runtime behavior
  -> high-value outcomes should be server-authoritative
```

### Lessons from the Tactical Flow

The most important aspect of this experiment is not a single technique, but the progression sequence:

- First, establish observation samples using repeatable behaviors
- Then, use breakpoints to find actual read/write paths
- When encountering protected data, understand the lifecycle before determining injection points
- When encountering container-type data, dereference the structure before discussing fields
- Finally, convert client-controllable results into defensive architecture findings

This also explains why the following sections emphasize `ObscuredTypes`, hook observation points, `List<T>` dereferencing, and server-authoritative design. The theory is not an afterthought decoration, but rules extracted from real-world bottlenecks.

## Phase 3: Understanding ACTk ObscuredTypes

The Anti-Cheat Toolkit (ACTk) is a common client-side anti-tamper suite in the Unity ecosystem. Its `ObscuredTypes` series of types prevents sensitive values from residing in memory as intuitive plaintext for extended periods.

Taking `ObscuredFloat` as an example, the actual data structure may include:

- encrypted value
- crypto key
- fake value / honeypot value
- hash or integrity check data
- initialized flag

Therefore, directly searching and modifying the numbers seen on screen may only be altering UI cache or intermediate values. If the internal ciphertext, key, hash, or validation process is inconsistent, the program may determine that the data has been tampered with, subsequently resetting values, rejecting states, or triggering other protection logic.

### Conceptual Memory Layout

Different ACTk versions, compilation settings, and game implementations may result in different structural details. Therefore, what follows is not a fixed offset table, but a conceptual model to understand `ObscuredTypes`.

```text
[Obscured Numeric Value: Conceptual Layout]
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ control / flags  │ runtime key       │ encrypted payload│ validation data  │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
          │                  │                  │
          │                  │                  └── encrypted or transformed value
          │                  └── dynamic key used during encode / decode
          └── initialization state, fake value marker, or integrity metadata
```

During dynamic analysis, it is common to see 16-byte moves, XMM registers, stack temporary areas, and struct fields appearing together. Such phenomena usually indicate that the program is not simply reading or writing a single `float` or `int`, but moving an entire block of protected data state.

### Why Plain Memory Editing Fails

Traditional memory editing often assumes:

```text
displayed value == real value in memory
```

But in ACTk types, it is closer to:

```text
displayed value = decrypt(encrypted value, key)
valid state = integrity_check(encrypted value, key, hash, fake value)
```

This means the research focus should shift from "finding numbers" to "understanding the data lifecycle":

- When is the value created?
- When is it encrypted?
- When is it decrypted?
- Where are consistency checks performed?
- Which fields are merely for display or cache?
- Which functions are the authoritative entry points for state changes?

### Practical Pitfall: Editing Before Decode

A common misjudgment in practice is jumping to conclusions right when data is loaded into a register. For example, seeing an instruction like "move 16 bytes to an XMM register" in one go intuitively leads one to assume the register already contains usable plaintext values.

However, for `ObscuredTypes`, what is obtained at this moment is likely still a combination of ciphertext data blocks, keys, flags, or validation data. Modifying it before the decryption or validation process is complete will cause subsequent algorithms to continue processing the incorrect data as a valid state, ultimately potentially resulting in several phenomena:

- Display values become fixed strange anomalous numbers
- Values change briefly and immediately revert
- Security validation zeroes out the data
- Game logic rejects the state
- The program throws an exception or crashes

```text
[Wrong Mental Model]
load value -> edit plaintext -> use value

[More Accurate Model]
load protected block -> decode / validate -> derive runtime value -> use value
```

This is why simply searching for screen numbers is usually unreliable. What is truly worth observing is the lifecycle of data transitioning from a "protected state" to an "operable state."

## Phase 4: Code Flow and Hooking Concepts

In binary security research, an Inline Hook is a technique used to observe or alter program control flow. The concept involves redirecting the execution flow at a specific instruction location, executing researcher-defined logic, and then returning to the original flow.

This project only discusses its research significance and defensive implications; it does not include complete injection scripts that can be directly applied to specific targets.

### Conceptual Model

```text
original function
    -> prepare data
    -> validate or transform data
    -> write result
    -> return

instrumented flow
    -> prepare data
    -> validate or transform data
    -> observe selected registers / memory
    -> optionally test controlled behavior in a lab environment
    -> return to original flow
```

From a research perspective, the value of hooking lies in understanding the program's actual state when "data is about to be written" or "a function is about to return." This assists in determining:

- Which data is plaintext before encryption
- Which data is ciphertext after encryption
- Whether keys and values coexist in registers or the stack
- Whether anti-tamper checks occur before writing, after writing, or during the next read

### Better Observation Points

If the goal is to research the data lifecycle, choosing the injection point is more important than the modification content itself. Common observation points include:

- Getter / setter entry points
- Before and after decode / encode routines
- Before and after integrity checks
- Before function return
- Before UI rendering or network serialization

```text
[Protected Value Lifecycle]
encrypted state
      │
      ▼
decode / validate
      │
      ▼
runtime value
      │
      ├── gameplay calculation
      ├── UI rendering
      └── network serialization
```

In a lab environment, researchers typically first observe the register, stack, and object field states at each node, then determine which segment is where the security mechanism actually takes effect. This approach is more stable than directly modifying data at the first suspicious instruction and is more likely to yield interpretable conclusions.

### C# Container Dereferencing

IL2CPP converts high-level C# containers into operable object structures in the native runtime. When analyzing `List<T>`, arrays, or custom data tables, it is often necessary to understand multi-layer pointer dereferencing.

The following is a conceptual model and does not imply that all versions and games use the same offsets:

```text
[List<T> Conceptual Dereference Chain]
List<T> object
      │
      └── internal array / items pointer
              │
              └── array header
                      │
                      └── element pointer or inline element storage
                              │
                              └── target object fields
```

This analysis method can be used to answer several questions:

- Does the function return the container itself, or elements within the container?
- Does the current register hold an object address, an array address, or a field value?
- Is the target field value type inline storage, or a reference type object?
- Do the field offsets match the structural descriptions in `dump.cs`?

The research value of container dereferencing lies in understanding data structures, rather than blindly chasing a fixed address. Fixed offsets easily become invalid due to version, platform, compiler, and data model changes.

### Defensive Takeaway

Tools like ACTk can increase the cost of client-side modification, but they cannot turn the client into a trusted environment. As long as important data is calculated or determined locally, attackers have the opportunity to observe data flows through dynamic analysis.

Therefore, security design should avoid leaving critical results entirely up to the client.

## Purple Team Findings (AI-Provided Recommendations)

This research can summarize several common focus points for both offense and defense.

### 1. Client-side memory is observable

Regardless of whether data is protected by XOR, hash, fake values, or runtime keys, as long as the data needs to be used on the client side, it will be loaded into registers, the stack, or the heap at some point.

Defenders cannot assume that "encrypted data residing in memory" equates to security; it should only be viewed as raising the barrier for analysis.

### 2. Server-authoritative design is critical

High-value data such as currency, drops, gacha, leaderboards, transactions, experience points, and paid items should have their final results determined by the server.

Safer design:

```text
client sends intent
server validates rules
server computes result
server returns signed state
client displays result
```

Higher-risk design:

```text
client computes result
client stores result
server accepts client state
```

### 3. Anti-cheat should be layered

Single protection mechanisms are easily bypassed or observed. A more reasonable approach is multi-layered implementation:

- Server-side validation
- Rate limiting
- Replay protection
- Session tokens and request signing
- Runtime integrity checks
- Anti-debugging
- Obfuscation
- Telemetry-based anomaly detection
- Economy and gameplay sanity checks

The point is not to make every layer perfect, but to form an overall defense line where attack cost, detection capability, and remediation speed work together.

### 4. Logs and telemetry matter

If the system only blocks on the client side and the server lacks sufficient telemetry, defenders will struggle to understand attack patterns.

Recommended observations:

- Unreasonable resource growth rates
- Deviations between drop results and probability models
- Anomalous level completion times
- High-frequency retries or failed requests
- Same device, multiple accounts, anomalous session patterns
- Correlations between client versions, integrity check results, and behavioral data




## Summary

These research notes demonstrate the analysis methods for Unity IL2CPP games at the binary level: first restoring high-level structures via metadata, then using dynamic tools to observe runtime behavior, and finally deriving client-side security boundaries from the protection models of types like ACTk.

The most important conclusion is: client-side protection can increase attack costs but cannot replace server authority. Truly robust game security design requires placing critical assets, probabilistic results, and state settlements on a trusted backend, and continuously validating overall risk through layered defenses and telemetry.
