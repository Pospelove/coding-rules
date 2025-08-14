### **Principle: Compiler-Assisted Refactoring through Function Renaming**

#### **Rule**

When modifying a function's signature in a way that alters its fundamental data types or contract, rename the function to reflect the change. This practice intentionally breaks all existing call sites, generating compile-time errors that guide the developer to update every usage correctly.

---

#### **Motivation and Rationale**

In large and complex codebases, changing a function's signature can introduce subtle and dangerous bugs. The primary risks are:

1.  **Silent Failures via Implicit Conversions:** The C++ compiler may find a way to implicitly convert types to match the new signature, potentially leading to unintended behavior or performance degradation that is not caught at compile time.
2.  **Incomplete Refactoring:** A developer (or an automated tool) might miss certain call sites, especially in disparate parts of the codebase. This leads to a mixed state where old and new calling conventions coexist, causing logical errors.

By deliberately renaming the function (e.g., from `Set` to `SetValueDump`), we transform a potentially silent logical issue into a mandatory compile-time error. The compiler effectively becomes a "to-do list" generator, pointing out every single location that requires a conscious, manual update.

This technique forces the developer to re-evaluate each call site in the context of the new contract, ensuring that the logic is correctly adapted rather than just syntactically patched.

---

#### **Example from the Provided Code**

The codebase is being refactored to pass around serialized JSON strings (`std::string`) instead of parsed JSON objects (`nlohmann::json`) to improve performance by avoiding repeated parsing and serialization.

**Initial State (Before Refactoring):**

The `DynamicFields` class stores properties as `nlohmann::json` objects. The `Set` method accepts a `nlohmann::json` object directly.

```cpp
// DynamicFields.h
class DynamicFields
{
public:
  void Set(const std::string& propName, nlohmann::json value);
  // ...
private:
  std::unordered_map<std::string, nlohmann::json> props;
};

// Usage
void MpObjectReference::SetProperty(const std::string& propertyName,
                                    nlohmann::json newValue, ...)
{
  // ...
  changeForm.dynamicFields.Set(propertyName, std::move(newValue));
  // ...
}
```

**Refactoring Process (Applying the Rule):**

The goal is to change the internal storage and function signature to use pre-serialized JSON strings (`std::string`). Instead of just changing the `Set` function's signature, it is renamed to `SetValueDump`.

```cpp
// DynamicFields.h (After Refactoring)
class DynamicFields
{
public:
  void SetValueDump(const std::string& propName, const std::string& valueDump);
  // ...
private:
  std::unordered_map<std::string, std::string> propDumps;
};

// Usage (After Refactoring)
void MpObjectReference::SetPropertyValueDump(const std::string& propertyName,
                                             const std::string& valueDump, ...)
{
  // ...
  // The old call `changeForm.dynamicFields.Set(...)` would now fail to compile.
  // The compiler forces the developer to use the new function and provide the correct type.
  changeForm.dynamicFields.SetValueDump(propertyName, valueDump);
  // ...
}
```

If the function had not been renamed, a call site passing a `const char*` might have implicitly converted to `std::string`, which could then be implicitly converted to a `nlohmann::json` object containing that string. Renaming prevents such ambiguities and forces an explicit `value.dump()` at the source.

---

#### **Value and Problems Solved**

Adhering to this principle provides significant value:

*   **Guaranteed Correctness:** It eliminates the possibility of missing a call site during a refactoring. The code will not compile until all usages are updated, ensuring a complete and consistent transition.
*   **Enhanced Code Clarity:** The new function name (e.g., `SetValueDump`, `ForEachValueDump`) is more descriptive and accurately communicates its new contract to future developers. It signals that the function expects a serialized string, not a JSON object.
*   **Risk Mitigation for Automated Tooling:** It safeguards against errors from automated refactoring tools or AI agents that might not grasp the full semantic context of a change. An agent tasked with "changing the type to string" might perform a naive replacement, but it cannot fix a compile error resulting from a function that no longer exists under its old name without a deeper understanding.
*   **Simplified Debugging:** It prevents a class of hard-to-diagnose runtime bugs that stem from incorrect data types or silent, unintended conversions, shifting the error detection from runtime to compile time.