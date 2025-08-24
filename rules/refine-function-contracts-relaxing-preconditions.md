### **Rule: Refine Function Contracts by Relaxing Preconditions**

A function's preconditions, especially those that validate state before an operation, should be periodically re-evaluated. If enforcing a strict precondition for a common, non-critical scenario leads to excessive logging ("log noise") or complicates caller logic, the precondition should be relaxed. The function's contract can be evolved from "fail if state is not X" to "ensure state becomes Y," making it more robust and idempotent.

This is a practical application of Design by Contract, where the contract is consciously evolved to better suit the system's real-world usage patterns. It involves transforming what was previously considered a caller error (a "soft assert") into a valid, gracefully-handled state.

***

### **Motivation**

Adhering to this principle provides several key benefits to a software system's maintainability and reliability:

1.  **Reduced Log Noise:** The primary benefit is a cleaner, more meaningful logging system. By eliminating warnings for predictable, non-erroneous conditions (e.g., attempting to remove an item that is already gone), developers can focus on logs that indicate genuine, unexpected problems. This improves the signal-to-noise ratio, making debugging and monitoring more effective.

2.  **Simplified Caller Logic:** When a function is responsible for gracefully handling edge-case states, the burden of checking those states is lifted from every caller. This prevents the proliferation of defensive `if`-checks throughout the codebase, leading to cleaner, more declarative, and less-duplicated code.

    *   **Before:** `if (actor.HasSpell(spell)) { actor.RemoveSpell(spell); }`
    *   **After:** `actor.RemoveSpell(spell);`

3.  **Increased Robustness and Decoupling:** A system with forgiving, idempotent functions is inherently more robust. It becomes less susceptible to race conditions or state changes that occur between a check and an action. Callers do not need to have perfect, up-to-the-millisecond knowledge of the object's state, which promotes looser coupling between components. The intent becomes "ensure this spell is absent," which the refined function fulfills perfectly, regardless of the initial state.

***

### **Example**

**Context:** A function `Actor.RemoveSpell` is designed to remove a spell from a character in a game.

#### **Initial Implementation**

The initial design enforces a strict contract: the caller must ensure the actor has learned the spell before attempting to remove it. Violating this contract results in a logged warning.

```cpp
// Original Code
VarValue PapyrusActor::RemoveSpell(VarValue self,
                                   std::vector<VarValue>& args)
{
  // ... argument parsing ...

  if (actor->IsSpellLearnedFromBase(spellId)) {
    spdlog::warn("Actor.RemoveSpell - can't remove spells inherited from "
                  "RACE/NPC_ records");
  } else if (!actor->IsSpellLearned(spellId)) {
    // This check creates log noise. It treats a common state
    // (spell is already gone) as an error.
    spdlog::warn("Actor.RemoveSpell - spell already removed/not learned");
  } else {
    // The action only occurs if all preconditions are strictly met.
    actor->RemoveSpell(spellId);
    // ...
  }
  return {};
}
```

**Problem:** In a dynamic system, multiple scripts might try to remove the same spell. The first call succeeds, but all subsequent calls flood the logs with `spell already removed/not learned` warnings. This is not an exceptional situation but a common outcome of "cleanup" logic, making the warnings unhelpful noise.

#### **Refined Implementation**

The contract is refined. The function's goal is now to **ensure the spell is removed**. If the spell is already absent, its goal is already satisfied, and it can silently do nothing. The check for the spell's existence is transformed from a precondition-check-with-failure to a guard for the operation itself.

```cpp
// Refined Code
VarValue PapyrusActor::RemoveSpell(VarValue self,
                                   std::vector<VarValue>& args)
{
  // ... argument parsing ...

  if (actor->IsSpellLearnedFromBase(spellId)) {
    spdlog::warn("Actor.RemoveSpell - can't remove spells inherited from "
                  "RACE/NPC_ records");
  } else if (actor->IsSpellLearned(spellId)) {
    // The contract is now relaxed. We only act if the spell is present.
    // If it's not present, we do nothing, generating no log noise.
    // The operation is now idempotent.
    actor->RemoveSpell(spellId);
    // ...
  }
  return {};
}
```

**Result:** The function is now more robust. It can be called safely at any time with the intent of ensuring a spell is gone, without requiring the caller to manage the state check. The system becomes cleaner, and the logs are reserved for more significant issues.