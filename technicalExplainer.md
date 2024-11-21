# FullInliner Non-Expression Split Argument Evaluation Order Bug in Solidity and Yul #
## Introduction ##
This is a technical explainer of the FullInliner Non-Expression Split Argument Evaluation Order bug in Solidity, which I was asked to present and explain recently for an interview with Immunefi's Robert Chen.

## The Role of Yul in Solidity Compilation ##
Before explaining the bug, it’s important to understand Yul, the low-level intermediate representation used by the Solidity compiler during the optimization process. Yul provides a simplified abstraction of the EVM bytecode, allowing for more efficient optimizations compared to directly manipulating high-level Solidity. The Yul-based optimization pipeline introduces multiple steps, such as the ExpressionSplitter and FullInliner, which transform and optimize code. While these steps aim to improve efficiency, their interactions occasionally lead to unintended consequences, as was the case with this bug.

## The core issue ##
When using Solidity, function arguments are evaluated before the function's body is executed. The order of this evaluation is usually deterministic: left to right. However, during the optimization sequence, the FullInliner step seemed to cause deviations from this expected behavior. The issue arises when the ExpressionSplitter, another optimization step, is executed before the FullInliner. The ExpressionSplitter modifies the contract's intermediate representation (IR) by splitting complex expressions into smaller, reusable components. These modifications cause the reordering of the evaluation of arguments, breaking the left-to-right assumption and causing smart contracts to behave unexpectedly.

## How the bug surfaced and why Yul matters ##
To understand the bug fully, we'll examine how ExpressionSplitter and FullInliner interact:

1. The ExpressionSplitter transforms complex expressions into smaller sub-expressions to improve efficiency. For instance, it might break a function argument like a + b * c into separate calculations that are reused across multiple locations.

2. The FullInliner substitutes calls to small, frequently used functions directly into the calling code. While this improves performance, it can inadvertently reorder argument evaluation.

3. The critical interaction:
The bug surfaces only when the ExpressionSplitter step precedes the FullInliner step. The ExpressionSplitter step creates intermediate variables, and the FullInliner step reorders their evaluation based on perceived efficiency, ignoring the left-to-right evaluation order required for predictable behavior. This leads to unintended execution sequences in contracts.

One of the key insights from this incident is the importance of avoiding custom optimization configurations. The bug was particularly problematic when developers enabled specific Yul optimizations in which ExpressionSplitter did NOT precede FullInliner, without ensuring their interplay preserved the intended execution order. Custom configurations increase the risk of subtle issues, as they may enable conflicting optimization steps that were not fully tested together.

## Example ##
Consider the following code snippet:

```
function test(a, b) -> result {
    result := add(a, b)
}

function updateState() -> updatedValue {
    sstore(0x0, 1) // Change state
    updatedValue := 1
}

function readState() -> stateValue {
    stateValue := sload(0x0) // Read state
}

function main() -> output {
    let x := updateState()
    let y := readState()
    output := test(x, y)
}
```
Here, `updateState()` modifies the state variable stored at key 0x0 and `readState()` reads the value from the same storage key.
In a deterministic left-to-right evaluation:

1. `updateState()` executes first, updating the storage key.
2. `readState()` then reads the updated value, resulting in test(1, 1).

However, after Yul-based optimization with ExpressionSplitter and FullInliner:

- `updateState()` and `readState()` are split into intermediate steps.
- The FullInliner reorders these steps, causing `readState()` to execute before `updateState()`
This leads to test(1, 0), an unintended result.

## The resolution and recommendations ##
The Solidity team addressed this issue in version 0.8.21 by modifying the FullInliner step to respect the left-to-right evaluation order for expressions involving side effects, essentially eliminating the bug from existence. Developers are encouraged to:

1. Upgrade to the Latest Compiler Version

2. Test with Optimization Enabled

3. Avoid Reliance on Evaluation Order

4. Stick to Standard Configurations

## Lessons learned ##
The FullInliner bug highlighted several important lessons for developers: 

1. Developers should recognize that optimization steps can significantly alter the behavior of contracts.

2. While optimizations are essential for performance, excessive or poorly understood custom configurations introduce risks that often outweigh the benefits.

3. Contracts should be tested under recent, stable compiler versions and settings to ensure consistent, predictable behavior.

## Conclusion ##
The FullInliner bug underscores the importance of cautious, well-tested optimizations. By avoiding custom optimization configurations and adhering to best practices, developers can build reliable and secure smart contracts. 
