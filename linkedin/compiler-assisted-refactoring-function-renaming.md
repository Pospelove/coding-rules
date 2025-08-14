Ever accidentally break something while refactoring? Changing a function's parameters can lead to sneaky bugs due to implicit conversions and missed call sites, making debugging a nightmare.

Rename functions when significantly altering their signature or behavior. The compiler then flags every instance of the old function, guaranteeing you update every usage, preventing hidden errors.

This simple rule becomes crucial with AI coding tools. It turns "change function X" into a verifiable "replace all 'old' with 'new'", using the compiler as the ultimate safety net. #Refactoring #Cpp #CodingBestPractices #AITools #CompilerAssisted