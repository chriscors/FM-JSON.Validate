# `JSON.validateSchema` for FileMaker

A lightweight, "sample-shape" based custom function for validating, cleaning, and normalizing JSON data within the Claris FileMaker Platform.

## Overview

`JSON.validateSchema` provides a simple yet powerful way to ensure that incoming JSON data conforms to a predefined structure. Instead of requiring a complex, formal JSON Schema definition, this function uses a "sample shape" approach. You define the expected structure and data types by providing a template JSON (`schema`) that looks just like the data you want to receive.

This is particularly useful for:
- Validating API responses before processing.
- Sanitizing user-provided JSON.
- Enforcing a consistent data structure across your application.
- Normalizing data types (e.g., converting `"true"` to a proper boolean `true`).

## Features

- **Simple "Sample Shape" Validation**: Easy-to-write schemas that mirror your expected data.
- **Data Sanitization**: Cleans and normalizes data to match the schema's types. Extra fields in the input data are stripped out.
- **Recursive Validation**: Handles nested objects and arrays automatically.
- **Optional Fields**: Declare optional fields simply by using `JSONNull`.
- **Two Modes of Operation**:
  - **Strict Mode (`safeParse = false`):** Returns a detailed error object if validation fails.
  - **Safe Parse Mode (`safeParse = true`):** Attempts to coerce the data into the schema's shape, returning a partially valid object or an empty structure on failure.
- **Clear Error Reporting**: Pinpoints which field failed validation and why.
- **Self-Contained**: No dependencies on other custom functions or plugins.

---

## How It Works

The core principle is that the **type** of the value for each key in your `schema` defines the required type for the corresponding key in the `data`.

| Schema Value Type | Required Data Type | Behavior |
| :--- | :--- | :--- |
| `"any string"` | String | Coerces the data value to a string. |
| `123` or `45.6` | Number | Coerces the data value to a number. Fails if not a valid number. |
| `true` or `false` | Boolean | Coerces `"true"`/`1` to `true` and `"false"`/`0` to `false`. |
| `{ "key": "value" }` | Object | Recursively validates the nested object against the schema's object. |
| `[ "example" ]` | Array | Validates that the data is an array and that each item in the data array matches the type of the **first item** in the schema array (homogeneous arrays). |
| `JSONNull` | **Optional Field** | The key is optional. If present in the data, its value is accepted as-is without type checking. If absent, it is simply omitted from the output. |

---

## Installation

1.  Open the FileMaker file where you want to use this function.
2.  Go to **File > Manage > Custom Functions...**.
3.  Click **New...**.
4.  Set the **Function Name** to `JSON.validateSchema`.
5.  Copy the entire function code (from the provided `.fmp12` file or the source code block) and paste it into the main calculation pane.
6.  Click **OK** to save the function.

## Function Signature

```
JSON.validateSchema ( schema ; data ; safeParse )
```

### Parameters

-   `schema` {JSON}: A valid JSON object or array that acts as the template for validation.
-   `data` {JSON}: A valid JSON object or array that you want to validate and parse.
-   `safeParse` {Boolean}:
    -   `false` (or `0`): Strict mode. If validation fails, the function returns a JSON error object.
    -   `true` (or `1`): Safe Parse mode. If validation fails, the function will attempt to build a valid object by coercing types and omitting invalid or missing data.

### Return Value

-   **On Success**: A cleaned, normalized JSON object or array that perfectly matches the schema's structure.
-   **On Failure (`safeParse = false`)**: A JSON object with a specific error structure:
    ```json
    {
      "error": {
        "code": 959,
        "text": "Descriptive error message"
      }
    }
    ```
-   **On Failure (`safeParse = true`)**: A best-effort JSON object or array. Missing required fields will result in the parent object being returned empty (`{}`).

---

## Examples

### 1. Basic Success

The `data` matches the `schema` perfectly.

-   **Schema:**
    ```json
    {
      "name": "string",
      "age": 0,
      "isStudent": false
    }
    ```
