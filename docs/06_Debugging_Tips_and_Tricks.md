# Debugging Tips and Tricks for Customizing LibreOffice Paste

This document records some of the methods and commands used during the development and debugging of the custom paste functionality.

## 1. Confirming C++ Code Execution

To verify if a specific C++ code block is being hit during runtime:

1.  Add a temporary `fprintf` statement to `stderr`:
    ```c++
    // In your .cxx file (e.g., sc/source/ui/view/cellsh1.cxx)
    // At the beginning of the case block or function you are testing:
    fprintf(stderr, "\n>>>>>> MY_CUSTOM_CODE_SECTION_HIT <<<<<<\n\n");
    fflush(stderr); // Ensure immediate output
    ```
2.  Recompile the relevant module (e.g., `make sc` from `libreoffice-core` root).
3.  Run the custom LibreOffice application (e.g., `./instdir/program/scalc`) from the terminal.
4.  Perform the action that should trigger your code.
5.  Check the terminal output for your debug message.

## 2. Finding Header File Locations

If the compiler reports "fatal error: some/header.hxx: No such file or directory":

1.  Navigate to the root of the `libreoffice-core` source directory.
2.  Use `find` to locate the header:
    ```bash
    find . -name "header.hxx" 
    # Or case-insensitive:
    find . -iname "header.hxx"
    ```
3.  Adjust the `#include` directive in your C++ file based on the actual location and LibreOffice's include conventions (e.g., `<module_prefix/header.hxx>` or `<header.hxx>` for same-module includes).

## 3. Finding Class/Enum Definitions

If the compiler reports an "undeclared identifier" for a class or enum:

1.  Navigate to the `libreoffice-core` root.
2.  **For class definitions (e.g., `MyClass`):**
    ```bash
    # More precise (Perl regex, looks for class keyword and braces)
    find . -name "*.hxx" -o -name "*.h" | xargs grep -Pzo -H -n 'class\s+([A-Z_0-9]+\s+)?MyClass(\s*:[^\{]+)?\s*\{'
    # Simpler alternative
    find . -name "*.hxx" -o -name "*.h" | xargs grep -H -n -E '^class\s+([A-Z_0-9]+\s+)?MyClass'
    ```
3.  **For enum definitions (e.g., `MyEnum`):**
    ```bash
    find . -name "*.hxx" -o -name "*.h" -o -name "*.hrc" | xargs grep -H -n -E 'enum\s+(class\s+)?MyEnum'
    ```
4.  Examine the output to find the header file that *defines* the class/enum, not just forward-declares it.

## 4. Grepping for Code Usage (Examples)

These commands help understand how certain functions or types are used elsewhere in the codebase, which can reveal correct include paths or usage patterns. Run from `libreoffice-core` root.
*(Note: For patterns with special characters like `->` or `(`, quote the pattern string for `grep`.)*

*   **Finding includes of a specific header (e.g., `request.hxx`):**
    ```bash
    find . -name "*.cxx" -print0 | xargs -0 grep -H -n '#include.*request\.hxx'
    ```

*   **Finding usage of a function/method (e.g., `GetCurrentClipboard`):**
    ```bash
    find . -name "*.cxx" | xargs grep -H -n -C 3 'TransferableDataHelper::GetCurrentClipboard'
    ```

*   **Finding usage of an enum value (e.g., `SfxCallMode::SYNCHRON`):**
    ```bash
    find . -name "*.cxx" | xargs grep -H -n -C 5 -E 'SfxCallMode::SYNCHRON|SFX_CALLMODE_SYNCHRON'
    ```

*   **Finding members of a struct/class (e.g., `ScAsciiOptions`):**
    ```bash
    # This was used to find the ScAsciiOptions definition
    find . -name "*.hxx" -o -name "*.h" | xargs grep -Pzo -H -n 'struct\s+ScAsciiOptions\s*\{[^}]*\}|class\s+ScAsciiOptions\s*\{[^}]*\}'
    ```

*   **General search for a text string within C++ source/headers:**
    ```bash
    find . -name "*.cxx" -o -name "*.hxx" -o -name "*.h" -o -name "*.hrc" | xargs grep -H -n -C 2 "your_search_string"
    ```
    *(Adjust `-C 2` for lines of context.)*

## 5. Compiling Specific Modules/Files

*   To compile a whole module (e.g., `sc` for Calc) after changes:
    ```bash
    # From libreoffice-core root
    make sc
    ```
*   To speed up testing of a single C++ file change (less reliable for include path issues across modules, but good for syntax checks in one file):
    1.  Delete its object file: `rm workdir/CxxObject/path/to/yourfile.o`
    2.  Then run `make module_name` (e.g., `make sc`). Or sometimes `make path/to/yourfile.o` works if the makefile structure supports it directly (less common from top-level).
    *Alternatively, just `touch path/to/yourfile.cxx` and then `make module_name`.*

## 6. Understanding LibreOffice Build System (gbuild)

*   Build configuration is primarily in `Makefile.gbuild` and module-specific `Module_*.mk` files.
*   Include paths are set via `-I` flags, often defined using variables like `$(INCLUDE)` or specific paths to other modules' `inc` directories if a dependency is declared.

## 7. Using `git diff`

*   Before committing or creating a patch, always review your changes:
    ```bash
    git diff
    ```
*   To see staged changes:
    ```bash
    git diff --staged
    ```
*   To create a patch file from uncommitted changes:
    ```bash
    git diff > my_feature.patch
    ```
    Or for specific files:
    ```bash
    git diff path/to/file.cxx path/to/another/file.xcu > my_changes.patch
    ```
