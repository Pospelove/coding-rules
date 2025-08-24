### **Rule: Decompose Large Methods into Specialized Handlers**

A single, large method responsible for handling multiple variations of a task should be refactored into a primary dispatching method and several smaller, specialized handler methods. The dispatcher's role is to inspect the input and delegate the work to the appropriate handler based on context.

---

### **Motivation**

This approach is a direct application of the **Separation of Concerns** and **Single Responsibility Principle**. Large, monolithic functions that use conditional logic to handle many different cases become difficult to manage over time.

This refactoring pattern addresses several key problems:

*   **Poor Readability and Cognitive Load:** A developer trying to understand a single, large function must mentally track all possible execution paths at once. By splitting the logic, each function becomes shorter, more focused, and easier to comprehend in isolation. The high-level logic is clear from the dispatcher, and the implementation details are encapsulated within the handlers.
*   **Difficult Maintenance:** Modifying the logic for one specific case (e.g., how spell damage is calculated) in a large function carries a high risk of unintentionally breaking another case (e.g., weapon damage). When logic is separated, changes are isolated to the relevant handler, reducing the risk of regressions.
*   **Code Duplication:** Often, different branches of a large conditional block share common setup or teardown steps. This can lead to duplicated code. Extracting these common parts into separate helper methods (as seen with `TrySendPapyrusOnHitEvent`) promotes reuse and simplifies the main logic.
*   **Limited Testability:** Writing unit tests for a function with numerous branches and complex internal state is challenging. Smaller, focused functions are significantly easier to test independently, leading to more robust and reliable code.

---

### **Example Analysis**

The provided code change refactors the `ActionListener::OnHit` function, which originally handled all incoming hit events in a single, complex block of code.

#### **Before Refactoring**

The original `OnHit` function was responsible for:
1.  Validating the aggressor and target.
2.  Checking if the aggressor possessed and had equipped the weapon used for the attack.
3.  Sending a generic "OnHit" Papyrus script event to the game engine.
4.  Checking for rapid attacks to prevent cheating.
5.  Calculating weapon-based damage.
6.  Updating the target's health.
7.  Logging the results.

This single method mixed validation, event dispatching, and damage calculation for what was implicitly only weapon-based combat. It had no clear way to handle other types of hits, such as those from spells.

#### **After Refactoring**

The changes implement the "Decompose into Specialized Handlers" pattern:

1.  **Creation of a Dispatcher:** The `OnHit` function is transformed into a lean dispatcher. Its primary responsibility is now to determine the *type* of the hit source (a spell or a weapon/unarmed attack).
    ```cpp
    // ActionListener::OnHit
    void ActionListener::OnHit(...)
    {
      // ... initial actor validation ...
    
      const bool isSourceSpell = ...; // Check if the hit source is a spell
    
      if (isSourceSpell && equipment.IsSpellEquipped(hitData.source)) {
        OnSpellHit(aggressor, hitData); // Delegate to spell handler
        return;
      }
    
      const bool isUnarmed = IsUnarmedAttack(hitData.source);
    
      if (equipment.inv.HasItem(hitData.source) || isUnarmed) {
        OnWeaponHit(aggressor, hitData, isUnarmed); // Delegate to weapon handler
        return;
      }
    
      // ... handle invalid/unequipped cases ...
    }
    ```

2.  **Extraction of Specialized Handlers:** The logic from the original function is moved into new, private methods with clear responsibilities:
    *   `OnWeaponHit`: Contains all logic specific to weapon and unarmed attacks, including the attack speed check and physical damage calculation.
    *   `OnSpellHit`: A new handler created to contain all logic specific to spell-based hits, including magic damage calculation and applying effects.

3.  **Extraction of a Common Helper:** The logic for sending the "OnHit" Papyrus event, which is common to both spell and weapon hits, is extracted into its own method, `TrySendPapyrusOnHitEvent`. This avoids code duplication and clarifies that sending this event is a distinct, reusable step in the process.

#### **Value and Outcome**

The refactored code is a significant improvement:
*   **Clarity:** It is immediately obvious how the system handles different types of hits. A developer can read `OnHit` to understand the high-level flow and then navigate to `OnSpellHit` or `OnWeaponHit` for specific implementation details.
*   **Extensibility:** If a new type of hit needs to be added (e.g., from a thrown potion or an environmental trap), a new `OnPotionHit` handler can be created and added to the dispatcher in `OnHit` without modifying the existing weapon or spell logic.
*   **Maintainability:** A bug in spell damage calculation can now be fixed within `OnSpellHit` with high confidence that it will not affect weapon combat.