-   **Data:**
    ```json
    {
      "name": "Jane Doe",
      "age": 30,
      "isStudent": true,
      "extraField": "this will be removed"
    }
    ```
-   **Call:** `JSON.validateSchema ( schema ; data ; false )`
-   **Result:**
    ```json
    {
      "name": "Jane Doe",
      "age": 30,
      "isStudent": true
    }
    ```

### 2. Nested Objects and Optional Fields

Using `JSONNull` to mark `middleName` as optional.

-   **Schema:**
    ```json
    {
      "id": 0,
      "name": {
        "first": "string",
        "middle": null,
        "last": "string"
      }
    }
    ```
-   **Data (with optional field):**
    ```json
    {
      "id": 123,
      "name": {
        "first": "John",
        "middle": "A",
        "last": "Smith"
      }
    }
    ```
-   **Result:**
    ```json
    {
      "id": 123,
      "name": {
        "first": "John",
        "middle": "A",
        "last": "Smith"
      }
    }
    ```
-   **Data (without optional field):**
    ```json
    {
      "id": 456,
      "name": {
        "first": "Mary",
        "last": "Jones"
      }
    }
    ```
-   **Result:**
    ```json
    {
      "id": 456,
      "name": {
        "first": "Mary",
        "last": "Jones"
      }
    }
    ```
    
### 3. Array Validation

The schema for an array requires a single element that defines the type for all items in the data array.

-   **Schema:**
    ```json
    [
      { "sku": "string", "quantity": 0 }
    ]
    ```
-   **Data:**
    ```json
    [
      { "sku": "ABC-123", "quantity": 5 },
      { "sku": "XYZ-789", "quantity": 2, "notes": "extra" }
    ]
    ```
-   **Call:** `JSON.validateSchema ( schema ; data ; false )`
-   **Result:**
    ```json
    [
      { "sku": "ABC-123", "quantity": 5 },
      { "sku": "XYZ-789", "quantity": 2 }
    ]
    ```

### 4. Validation Failure (Strict Mode)

A required field is missing and another has the wrong type.

-   **Schema:**
    ```json
    {
      "name": "string",
      "age": 0,
      "isStudent": false
    }
    ```
-   **Data:**
    ```json
    {
      "name": "Timmy",
      "isStudent": "true"
    }
    ```
-   **Call:** `JSON.validateSchema ( schema ; data ; false )`
-   **Result:** (Error reporting stops at the first failure)
    ```json
    {
      "error": {
        "code": 959,
        "text": "Field 'age' expected required field"
      }
    }
    ```
    
### 5. Coercion with `safeParse = true`

Using the same invalid data as above, `safeParse` will attempt to build a valid object.

-   **Schema:**
    ```json
    {
      "name": "string",
      "age": 0,
      "isStudent": false
    }
    ```
-   **Data:**
    ```json
    {
      "name": "Jane Doe",
      "age": "30",
      "isStudent": 1
    }
    ```
-   **Call:** `JSON.validateSchema ( schema ; data ; true )`
-   **Result:** (`age` is converted to a number, `isStudent` is converted to a boolean)
    ```json
    {
      "name": "Jane Doe",
      "age": 30,
      "isStudent": true
    }
    ```
-   **Data (with missing required field):**
    ```json
    {
      "name": "Timmy"
    }
    ```
-   **Result:** (Since a required field is missing, the entire object construction fails and an empty object is returned)
    ```json
    {}
    ```
    
## Limitations

-   **Homogeneous Arrays**: Arrays are validated based on the type of the *first element* in the schema array. The function does not support arrays with mixed types (e.g., `[ "string", 123, true ]`).
-   **Not a Full JSON Schema Implementation**: This function does not support complex validation rules like regular expressions (`pattern`), string lengths (`minLength`), number ranges (`minimum`), or `enum`. It is designed for shape and type validation only.
-   **First Error Only**: In strict mode (`safeParse = false`), validation stops and reports the very first error it encounters. It does not return a list of all validation errors.

---

## Author & History

-   **Author:** Chris Corsi (chris.corsi@proofgeist.com)
-   **Created:** 2025-06-06