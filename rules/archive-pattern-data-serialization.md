### **Rule: Employ the Archive Pattern for Data Serialization**

A data structure should be decoupled from its specific serialized representation (e.g., JSON, binary). This is achieved by defining a single `Serialize` method within the data structure that interacts with a generic `Archive` interface. Concrete `Archive` classes then implement the format-specific logic for reading and writing data.

***

### **Motivation**

In systems with client-server communication, data must be serialized (converted to a stream of bytes for transmission) and deserialized (converted back into a data structure). A naive approach involves writing separate `ToJson`, `FromJson`, `ToBinary`, and `FromBinary` functions for every data structure. This leads to several problems that the provided code refactoring successfully solves:

1.  **Format Brittleness:** Tightly coupling data structures to a specific format like JSON makes the system rigid. Adding a new format (e.g., a more efficient binary protocol) or changing an existing one requires modifying every single data structure.
2.  **Code Duplication:** The logic for serializing common types like `std::vector`, `std::optional`, or `std::string` is repeated across many different `ToJson`/`FromJson` implementations, violating the Don't Repeat Yourself (DRY) principle.
3.  **Maintenance Overhead:** When a field is added to or removed from a struct, developers must remember to update multiple serialization and deserialization functions, increasing the risk of bugs and inconsistencies.
4.  **Difficulty with Complex Types:** As seen in the provided changes, adding support for complex types like `std::variant` becomes extremely cumbersome. Without the Archive pattern, one would need to implement complex `std::visit` logic inside every `ToJson` and `FromJson` function that uses a variant.

The Archive pattern addresses these issues by inverting the responsibility. Instead of the data structure knowing how to write itself to JSON, it simply describes its members to a generic "Archive." The Archive, in turn, knows how to handle the format-specific details.

This transition is the core principle demonstrated in the diff, moving from a system of disparate serialization functions to a unified, extensible, and maintainable architecture.

### **Example**

Consider a simple struct representing a message payload.

#### **Before: The Manual, Format-Coupled Approach**

The data structure is tightly coupled with the `nlohmann::json` library. Adding a binary format would require a completely separate set of `ToBinary`/`FromBinary` methods.

```cpp
#include <nlohmann/json.hpp>

struct PlayerUpdate {
    uint32_t id;
    std::string name;
    std::vector<float> position;

    // Logic is tied directly to the JSON format
    nlohmann::json ToJson() const {
        return nlohmann::json{
            {"id", id},
            {"name", name},
            {"position", position}
        };
    }

    static PlayerUpdate FromJson(const nlohmann::json& j) {
        PlayerUpdate p;
        p.id = j.at("id").get<uint32_t>();
        p.name = j.at("name").get<std::string>();
        p.position = j.at("position").get<std::vector<float>>();
        return p;
    }
};
```

#### **After: The Archive Pattern**

The `PlayerUpdate` struct is now completely agnostic of the serialization format. It only knows how to describe its members to an archive. The same struct can now be serialized to JSON, a binary BitStream, or any other format for which an archive is implemented, with no changes to the struct itself.

```cpp
struct PlayerUpdate {
    uint32_t id;
    std::string name;
    std::vector<float> position;

    // Single method for all serialization logic
    template <class Archive>
    void Serialize(Archive& archive) {
        archive.Serialize("id", id)
               .Serialize("name", name)
               .Serialize("position", position);
    }
};

// --- In another file (e.g., archives/JsonOutputArchive.h) ---
class JsonOutputArchive {
public:
    nlohmann::json j;
    template <typename T>
    JsonOutputArchive& Serialize(const char* key, T& value) {
        // (Simplified) Logic to write value to the JSON object
        j[key] = value;
        return *this;
    }
};

// --- In another file (e.g., archives/BitStreamOutputArchive.h) ---
class BitStreamOutputArchive {
public:
    SLNet::BitStream& stream;
    template <typename T>
    BitStreamOutputArchive& Serialize(const char* key, T& value) {
        // (Simplified) Logic to write value to the binary stream
        stream.Write(value);
        return *this;
    }
};

// --- Usage ---
PlayerUpdate update = { 1, "Alice", {10.0f, 20.0f, 30.0f} };

// Serialize to JSON
JsonOutputArchive jsonArchive;
update.Serialize(jsonArchive);
std::string jsonString = jsonArchive.j.dump(); // {"id":1,"name":"Alice","position":[10.0,20.0,30.0]}

// Serialize to Binary
SLNet::BitStream bitStream;
BitStreamOutputArchive bitStreamArchive(bitStream);
update.Serialize(bitStreamArchive);
// bitStream now contains the binary representation of the object
```

### **Value to Other Developers**

Adopting this pattern provides significant, long-term benefits to a project's architecture:

1.  **Flexibility and Extensibility:** New serialization formats can be supported by simply creating a new set of `Archive` classes. Existing data structures will work with the new format immediately, without modification. This is invaluable for performance tuning (e.g., switching from verbose JSON to a compact binary format) or for interoperability with other systems.
2.  **Maintainability (Separation of Concerns):** Data structures are responsible only for defining the data. Archives are responsible only for handling serialization formats. This clear separation makes the code easier to understand, test, and maintain. Adding a new field to a struct requires changing only one line in its `Serialize` method.
3.  **Code Reusability (DRY Principle):** The logic for serializing primitive types and standard library containers (`vector`, `optional`, `variant`, etc.) is implemented once within the `Archive` classes and reused everywhere. The provided diff shows this clearly with the addition of `std::variant` support—the complex `std::visit` logic was implemented once in the archives, making it instantly available to all data structures.
4.  **Consistency:** All data structures follow the same pattern for serialization, making the codebase more uniform and predictable for developers.