# LocJSON

This is a specification for JSON-based translation interchange file format. It can be used in a bilingual mode in place of PO or XLIFF files to send strings for translation, and also as a monolingual resource file format.

The following requirements were taken into consideration. The file format should be:

1. Easy to parse/serialize (JSON has the widest support among programming languages);

2. Easy to traverse / amend (straightforward structure, no conditional blocks or alternative sub-structures);

3. Easy to read by a human; in canonical key sort order, the overall sequence of keys should make sense;

4. Easy to diff (when used in version control systems);

5. Easy to store large strings (arbitrary line splitting, as in GetText PO files);

6. Preserving unit order (i.e. not a key-value dictionary);

7. Capable of storing multi-line comments;

8. Suitable for bilingual and monolingual use (share the same basic structure);

9. Easy to extend to store extra properties, if needed for external tools.

# Status

-   **DRAFT**. The document is expected to be updated/clarified.

# Terminology

-   **originating application** — an application that creates LocJSON files for translation and that consumes localized copies of such a file.
-   **translation tool** — a TMS (Translation Management System), CAT (Computer-Aided Translation) tool, or some other tool that takes LocJSON files and updates them with translations.

# Specification

1. LocJSON files are always pretty-printed.
2. Indentation is always 4 spaces.
3. All keys in dictionaries are always sorted alphabetically.
4. All line breaks inside strings are represented with a single `\n` symbol (Unix-style).
5. LocJSON files use the `.locjson` file extension.

## Top-level structure

The top-level structure represents a dictionary with exactly two keys:

```json
{
    "properties" : {
        ...
    },
    "units" : [
        ...
    ]
}
```

1. `properties` [optional, dictionary] — information related to the file as a whole.
2. `units` [required, array] — an ordered list of translatable units (segments).

### Top-level `properties` block contents

This dictionary stores information related to the file as a whole:

```json
    "properties": {
        "comments": ["This file was generated by AwesomeTool"],
        "version": 1
    }
```

1. `comments` [optional, array of strings] — a list of file-wide comments. To render a multiline comment string, the array is concatenated with a single newline (`\n`) character.
2. `version` [optional, number] — the LocJSON file format version. If omitted, `1` is implied.

#### Notes on versioning

File format version is always an integer number, and will only increase if any backward-incompatible changes are introduced. But the intent is to always stay at version `1` and not introduce any backward-incompatible changes.

### Top-level `units` block

Each unit in this array is a dictionary:

```json
        ...
        {
            "key": "someUniqueKey",
            "properties": {
                ...
            },
            "source": [
                "Line 1\n",
                "Line 2\n",
                "\n",
                "A very very very long line split into several ",
                "based on a 50-character limit."
            ]
        },
        ...
```

1. `key` [required, string] — a unique (within that file) identifier of the string.
2. `properties` [optional, dictionary] — properties of a particular unit.
3. `source` [required<sup>1</sup>, array] — an array of strings that, if concatenated together, comprises a source text to translate. Lines must be split after `\n`. It is also recommended to split long strings into multiple ones, each of 50 symbols or less, including two symbols for the line break symbol (`\n`).
4. `target` [optional<sup>1</sup>, array] — an array of strings that, if concatenated together, comprises a target (translated) text. The same line splitting rules apply as with `source`.

<sup>1</sup> In bilingual use, it is expected to have both `source` and `target` fields present, and translation goes into `target` field; in monolingual use, a translation tool must put the translation back into `source` field when generating a localized copy of a file and omit the `target` field.

### Per-unit `properties` block contents

This dictionary stores information related to the file as a whole:

```json
            "properties": {
                "comments": [
                    "Unit comment line 1",
                    "Unit comment line 2"
                ]
            }
```

1. `comments` [optional, array of strings] — a list of unit-specific comments. To render a multiline comment string, the array is concatenated with a single newline (`\n`) character.

## Application-specific extensions

An application that generates a LocJSON file for translation may want to pass some extra proprietary information in the file. This is allowed only in `properties` dictionaries (both at top level and for each unit). All application-specific keys must start with an `x-` prefix, followed by a tool-specific sub-prefix, forming a namespace for that particular tool.

