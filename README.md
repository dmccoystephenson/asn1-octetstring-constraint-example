# ASN.1 OCTET STRING Constraint Example

This repository demonstrates how changing the **maximum size of an OCTET STRING** in an ASN.1 schema causes a **breaking change**.  
It mirrors the asn1-person-example structure, but uses a `Message` type with two schema variants:

- 1–1 character limit  
- 1–2 character limit

Encoding/decoding between converters built from different schemas illustrates the incompatibility.

---

## Install asn1c
Install `asn1c` from the forked repository:  
https://github.com/Trihydro/asn1_codec/tree/develop/asn1c_combined#installing-asn1c

---

## 1. ASN.1 Schemas

### message-1.asn
MessageModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN
```
    Message ::= SEQUENCE {
        advisoryMessage OCTET STRING (SIZE(1..1))
    }

END
```

### message-2.asn
```
MessageModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

    Message ::= SEQUENCE {
        advisoryMessage OCTET STRING (SIZE(1..2))
    }

END
```

- advisoryMessage → binary blob or UTF-8 text, limited by schema constraint.

---

## 2. Directory Structure

To keep the two converters separate, compile each ASN.1 schema in its own directory:
```
asn1-octetstring-constraint-example/
- converter-1/
  - message-1.asn
  - *.c, *.h (generated)
  - converter-1.mk
- converter-2/
  - message-2.asn
  - *.c, *.h (generated)
  - converter-2.mk
- msg-1.xml
- msg-2.xml
```

---

## 3. Generate Code

Navigate into each directory and compile:

In converter-1:
```
asn1c -fcompound-names -fincludes-quoted -pdu=all message-1.asn
```

In converter-2:
```
asn1c -fcompound-names -fincludes-quoted -pdu=all message-2.asn
```

This produces *.c and *.h files for each variant within its directory.

---

## 4. Build Converter Examples

In each directory, run:
```
make -f converter-example.mk    # in converter-1
cp converter-example converter1
make -f converter-example.mk    # in converter-2
cp converter-example converter2
```

Verify with:
```
./converter1 -help
./converter2 -help
```

Both should support XML (XER) and UPER.

---

## 5. Test Messages

msg-1.xml:
```
<Message>
  <advisoryMessage>A</advisoryMessage>
</Message>
```

msg-2.xml:
```
<Message>
  <advisoryMessage>AB</advisoryMessage>
</Message>
```

---

## 6. Encode / Decode Tests

Encode XML → UPER:
```
./converter-1/converter1 -p Message -ixer -ouper msg-1.xml > msg-1.uper  
./converter-2/converter2 -p Message -ixer -ouper msg-2.xml > msg-2.uper
```

Decode UPER → XML:

Works:
```
./converter-1/converter1 -p Message -iuper -oxer msg-1.uper > decoded-1.xml  
./converter-2/converter2 -p Message -iuper -oxer msg-2.uper > decoded-2.xml
```

Fails:
```
./converter-2/converter2 -p Message -iuper -oxer msg-1.uper  
./converter-1/converter1 -p Message -iuper -oxer msg-2.uper
```

---

## 7. Observed Results

| Message | Encoded With | Decoded With | Result | Notes |
|---------|--------------|--------------|--------|-------|
| msg-1   | 1            | 1            | ✅     | Fits within original constraint |
| msg-1   | 1            | 2            | ❌     | Schema mismatch → decoding fails |
| msg-2   | 2            | 2            | ✅     | Fits within new constraint |
| msg-2   | 2            | 1            | ❌     | Exceeds old constraint → decoding fails |

---

## 8. Key Insight

Changing the advisoryMessage constraint from 1..1 to 1..2 characters is **not backward compatible**:

- A decoder compiled for 1..2 cannot read 1..1-encoded messages exceeding its max.  
- A decoder compiled for 1..1 cannot read 1..2-encoded messages exceeding its max.  

Messages must be encoded and decoded with the same schema.

---

## Summary

This minimal example reproduces the **OCTET STRING constraint breaking change**:

1. Compile each schema in its own directory (converter-1, converter-2)  
2. Build two converters (converter-1, converter-2)  
3. Encode/decode test messages of lengths 1 and 2  
4. Observe compatibility failures when schema constraints differ
