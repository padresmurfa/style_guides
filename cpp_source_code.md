### C++ Style Guide

This document outlines the conventions and best practices for writing clear, structured, and maintainable C/C++ code for our project. It applies to all layers (e.g., the ICU4C wrapper, grapheme cluster routines, and related modules) and assumes that a central header pulls in all necessary standard and third-party includes. Each layer may remap exceptions from lower layers, so exception names are kept succinct.

#### 1. Target Audience
- **Primary Audience:**  
  - Team members and future developers (including those with moderate experience).  
  - Large language models acting as coding assistants.
- **Goal:**  
  - Ensure that code is self‑explanatory, well‑documented, and easy to maintain.

#### 2. Naming Conventions

##### 2.1. Functions, Variables, and Member Names
- **Lower Snake Case:**  
  All function names, variable names, and member functions use `lower_snake_case`.  
  - **Example:**  
    `to_char32()`, `m_data_ptr`, `is_empty()`

##### 2.2. Constants and Macros
- **Upper Snake Case:**  
  Constants and macro-like values are written in `UPPER_SNAKE_CASE`.  
  - **Example:**  
    `INLINE_CAPACITY`, `HIGH_SURROGATE_START`

##### 2.3. Exception Names
- **Descriptive Naming:**  
  Use descriptive names for exceptions (e.g., `grapheme_cluster_error`, `grapheme_cluster_invalid_error`, etc.).  
  With our simplified exception handling, we now use:
  - **error** (the base class),
  - **logic_error** (for logic-related failures),
  - **invalid_arguments_logic_error** (for invalid arguments that should have been caught at design time), and
  - **data_error** (for errors arising from the data, such as unpaired surrogates).

##### 2.4. Strongly Typed Integers and Position Types
- **index_of_code_unit_t:**  
  Use this type for parameters referring to the index of an actual code unit in a string (valid range: `[0, length - 1]`).  
  - **Example:** Use `index_of_code_unit_t` in functions that accept a code‑unit position.
- **boundary_in_code_units_t (if needed):**  
  Use this type when you need to express a position _between_ code units. (For now, our code uses `index_of_code_unit_t` for positions referring to existing code units.)

#### 3. Documentation

##### 3.1. Doxygen Comments
- **Triple‑Slash Style:**  
  Every function should be preceded by a documentation comment using triple‑slash (`///`) with backslash commands.
- **Section Separation:**  
  Separate the `\brief`, `\param`, `\return`, and `\throws` sections with an empty line for readability.

**Example:**
```cpp
/// \brief Computes the total number of code units spanned by a specified number of grapheme clusters,
/// counting backwards from the end of the text.
/// 
/// \param text Pointer to the UTF‑16 text.
/// \param text_length The length of the text in code units.
/// \param clusters_to_process The number of grapheme clusters to process.
/// 
/// \return The total number of code units spanned.
/// 
/// \throws invalid_arguments_logic_error if text is null, text_length is non-positive, or if clusters_to_process
///         is non-positive.
/// \throws data_error if there are insufficient grapheme clusters in the text.
```

##### 3.2. Inline Comments
- **Explanatory Comments:**  
  Insert one‑line comments to describe the purpose of code sections or complex logic steps.  
  For example, before validating surrogate pairs, include a comment explaining the intent.

#### 4. Function Declarations and Definitions

##### 4.1. Function Declarations (in Header Files)
- **Layout:**  
  - **Return Type:** Indented one level.
  - **Function Name:** Flush left on the next line.
  - **Parameters:** Each parameter appears on its own line, indented twice.
  - **Terminator:** The closing parenthesis and semicolon remain on the same line as the last parameter.

**Example:**
```cpp
    length_in_grapheme_clusters_t
length_in_grapheme_clusters_to_length_in_code_units(
        const unicode_string_t* text,
        length_in_code_units_t text_length,
        index_of_code_unit_t starting_position,
        length_in_grapheme_clusters_t clusters_to_process
);
```

##### 4.2. Function Definitions (in Source Files)
- **Layout:**  
  Similar to declarations, but in definitions, the closing parenthesis is on its own line, followed by a space and the opening curly brace.
- **Indentation:**  
  In the primary namespace, function body content is indented once relative to the function signature. In anonymous namespaces, add the extra indentation uniformly.

**Example:**
```cpp
length_in_code_units_t
length_in_grapheme_clusters_to_length_in_code_units(
        const unicode_string_t* text,
        length_in_code_units_t text_length,
        index_of_code_unit_t starting_position,
        length_in_grapheme_clusters_t clusters_to_process
) {
    if (text == nullptr
            || text_length < 0
            || starting_position < 0
            || starting_position >= (index_of_code_unit_t::integer_type)text_length
            || clusters_to_process < 0) {
        throw invalid_arguments_logic_error("length_in_grapheme_clusters_to_length_in_code_units: invalid arguments");
    }
    // ... function body ...
}
```

#### 5. Indentation and Bracing

- **Indentation:**  
  Use spaces consistently for indentation (the number of spaces per level should be fixed).
- **Curly Brackets:**  
  Opening curly braces are on the same line as the statement that opens the block, separated by a space.
  
**Example:**
```cpp
void
validate_surrogate_pair(
        const y::icu4c::unicode_string_t* text,
        int32_t current_boundary,
        y::icu4c::length_in_code_units_t text_length
) {
    // Retrieve the first code unit.
}
```

#### 6. Section Separator Comments

- **Usage:**  
  Use block separator comments with vertical space before and after to clearly mark sections. For example:
```cpp
///////////////////////////////////////////////////////////////////////////////
// Anonymous Namespace
///////////////////////////////////////////////////////////////////////////////
```
In test files, we use a lighter separator for docstrings (see unit testing style guide).

#### 7. Includes

- **Centralized Includes:**  
  Do not include individual system or third‑party headers in module files. Each file should include `"yicu4c/source/yicu4c.hpp"`, which in turn pulls in all necessary headers.

#### 8. Strongly Typed Integer Conversions

- **Typecasting:**  
  When a conversion is needed, use explicit typecasting (e.g. `int32_t(text_length)`) rather than a `to_int32()` method.

#### 9. Control Structures

- **Curly Braces:**  
  Every `if`, `else`, `for`, and `while` block must use curly braces, even if the block contains a single statement.
- **Multi-line Conditions:**  
  Break complex expressions into multiple lines with each condition on a separate line, properly indented.  
  **Example:**
  ```cpp
  if (text == nullptr
          || text_length < 0
          || starting_position < 0
          || starting_position >= (index_of_code_unit_t::integer_type)text_length
          || clusters_to_process < 0) {
      throw invalid_arguments_logic_error("function_name: invalid arguments");
  }
  ```

#### 10. Exception Handling

- **Exception Remapping:**  
  Lower layers throw exceptions with short names; upper layers should remap these exceptions as needed.
- **Simplified Exception Classes:**  
  Use our simplified exception hierarchy:  
  - **error** (base class)  
  - **logic_error**  
  - **invalid_arguments_logic_error**  
  - **data_error**
  
- **Usage Example:**
  ```cpp
  if (text == nullptr
          || text_length < 0
          || starting_position < 0
          || starting_position >= (index_of_code_unit_t::integer_type)text_length
          || clusters_to_process < 0) {
      throw invalid_arguments_logic_error("function_name: invalid arguments");
  }
  ```

#### 11. Code Organization

- **File Organization:**  
  Group related functions and definitions into files based on functionality. Use section separator comments to demarcate groups (e.g. anonymous namespaces, utility functions, primary namespace functions).
