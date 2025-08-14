### **Rule: Compiler-Assisted Refactoring via Function Renaming**

When a function's signature is fundamentally altered in a way that changes the type, semantics, or pre-conditions of its parameters, the function should be renamed. The old function name should be deprecated or removed entirely.

This renaming forces the compiler to flag every call site of the original function as an error (e.g., "undeclared identifier" or "no matching function for call"). This transforms a potentially silent and error-prone refactoring process into a clear, compiler-guided checklist of locations that require manual review and update.

---

### **Motivation**

This practice provides several key advantages over simply changing a function's signature and relying on the compiler to catch type mismatches.

1.  **Eliminates Ambiguity and Silent Failures:** The primary motivation is to prevent the compiler from performing unintended implicit conversions. If a function signature changes from `void Foo(Bar b)` to `void Foo(Baz b)`, and an implicit conversion from `Bar` to `Baz` exists, the code will continue to compile. However, this may introduce subtle bugs, performance regressions, or incorrect behavior that is difficult to trace. Renaming the function to `FooWithBaz` makes all previous calls to `Foo` fail compilation, guaranteeing that a developer must address each one.

2.  **Enforces Conscious Updates:** A list of compiler errors serves as a mandatory to-do list. It forces the developer to visit every location where the function was used and make a conscious decision about how to adapt the calling code to the new contract. This is significantly more reliable than relying on text-based search-and-replace, which can be imprecise.

3.  **Enhances Code Clarity:** The new function name often serves as better documentation. In the provided example, changing `SetProperty` to `SetPropertyValueDump` immediately communicates to future readers that the function expects a pre-serialized string, not a structured JSON object.

4.  **Reduces Risk from Automated Tooling:** Modern development workflows increasingly involve AI agents and sophisticated refactoring tools. These tools may not have the full context to understand the semantic implications of a signature change. By making the old function name invalid, we create a safeguard that prevents an automated tool from incorrectly "fixing" the call sites in a way that compiles but is logically flawed. A hard compiler error is an unambiguous signal that human intervention is required.

---

### **Example**

Consider a class that stores properties as parsed `nlohmann::json` objects.

**Initial State:**

The `PropertyStore` class uses a `SetProperty` method that accepts a `nlohmann::json` object.

```cpp
// property_store.h
#include <nlohmann/json.hpp>
#include <string>
#include <unordered_map>

class PropertyStore {
public:
    void SetProperty(const std::string& key, const nlohmann::json& value);
private:
    std::unordered_map<std::string, nlohmann::json> properties_;
};

// usage.cpp
void UpdateUser(PropertyStore& store) {
    nlohmann::json userStatus = {{"loggedIn", true}, {"level", 15}};
    store.SetProperty("status", userStatus); // Compiles and works as expected.
}
```

**Refactoring Goal:**

For performance reasons, we decide to stop storing parsed `nlohmann::json` objects and instead store them as serialized JSON strings (`std::string`) to avoid repeated parsing and serialization costs.

**Incorrect Approach (Signature Change Only):**

If we only change the signature, we introduce risk.

```cpp
// property_store.h (Modified)
class PropertyStore {
public:
    // Signature changed from nlohmann::json to std::string
    void SetProperty(const std::string& key, const std::string& valueDump);
private:
    std::unordered_map<std::string, std::string> properties_;
};

// usage.cpp (Now has a bug)
void UpdateUser(PropertyStore& store) {
    nlohmann::json userStatus = {{"loggedIn", true}, {"level", 15}};
    
    // COMPILE ERROR: No matching function for call to 'SetProperty'.
    // In this specific case, the compiler helps. But if an implicit
    // conversion from nlohmann::json to std::string were available,
    // this could compile and introduce a runtime bug.
    store.SetProperty("status", userStatus); 
}
```
While the compiler catches the error here, relying on this is fragile. A different scenario with a plausible implicit conversion could lead to a silent bug.

**Recommended Approach (Function Renaming):**

By renaming the function, we make the change explicit and force a deliberate update at all call sites.

```cpp
// property_store.h (Renamed)
class PropertyStore {
public:
    // Renamed to reflect the new contract: it expects a string "dump".
    void SetPropertyValueDump(const std::string& key, const std::string& valueDump);
private:
    std::unordered_map<std::string, std::string> properties_;
};

// usage.cpp (Developer is forced to update correctly)
void UpdateUser(PropertyStore& store) {
    nlohmann::json userStatus = {{"loggedIn", true}, {"level", 15}};

    // OLD CODE: Now produces a clear "undeclared identifier" error.
    // store.SetProperty("status", userStatus); 
    // ^-- COMPILE ERROR: 'SetProperty' is not a member of 'PropertyStore'

    // NEW, CORRECT CODE: The developer must explicitly serialize the JSON.
    // The new function name makes the required action obvious.
    store.SetPropertyValueDump("status", userStatus.dump());
}
```

This disciplined approach leverages the compiler as a safety net, ensuring that significant refactoring is completed robustly and correctly across the entire codebase.