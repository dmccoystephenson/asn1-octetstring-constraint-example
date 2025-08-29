# ASN.1 OCTET STRING Constraint Example

This repository demonstrates how changing the **maximum size of an OCTET STRING** in an ASN.1 schema causes a **breaking change**.  
It mirrors the [asn1-person-example](https://github.com/dmccoystephenson/asn1-person-example) structure, but uses a `Message` type with two schema variants:

- **1400-character limit**  
- **7000-character limit**

Encoding/decoding between converters built from different schemas illustrates the incompatibility.

---

## Install asn1c
Install `asn1c` from the forked repository:  
https://github.com/Trihydro/asn1_codec/tree/develop/asn1c_combined#installing-asn1c

---

## 1. ASN.1 Schemas

### message-1400.asn
MessageModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

    Message ::= SEQUENCE {
        advisoryMessage OCTET STRING (SIZE(1..1400))
    }

END

### message-7000.asn
MessageModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

    Message ::= SEQUENCE {
        advisoryMessage OCTET STRING (SIZE(1..7000))
    }

END

- **advisoryMessage** → binary blob or UTF-8 text, limited by schema constraint.

---

## 2. Generate Code

Compile each schema separately:

### For the 1400-character schema
```
asn1c -fcompound-names -fincludes-quoted -pdu=all message-1400.asn
```

### For the 7000-character schema
```
asn1c -fcompound-names -fincludes-quoted -pdu=all message-7000.asn
```

This produces `*.c` and `*.h` files for each variant.

---

## 3. Build Converter Examples

Use the provided makefiles to build:
```
make -f converter-1400.mk
make -f converter-7000.mk
```
After building, verify with:
```
./converter-1400 -help
./converter-7000 -help
```
Both should support XML (XER) and UPER.

---

## 4. Test Messages

### msg-1400.xml
```
<Message>
  <advisoryMessage>AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA</advisoryMessage>
</Message>
*(repeat "A" until total length = 1400 characters)*
```

### msg-7000.xml
```
<Message>
  <advisoryMessage>BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB</advisoryMessage>
</Message>
*(repeat "B" until total length = 7000 characters)*
```

---

## 5. Encode / Decode Tests

### Encode XML → UPER
```
./converter-1400 -p Message -ixer -ouper msg-1400.xml > msg-1400.uper
./converter-7000 -p Message -ixer -ouper msg-7000.xml > msg-7000.uper
```

### Decode UPER → XML
# Works
```
./converter-1400 -p Message -iuper -oxer msg-1400.uper > decoded-1400.xml
./converter-7000 -p Message -iuper -oxer msg-7000.uper > decoded-7000.xml
```

# Fails
```
./converter-7000 -p Message -iuper -oxer msg-1400.uper
./converter-1400 -p Message -iuper -oxer msg-7000.uper
```

---

## 6. Observed Results

| Message   | Encoded With | Decoded With | Result | Notes                                 |
|-----------|--------------|--------------|--------|---------------------------------------|
| msg-1400  | 1400         | 1400         | ✅     | Fits within original constraint       |
| msg-1400  | 1400         | 7000         | ❌     | Schema mismatch → decoding fails      |
| msg-7000  | 7000         | 7000         | ✅     | Fits within new constraint            |
| msg-7000  | 7000         | 1400         | ❌     | Exceeds old constraint → decoding fails |

---

## 7. Key Insight

Changing the `advisoryMessage` constraint from 1400 to 7000 characters is **not backward compatible**:

- A decoder compiled for 7000 cannot read 1400-encoded messages.
- A decoder compiled for 1400 cannot read 7000-encoded messages.

**Messages must be encoded and decoded with the same schema.**

---

## Summary

This minimal example reproduces the **OCTET STRING constraint breaking change**:

1. Define two schemas (`message-1400.asn`, `message-7000.asn`).  
2. Build two converters (`converter-1400`, `converter-7000`).  
3. Encode/decode test messages of lengths 1400 and 7000.  
4. Observe compatibility failures when schema constraints differ.  
