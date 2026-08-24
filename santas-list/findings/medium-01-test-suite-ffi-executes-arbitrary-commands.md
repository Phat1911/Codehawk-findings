Title: Test suite executes arbitrary local commands through Foundry FFI
Impact: Medium
Likelihood: Medium
Scope: test/unit/SantasListTest.t.sol

# Root + Impact

## Description

Tests should be deterministic and should not execute arbitrary commands on the machine running the repository. Developers and auditors commonly run test suites during review, CI, or local setup.

The test suite includes `testPwned()`, which constructs a command array and passes it to Foundry's `ffi` cheatcode. In the current code, it executes `touch youve-been-pwned`. This demonstrates arbitrary command execution. A malicious change could replace this with commands that read secrets, alter files, install malware, or open network connections.

```solidity
function testPwned() public {
    string[] memory cmds = new string[](2);
    cmds[0] = "touch";
    cmds[1] = string.concat("youve-been-pwned");
    // @> Executes an arbitrary local command via FFI
    cheatCodes.ffi(cmds);
}
```

## Risk

**Likelihood**:

* This occurs whenever someone runs the test suite with Foundry FFI enabled.

* Reviewers may enable FFI to satisfy project tests or copied commands without inspecting every test body first.

**Impact**:

* The repository can execute arbitrary commands on the developer, auditor, or CI machine.

* Sensitive data, local files, environment variables, or CI secrets could be exposed or modified if the command is changed maliciously.

## Proof of Concept

The existing test is already a PoC:

```solidity
function testPwned() public {
    string[] memory cmds = new string[](2);
    cmds[0] = "touch";
    cmds[1] = string.concat("youve-been-pwned");
    cheatCodes.ffi(cmds);
}
```

Run tests with FFI enabled:

```bash
forge test --ffi --match-test testPwned
```

The command creates a local file named `youve-been-pwned`, proving that the test suite can execute commands on the host machine.

## Recommended Mitigation

Remove the FFI-based test and avoid FFI unless it is strictly required and reviewed. If FFI is required, isolate it behind clearly documented scripts and do not run it in default tests or CI.

```diff
-function testPwned() public {
-    string[] memory cmds = new string[](2);
-    cmds[0] = "touch";
-    cmds[1] = string.concat("youve-been-pwned");
-    cheatCodes.ffi(cmds);
-}
```

Also avoid enabling FFI by default in project configuration or CI.
