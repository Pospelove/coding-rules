### **Principle: Implementing Fail-Safe Logic with Non-Throwing Wrappers and Detailed Diagnostics**

#### **Formal Definition**

When a critical operation is susceptible to failure due to external state changes (e.g., configuration updates, data file modifications, or resource unavailability), it should be encapsulated within a non-throwing wrapper function. This wrapper's responsibility is to convert exceptions or failure conditions into a predictable, manageable return type, such as `std::optional` or a status object, and to capture detailed diagnostic information. Higher-level calling code can then use this wrapper to probe the validity of the system's state without risking program termination, allowing it to implement fail-safe or graceful degradation logic, such as reverting to a default state or initiating a recovery process.

This principle combines three key practices:
1.  **Detailed Logging:** Instrumenting the core operation to produce a step-by-step trace of its execution.
2.  **Non-Throwing (NoThrow) Wrapper:** Creating a `try...catch` shell around the core operation that translates exceptions into non-fatal return values.
3.  **Fail-Safe Fallback:** Using the wrapper to validate system state and, upon detecting an inconsistency, reverting to a known-good behavior rather than crashing.

---

#### **Motivation**

In long-running or data-dependent applications like servers, the environment is not static. Data files, plugins, or configurations can be altered while the system is operational. An operation that was once valid may suddenly fail because its underlying data has become inconsistent or unavailable. Relying solely on exceptions to handle such cases can lead to unhandled exceptions and abrupt crashes, degrading service reliability.

The value of this principle is fourfold:

1.  **Enhanced System Resilience:** The application can anticipate and handle data inconsistencies gracefully. Instead of crashing when a data record is missing, it can detect the issue, log it, and fall back to a safe state (e.g., recalculating the data, using a default, or notifying an administrator). This transforms a fatal error into a recoverable event.
2.  **Superior Diagnostics:** By adding detailed logging within the core operation and capturing the final error message in the wrapper, developers are provided with a complete context of the failure. The log reveals not just *that* an error occurred, but precisely *where* and *why*, dramatically reducing debugging time.
3.  **Simplified Control Flow:** The non-throwing wrapper allows calling code to treat potential failures as a part of its normal logic. Instead of complex, nested `try...catch` blocks, the caller can perform a series of checks and then evaluate the results in a single, clear conditional block. This improves code readability and maintainability.
4.  **Proactive State Validation:** This pattern enables the system to proactively validate its own cached or persistent state. By periodically probing critical data structures with the non-throwing wrapper, the system can self-heal or flag inconsistencies before they are triggered by a user-facing operation.

---

#### **Example from Practice**

The provided code demonstrates this principle in the context of a game server evaluating an NPC's (Non-Player Character) template data. An NPC's properties (stats, inventory, etc.) are defined by a chain of templates. This chain can become invalid if a game data file (an `.esp` or `.esm` plugin) is updated or removed.

**Context:** The function `EvaluateTemplate` traverses this chain to find a specific property. If a link in the chain is broken (e.g., a template form ID no longer exists), the original function would throw a `std::runtime_error`, potentially crashing the process that invoked it.

**1. Augmenting the Core Function with Detailed Logging**

The core `EvaluateTemplate` function is first enhanced to build a detailed log of its progress. This provides a trace of its internal steps, which will be invaluable if an exception is thrown.

```cpp
// in EvaluateTemplate(...)
std::stringstream detailedLog;

for (auto it = chain.begin(); it != chain.end(); it++) {
  // ...
  detailedLog << "Processing FormDesc: " << it->ToString() << "\n";
  auto npcLookupResult = worldState->GetEspm().GetBrowser().LookupById(templateChainElement);
  detailedLog << "Variable npcLookupResult: " << npcLookupResult.rec << "\n";
  auto npc = espm::Convert<espm::NPC_>(npcLookupResult.rec);
  detailedLog << "Variable npc: " << npc << "\n";
  // ...
}

// The log is attached to the exception if one is thrown
ss << ", detailedLog=" << detailedLog.str();
throw std::runtime_error(ss.str());
```

**2. Creating the Non-Throwing Wrapper**

A new function, `EvaluateTemplateNoThrow`, is introduced. It wraps the call to the original `EvaluateTemplate` in a `try...catch` block. It returns a `std::optional`, which is empty if an exception occurs. It also accepts a pointer to a string to store the detailed error message.

```cpp
template <uint16_t TemplateFlag, class Callback>
auto EvaluateTemplateNoThrow(WorldState* worldState, uint32_t baseId,
                             const std::vector<FormDesc>& templateChain,
                             const Callback& callback,
                             std::string* outException)
{
  using ResultType = decltype(EvaluateTemplate<...>(...));
  std::optional<ResultType> result;

  try {
    result = EvaluateTemplate<...>(worldState, baseId, templateChain, callback);
  } catch (std::exception& e) {
    if (outException) {
      *outException = e.what(); // Capture the detailed error
    }
    result.reset(); // Ensure the optional is empty on failure
  }
  return result;
}
```

**3. Implementing Fail-Safe Logic in the Caller**

The `MpActor::EnsureTemplateChainEvaluated` function uses this new wrapper to validate an actor's cached template chain. Instead of assuming the chain is valid, it now probes it by attempting to evaluate every single template flag.

```cpp
void MpActor::EnsureTemplateChainEvaluated(...)
{
  // ...
  if (!templateChain.empty()) {
    std::string errorTraits, errorStats, errorFactions /*, ... and so on */;

    // Probe each property using the non-throwing wrapper.
    // The goal is not the result, but to see if it succeeds.
    auto doNothing = [](const auto&, const auto&) { return 1; };
    EvaluateTemplateNoThrow<espm::NPC_::UseTraits>(..., doNothing, &errorTraits);
    EvaluateTemplateNoThrow<espm::NPC_::UseStats>(..., doNothing, &errorStats);
    // ... many more calls for other flags

    // If all probes succeed, all error strings will be empty.
    if (errorTraits.empty() && errorStats.empty() && /* ... all others */) {
      return; // The cached chain is valid, do nothing.
    }

    // If any probe failed, at least one string is not empty.
    // This indicates the cached data is stale/corrupted.
    spdlog::info(
      "MpActor ... One of EvaluateTemplate errored, forgetting previous "
      "template chain. Likely, an update on esp/esm side.");
  }
  
  // Fallback behavior: The function proceeds to re-calculate the template chain from scratch.
  EditChangeForm(...); 
}
```

This final step perfectly illustrates the pattern. The system detects an invalid state without crashing. It logs a descriptive message explaining the likely cause and then executes its fallback logic: discarding the invalid data and regenerating it. This makes the server resilient to external data file changes.