Consider the following example, where an imaginary tool, _AwesomeTool_, adds its own internal identifiers that start with `x-awesometool-` prefix:

```json
{
    "properties": {
        "comments": ["This file was generated by AwesomeTool 2.34"],
        "x-awesometool-generator-version": "2.34",
        "x-awesometool-file-id": "xf-12345",
        "x-awesometool-original-filename": "resources.js"
    },
    "units": [
        {
            "key": "testKey",
            "properties": {
                "comments": ["This is a comment."],
                "x-awesometool-unit-id": "xu-65432"
            },
            "source": ["Some text."]
        }
    ]
}
```

# Translation tool behavior

A translation tool must keep the entire structure of a LocJSON file intact. It is only allowed to add, remove, or modify the contents of a `target` array in each unit definition (or only modify the `source` array in case of monolingual use).

A translation tool may read and use other properties, including the ones that start with `x-` (for example, show them in the translation UI as an additional context).

A translation tool must not add any custom properties, reorder units, or reformat the contents of `source` or `target` array in any unit unless it modifies a particular translation for that unit.

A translation tool may only modify it's own known properties (i.e. the ones that _pre-existed_ in the LocJSON file). This gives an originating application that generates a LocJSON file to control the set of properties it supports, and ensures it can parse the returned file. For example, if LocJSON is generated for an imaginary translation tool `Foo`, and it is known that this tool supports a property `x-foo-fuzzy` (which also has an equivalent in an originating application), then an originating application can include `x-foo-fuzzy` in LocJSON file, and this property will become a part of a contract between an originating application and a translation tool.

A translation tool may be explicitly instructed to remove all `properties` keys (both file-level and unit-level) upon generating a localized version of a file. This reduces the file size of all resources and keeps only the minimal data needed (an array of units with keys and translations; see the _Minimal example_ section below). A translation tool should never remove `properties` keys by default.

# Minimal example

Given the optional nature of `properties` blocks, a minimal generated LocJSON file would look like this:

```json
{
    "units": [
        {
            "key": "key1",
            "source": ["String 1"],
        },
        {
            "key": "key2",
            "source": ["String 2"],
        },
        ...
    ]
}
```

## Bilingual use

In a bilingual use, a localized copy returned by a translation tool would look like this:

```json
{
    "units": [
        {
            "key": "key1",
            "source": ["String 1"],
            "target": ["Translated string 1"],
        },
        {
            "key": "key2",
            "source": ["String 2"],
            "target": ["Translated string 2"],
        },
        ...
    ]
}
```

## Monolingual use

In a monolingual use, a localized copy returned by a translation tool (with translations written directly into `source`) would look like this:

```json
{
    "units": [
        {
            "key": "key1",
            "source": ["Translated string 1"],
        },
        {
            "key": "key2",
            "source": ["Translated string 2"],
        },
        ...
    ]
}
```

There are two reasons translations are written directly into `source` in this mode:

1. The structure between source and localized files stays the same to simplify resource file handling.
2. A localized file can immediately serve as a source file to localize it into other languages (for example, a Chinese source file is translated into English first, and English one is then translated into all other languages).

# Full example

Here's a fuller example of LocJSON file that has comments:

```json
{
    "properties": {
        "comments": ["This file was generated by AwesomeTool"],
        "version": 1
    },
    "units": [
        {
            "key": "welcomeMessage",
            "properties": {
                "comments": ["{USER} here is replaced with the first name of a signed in user"]
            },
            "source": ["Hello, {USER}!"]
        },
        {
            "key": "signInButtonCaption",
            "properties": {
                "comments": ["https://example.com/preview/sign-in-dialog-screenshot.png"]
            },
            "source": ["Sign In"]
        },
        {
            "key": "signInFooterText",
            "properties": {
                "comments": [
                    "This text is displayed below the sign in form.",
                    "https://example.com/preview/sign-in-dialog-screenshot.png"
                ]
            },
            "source": [
                "Please read our <a ",
                "href=\"https://example.com/legal/pp\">Privacy ",
                "Policy</a> and <a ",
                "href=\"https://example.com/legal/tos\">Terms of ",
                "Service</a>"
            ]
        }
    ]
}
```
