# ASN.1 OCTET STRING Constraint Example

This repository demonstrates how changing the **maximum size of an OCTET STRING** in an ASN.1 schema causes a **breaking change**.  
It mirrors the asn1-person-example structure, but uses a `Message` type with two schema variants:

- 1–2 character limit  
- 1–3 character limit

Encoding/decoding between converters built from different schemas illustrates the incompatibility.

---

## Install asn1c
Install `asn1c` from the forked repository:  
https://github.com/Trihydro/asn1_codec/tree/develop/asn1c_combined#installing-asn1c

---

## 1. ASN.1 Schemas

message-1-2.asn:
```
MessageModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

    Message ::= SEQUENCE {
        advisoryMessage OCTET STRING (SIZE(1..2))
    }

END
```

message-1-3.asn:
```
MessageModule DEFINITIONS AUTOMATIC TAGS ::= BEGIN

    Message ::= SEQUENCE {
        advisoryMessage OCTET STRING (SIZE(1..3))
    }

END
```

- advisoryMessage → binary blob or UTF-8 text, limited by schema constraint.

---

## 2. Generate Code and Build Converters

We will generate code and build both converters from a single directory to minimize switching:

1. Create separate output directories for each variant:
```
mkdir -p build/converter-1-2
mkdir -p build/converter-1-3
```

2. Generate C sources with asn1c:

For 1..2 constraint:
```
asn1c -fcompound-names -fincludes-quoted -pdu=all -output build/converter-1-2 message-1-2.asn
```

For 1..3 constraint:
```
asn1c -fcompound-names -fincludes-quoted -pdu=all -output build/converter-1-3 message-1-3.asn
```

3. Build the converters using provided makefiles:
```
make -C build/converter-1-2 -f converterexample.mk
make -C build/converter-1-3 -f converterexample.mk
```

4. Verify:
```
build/converter-1-2/converter -help
build/converter-1-3/converter -help
```

Both should support XML (XER) and UPER.

---

## 3. Test Messages

msg-1-2.xml:
```
<Message>
  <advisoryMessage>AB</advisoryMessage>
</Message>
```

msg-1-3.xml:
```
<Message>
  <advisoryMessage>ABC</advisoryMessage>
</Message>
```

---

## 4. Encode / Decode Tests

Encode XML → UPER:
```
build/converter-1-2/converter -p Message -ixer -ouper msg-1-2.xml > msg-1-2.uper  
build/converter-1-3/converter -p Message -ixer -ouper msg-1-3.xml > msg-1-3.uper
```

Decode UPER → XML:

Works:
```
build/converter-1-2/converter -p Message -iuper -oxer msg-1-2.uper > decoded-1-2.xml  
build/converter-1-3/converter -p Message -iuper -oxer msg-1-3.uper > decoded-1-3.xml
```

Fails:
```
build/converter-1-3/converter -p Message -iuper -oxer msg-1-2.uper  
build/converter-1-2/converter -p Message -iuper -oxer msg-1-3.uper
```

---

## 5. Observed Results

| Message     | Encoded With | Decoded With | Result | Notes |
|-------------|--------------|--------------|--------|-------|
| msg-1-2     | 1..2         | 1..2         | ✅     | Fits within original constraint |
| msg-1-2     | 1..2         | 1..3         | ❌     | Schema mismatch → decoding fails |
| msg-1-3     | 1..3         | 1..3         | ✅     | Fits within new constraint |
| msg-1-3     | 1..3         | 1..2         | ❌     | Exceeds old constraint → decoding fails |

---

## 6. Key Insight

Changing the advisoryMessage constraint from 1..2 to 1..3 characters is **not backward compatible**:

- A decoder compiled for 1..3 cannot read messages encoded with 1..2 exceeding its limits.  
- A decoder compiled for 1..2 cannot read messages encoded with 1..3 exceeding its limits.  

Messages must be encoded and decoded with the same schema.

---

## Summary

This minimal example reproduces the **OCTET STRING constraint breaking change**:

1. Generate code for both schemas in separate output directories (build/converter-1-2, build/converter-1-3)  
2. Build two converters  
3. Encode/decode test messages of lengths 2 and 3  
4. Observe compatibility failures when schema constraints differ
