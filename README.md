# ASN.1 Person Example

This repository demonstrates a **minimal ASN.1 module** compiled with `asn1c`, and how to encode/decode a `Person` object between **XML** and **Unaligned PER (UPER)** using the generated `converter-example` utility.

## Install asn1c
Install `asn1c` from the forked repository:  
https://github.com/Trihydro/asn1_codec/tree/develop/asn1c_combined#installing-asn1c

---

## 1. ASN.1 Module

Save the following schema as `person.asn`:

    PersonModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

        Person ::= SEQUENCE {
            name UTF8String,
            age  INTEGER (0..150)
        }

    END

- **name** → UTF-8 encoded string  
- **age** → integer limited to 0–150  

---

## 2. Generate Code

Run `asn1c` to compile the ASN.1 module into C sources:

    asn1c -fcompound-names -fincludes-quoted -pdu=all person.asn

**Flags explained:**
- `-fcompound-names` → generate longer, less ambiguous C identifiers.  
- `-fincludes-quoted` → use `#include "file.h"` instead of `<file.h>`.  
- `-pdu=all` → generate encoder/decoder entry points for all top-level types.  

This produces `*.c` and `*.h` files.

---

## 3. Build Converter Example

Use the provided makefile to build `converter-example`:

    make -f converter-example.mk

This links your generated code with the ASN.1 runtime library.  
After building, verify with:

    ./converter-example -help

You should see support for XML (XER) and UPER.

---

## 4. Test Encoding and Decoding

### Input XML (`person.xml`)

    <Person>
      <name>Alice</name>
      <age>30</age>
    </Person>

### Encode XML → UPER

    ./converter-example -p Person -ixer -ouper person.xml > person.uper

- `-ixer` → input is XER (XML).  
- `-ouper` → output is UPER.  
- `-p Person` → specify the top-level ASN.1 type.

### Decode UPER → XML

    ./converter-example -p Person -iuper -oxer person.uper > decoded.xml

- `-iuper` → input is UPER.  
- `-oxer` → output is XER (XML).

The file `decoded.xml` should match the original input.

---

## 5. Inspect Encoded Data

Since UPER output is binary, you can inspect it with `xxd`:

    xxd -b person.uper   # show bits
    xxd person.uper      # show hex

---

## Summary

You now have a minimal flow:
1. Write ASN.1 schema (`person.asn`)  
2. Compile with `asn1c`  
3. Build `converter-example`  
4. Encode/decode between XML and UPER  

This provides a foundation for experimenting with more complex ASN.1 structures.
