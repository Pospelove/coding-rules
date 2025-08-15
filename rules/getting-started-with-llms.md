### **Rule: Isolate Complex Object Construction into a Dedicated Factory**

Isolate the logic of creating and populating data structures from the definitions of those structures. This is achieved by introducing a "Factory" class or method whose sole responsibility is to handle the complex construction process, encapsulating all dependencies and logic required to build a valid object.

---

### **Motivation**

The primary goal of this rule is to improve code quality by adhering to the **Single Responsibility Principle (SRP)**. A class should have only one reason to change. By separating construction logic from data representation, we create two distinct components with clear, single responsibilities:

1.  **The Data Structure (e.g., `FunctionsDumpFormat::Root`)**: Its only responsibility is to represent data. It should contain member variables, serialization methods, and simple accessors. It should not be concerned with *how* it is created or from what sources. This makes the data structure simple, portable, and easy to understand.

2.  **The Factory (e.g., `FunctionsDumpFactory`)**: Its only responsibility is to create instances of the data structure. It encapsulates the complex assembly process, including iterating over source data, transforming types, looking up information, and handling external dependencies (like PEX scripts or engine-specific types).

This separation provides several key benefits:

*   **Improved Readability and Maintainability**: Developers can understand the data format by looking at `FunctionsDumpFormat.h` without being burdened by the complex creation logic. Conversely, when the creation process needs to be changed, they know to look exclusively in `FunctionsDumpFactory.cpp`.
*   **Reduced Coupling**: The data structure classes no longer depend on the source types (`RE::BSScript::IFunction`, `PexScript`, etc.). This decoupling makes the data structures more reusable and the system as a whole more modular.
*   **Enhanced Testability**: The data structures can be tested in isolation by manually populating them with test data. The factory can also be tested independently by verifying that it produces correctly structured objects given a specific input.
*   **Increased Flexibility**: If a new source of data is introduced for creating the same format, a new creation method can be added to the factory without any changes to the data structure classes themselves.

---

### **Example Analysis**

The provided code change is a canonical example of applying this rule.

#### **Before Refactoring**

The construction logic was embedded directly within the constructor of the data-holding class, `FunctionsDumpFormat::Root`.

**`FunctionsDumpFormat.h` (conceptual before state):**
```cpp
namespace FunctionsDumpFormat {
  struct Root {
    // Constructor mixes creation logic with the class definition.
    explicit Root(
      const std::vector</*...complex tuple...*/ >& data,
      const std::vector<std::shared_ptr<PexScript>>& pexScripts);

    std::map<std::string, Type> types;
    // ... other data members
  };
}
```

**`DumpFunctions.cpp` (call site):**
```cpp
// Direct construction, hiding the complexity within the constructor.
FunctionsDumpFormat::Root root(data, pexScripts);
```

This design violates the Single Responsibility Principle. The `Root` class was responsible for both *representing* the final data and *knowing how to build itself* from raw engine and script data. This made the `FunctionsDumpFormat.h` header file complex and coupled the data format directly to its sources.

#### **After Refactoring**

A new `FunctionsDumpFactory` class is introduced to encapsulate all construction logic. The `FunctionsDumpFormat` classes become pure data containers.

**`FunctionsDumpFormat.h` (after):**
```cpp
namespace FunctionsDumpFormat {
  struct Root {
    // A simple default constructor. The class only defines the data shape.
    Root() = default;

    std::map<std::string, Type> types;
    // ... other data members
  };
}
```

**`FunctionsDumpFactory.h` (new file):**
```cpp
class FunctionsDumpFactory {
public:
  // A static 'Create' method is the public interface for construction.
  static FunctionsDumpFormat::Root Create(
    const std::vector</*...complex tuple...*/ >& data,
    const std::vector<std::shared_ptr<PexScript>>& pexScripts);
private:
  // All helper functions for creation are private implementation details.
  static FunctionsDumpFormat::Function MakeFunction(...);
  // ... other helper methods
};
```

**`DumpFunctions.cpp` (new call site):**
```cpp
// The intent is now explicit: use a factory to create the object.
FunctionsDumpFormat::Root root =
  FunctionsDumpFactory::Create(data, pexScripts);
```

The result is a much cleaner design. The `FunctionsDumpFormat` classes are now simple, self-contained data structures. All the complex logic for iterating `data` and `pexScripts`, finding module names, and enriching types is cleanly isolated within `FunctionsDumpFactory`, which is its proper and single responsibility.