### **Principle: Isolate Data Representation from Construction Logic**

#### **General Rule**

Use a dedicated Factory or Builder class to construct data-holding objects from external or complex source data, rather than embedding this creation logic within the data objects' constructors. The data-holding classes should be kept as simple as possible, responsible only for representing the data itself.

---

#### **Motivation and Rationale**

This principle is an application of the **Single Responsibility Principle (SRP)** and promotes **decoupling**.

1.  **Separation of Concerns:**
    *   A data structure's primary responsibility is to *represent* data in a clear and accessible format (e.g., for serialization, transfer, or use in other parts of the application).
    *   A factory's primary responsibility is to *create* and *assemble* that data structure, often by translating from a different, more complex, or external data format.
    *   By mixing these concerns, the data structure becomes bloated, harder to understand, and more difficult to maintain.

2.  **Decoupling from External Dependencies:**
    *   When constructors handle complex object creation, the data structure's header file must include headers for all source data types. This creates a tight coupling. If the external source types change, the data structure itself must be modified.
    *   By using a factory, only the factory is coupled to the external source types. The data structure remains pure and independent, unaware of where its data came from. This minimizes ripple effects from changes in external libraries or low-level APIs.

3.  **Improved Cohesion and Maintainability:**
    *   All logic related to the creation and transformation of data is centralized within the factory. This includes helper functions, data lookups, and format conversions. This makes the creation process easier to find, understand, debug, and modify.

4.  **Simplified and Reusable Data Structures:**
    *   The resulting data structures become simple Plain Old Data (POD) or Data Transfer Objects (DTOs). They are easy to serialize, test, and reuse in different contexts because they have no complex construction logic or external dependencies.

---

#### **Example: Refactoring Data Dump Creation**

The provided code change illustrates a move from a coupled design to a decoupled, factory-based approach.

##### **Before: Coupled Construction**

Initially, the data format classes (`FunctionsDumpFormat::Root`, `::Function`, etc.) were responsible for their own construction from raw, engine-specific types (`RE::BSScript::IFunction*`, `PexScript`, etc.).

**`FunctionsDumpFormat.h` (Before)**
```cpp
// This header is now polluted with dependencies from external APIs
#include "papyrus-vm/Reader.h" 
#include <vector>
#include <tuple>

namespace FunctionsDumpFormat {

struct Root
{
  // Constructor contains complex logic and is tied to specific input types
  explicit Root(
    const std::vector<
      std::tuple<std::string, std::string, RE::BSScript::IFunction*, ...>>& data,
    const std::vector<std::shared_ptr<PexScript>>& pexScripts);
  
  // ... other members
};
}
```

**Usage (Before)**
```cpp
// The calling code directly instantiates the data format,
// passing in raw, low-level data.
FunctionsDumpFormat::Root root(data, pexScripts); 
```

This design couples the `FunctionsDumpFormat` module to the internal representations of the game engine (`RE::*`) and the Papyrus VM (`PexScript`).

##### **After: Decoupled Factory**

The refactoring introduces a `FunctionsDumpFactory` to encapsulate all creation logic. The `FunctionsDumpFormat` classes are simplified to pure data containers.

**`FunctionsDumpFormat.h` (After)**
```cpp
// Clean header with no external dependencies for construction.
// It only defines the shape of the data.
#pragma once
#include <string>
#include <vector>
#include <map>

namespace FunctionsDumpFormat {

struct Root
{
  Root() = default; // Simple default constructor

  std::map<std::string, Type> types;
  // ... other members
};
}
```

**`FunctionsDumpFactory.h` (New)**
```cpp
// This class now holds all the complex dependencies and logic.
#pragma once
#include "FunctionsDumpFormat.h"
#include "papyrus-vm/Reader.h"
// ... other necessary includes for source data

class FunctionsDumpFactory
{
public:
  // The public interface takes the raw data and returns the clean data structure.
  static FunctionsDumpFormat::Root Create(
    const std::vector<
      std::tuple<std::string, std::string, RE::BSScript::IFunction*, ...>>& data,
    const std::vector<std::shared_ptr<PexScript>>& pexScripts);

private:
  // All helper functions are now private implementation details of the factory.
  static FunctionsDumpFormat::Function MakeFunction(...);
  static std::string FindModuleName(uintptr_t moduleBase);
};
```

**Usage (After)**
```cpp
// The calling code requests the final object from the factory.
// It is unaware of the complex construction process.
FunctionsDumpFormat::Root root =
  FunctionsDumpFactory::Create(data, pexScripts);
```

---

#### **Value and Conclusion**

Adopting this pattern provides significant value:

*   **Maintainability:** When the process of extracting function data from the game engine changes, only the `FunctionsDumpFactory` needs to be updated. The `FunctionsDumpFormat` structures, and any code that consumes them (like the JSON serializer), remain completely untouched.
*   **Clarity:** The code becomes easier to read. `FunctionsDumpFormat.h` clearly and concisely defines the *output format*, while `FunctionsDumpFactory.cpp` details the complex process of *generating* that output.
*   **Testability:** The pure data structures in `FunctionsDumpFormat` are trivial to create and use in unit tests. The factory can be tested independently to ensure it correctly transforms source data into the target format.

In summary, separating data representation from construction logic via a factory leads to a more robust, modular, and maintainable codebase.