### **Rule: Compiler-Assisted Refactoring via Function Renaming**

When performing a significant refactoring that alters a function's signature, semantics, or core contract, the function should be renamed rather than modified in-place. This forces the compiler to flag every call site of the original function as an error, ensuring a complete and deliberate migration.

***

### **Motivation**

Modifying the signature of a widely-used function is a high-risk operation. Simply changing parameter types can lead to subtle and dangerous issues that are difficult to detect.

1.  **Elimination of Implicit Conversions and Silent Failures:** C++ may attempt to perform implicit type conversions to match a modified function signature. These conversions can be unintended, inefficient, or lead to incorrect runtime behavior that the compiler does not flag as an error. By renaming the function, all calls to the old name become unresolved, creating explicit, unavoidable compile-time errors. This transforms hidden logical risks into obvious syntax errors.

2.  **Guaranteed Completeness:** A manual or automated search-and-replace can easily miss call sites, especially in a large codebase with complex overloading or templating. Leveraging the compiler as a "to-do list" guarantees that the project will not build until every single usage of the deprecated function has been reviewed and updated to the new contract. Nothing is missed.

3.  **Enhanced Safety for Automated Tooling:** When using refactoring tools or AI-powered code assistants, the instruction to "change the signature of function X" can be ambiguous. The tool might not understand the full context of every call site. By enforcing a rename, the task becomes "replace all calls to `old_name` with `new_name`, adapting the arguments accordingly." The compiler serves as a final, definitive verifier of the tool's work.

4.  **Clarity of Intent:** The new function name can more accurately describe its new behavior. As seen in the example, changing `Set` to `SetValueDump` immediately communicates to future developers that the function no longer works with a parsed object but with its serialized string representation. This makes the code more self-documenting.

***

### **Example**

Consider a class that stores dynamic properties as parsed `nlohmann::json` objects.

**Initial State:**

```cpp
// DynamicFields.h
class DynamicFields {
public:
    void Set(const std::string& propName, nlohmann::json value);
    // ...
private:
    std::unordered_map<std::string, nlohmann::json> props;
};

// SomeOtherFile.cpp
void UpdateUserStatus(DynamicFields& fields, bool isOnline) {
    nlohmann::json statusValue = isOnline;
    fields.Set("isOnline", statusValue); // Works perfectly
}
```

**Refactoring Goal:**

For performance reasons, we decide to stop parsing and storing `nlohmann::json` objects directly. Instead, we will store the pre-serialized `std::string` (the "dump") to avoid repeated serialization/deserialization costs.

**Applying the Rule:**

Instead of just changing the signature of `Set`, we rename it to reflect its new contract.

```cpp
// DynamicFields.h (Updated)
class DynamicFields {
public:
    // The old 'Set' function is removed or marked as deprecated.
    // The new function has a more descriptive name.
    void SetValueDump(const std::string& propName, const std::string& valueDump);
    // ...
private:
    // The underlying storage is changed.
    std::unordered_map<std::string, std::string> propDumps;
};
```

Now, when we try to compile the project, the compiler will immediately fail at the call site.

**Compiler Output:**

```
error: 'class DynamicFields' has no member named 'Set'
   fields.Set("isOnline", statusValue);
          ^~~
```

This error forces the developer to consciously address the call site and adapt it to the new, more performant contract.

**Final Corrected State:**

```cpp
// SomeOtherFile.cpp (Corrected)
void UpdateUserStatus(DynamicFields& fields, bool isOnline) {
    nlohmann::json statusValue = isOnline;
    // The caller is now responsible for serialization,
    // and calls the new, explicitly named function.
    fields.SetValueDump("isOnline", statusValue.dump()); 
}
```

The provided code diff is a large-scale application of this principle. Functions like `SetProperty`, `Get`, and `CreatePropertyMessage` were fundamentally changed to operate on serialized strings (`std::string`) instead of `nlohmann::json` objects. By renaming them to `SetPropertyValueDump`, `GetValueDump`, and `CreatePropertyMessage_` respectively, the developers ensured that every single usage across the C++ codebase was found and updated by the compiler, guaranteeing a safe and complete refactoring.