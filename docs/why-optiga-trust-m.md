# Why OPTIGA Trust M is Essential for IoT Security

**Understanding the necessity of hardware root of trust in modern IoT systems**

---

## Table of Contents

1. [The Problem: Software-Only Security Fails](#the-problem-software-only-security-fails)
2. [The Solution: Hardware Root of Trust](#the-solution-hardware-root-of-trust)
3. [How OPTIGA Trust M Works](#how-optiga-trust-m-works)
4. [OPTIGA vs Alternatives](#optiga-vs-alternatives)
5. [When You Need OPTIGA](#when-you-need-optiga)
6. [Technical Specifications](#technical-specifications)

---

## The Problem: Software-Only Security Fails

### Attack Scenario 1: Private Key Extraction from Firmware

**Typical software-only approach:**

```c
// ❌ INSECURE: Private key embedded in firmware
const uint8_t device_private_key[32] = {
    0xC9, 0x0F, 0xDA, 0xA2, 0x21, 0x68, 0xC2, 0x34,
    0xC4, 0xC6, 0x62, 0x8B, 0x80, 0xDC, 0x1C, 0xD1,
    // ... 32 bytes total (ECDSA P-256 key)
};

void sign_device_message(uint8_t *data, size_t len) {
    uint8_t signature[64];

    // Key is loaded into RAM here!
    ecdsa_sign_p256(device_private_key, data, len, signature);

    send_to_server(data, signature);
}
```

**What can go wrong:**

#### Attack Vector 1: Firmware Dump
```bash
# Attacker uses debug probe (JTAG/SWD)
$ openocd -f psoc_edge.cfg
> flash read_bank 0 firmware_dump.bin

# Extract key from binary
$ strings firmware_dump.bin | grep -A 32 "private_key"
# ... or use binary analysis tools

# Result: Private key extracted in 5 minutes
```

#### Attack Vector 2: RAM Dump During Execution
```c
// During ecdsa_sign_p256() execution:
// Private key is in RAM at address 0x20001000

// Attacker pauses execution via debugger:
// (gdb) x/32xb 0x20001000
// 0xC9 0x0F 0xDA 0xA2 ... (KEY EXPOSED!)
```

#### Attack Vector 3: Side-Channel Power Analysis
```
Oscilloscope connected to MCU power rail:
┌─────────────────────────────────┐
│  Power consumption spikes       │
│  during ECDSA multiplication    │
│  reveal private key bits        │
└─────────────────────────────────┘

After ~1000 signature captures:
→ Statistical analysis → Private key recovered
```

**Real-World Impact:**

🔴 **One compromised device = ALL devices vulnerable**
- If all devices share same software key
- Attacker can impersonate entire fleet

🔴 **Permanent compromise**
- Once key is extracted, it can't be "unextracted"
- Firmware update doesn't help (key is already out)

🔴 **Supply chain vulnerability**
- Contract manufacturer could extract keys
- Malicious firmware update could exfiltrate keys

<!--
📸 IMAGE NEEDED: images/diagrams/software-key-attack-vectors.png
Description: Diagram showing MCU with arrows pointing to:
- Flash (firmware dump attack)
- RAM (debug probe attack)
- Power rail (side-channel attack)
Each with red X marks
Generate with: draw.io
Size: 800x600px
-->

---

### Attack Scenario 2: Cloud Credential Theft

**Example: AWS IoT Device**

```c
// ❌ INSECURE: AWS IoT credentials in flash
const char* aws_client_cert =
    "-----BEGIN CERTIFICATE-----\n"
    "MIICgDCCAiagAwIBAgIJAKpqJ...\n"
    // ... certificate ...
    "-----END CERTIFICATE-----\n";

const char* aws_private_key =
    "-----BEGIN PRIVATE KEY-----\n"
    "MIGHAgEAMBMGByqGSM49AgEG...\n"
    // ... PRIVATE KEY STORED IN FLASH! ...
    "-----END PRIVATE KEY-----\n";

void connect_to_aws(void) {
    aws_iot_mqtt_connect(
        .host = "xxxxxx.iot.us-east-1.amazonaws.com",
        .client_cert = aws_client_cert,
        .private_key = aws_private_key  // ❌ Vulnerable!
    );
}
```

**Attack Timeline:**

```
Day 1: Device deployed in field
Day 30: Attacker gains temporary physical access (5 minutes)
        → Uses flash programmer to dump firmware
        → Extracts client certificate + private key

Day 31: Attacker creates "shadow device" with stolen credentials
        → AWS IoT Core accepts it as legitimate device
        → Attacker can:
            ✗ Read sensor data from your account
            ✗ Send malicious commands to real device
            ✗ Exhaust AWS quotas (DoS)
            ✗ Pivot to other devices in same AWS account

Day 32: You discover suspicious activity
        → TOO LATE: Key is already compromised
        → Must revoke certificate (breaks real device!)
        → Must physically retrieve and re-flash ALL devices
```

**Cost of Breach:**
- Device recall: $50-500 per device
- Downtime: Lost revenue
- Reputation damage: Customer trust
- Regulatory fines: GDPR, HIPAA violations

---

### Attack Scenario 3: Firmware Reverse Engineering

**Common mistake: Obfuscation is NOT security**

```c
// ❌ FALSE SENSE OF SECURITY
#define KEY_PART_1 0x4F50  // "OP"
#define KEY_PART_2 0x5449  // "TI"
#define KEY_PART_3 0x4741  // "GA"
// ... developer thinks they're clever ...

static uint8_t deobfuscate_key(int index) {
    uint8_t obf[] = {0x12^0xAB, 0x34^0xCD, ...};
    return obf[index] ^ SECRET_XOR_MASK;
}
```

**Reality check:**

```bash
# Reverse engineering tools see through this instantly
$ radare2 firmware.elf
> aaa  # Auto-analyze
> pdf @ sym.deobfuscate_key
# ... shows XOR operation ...
# ... shows SECRET_XOR_MASK constant ...
# Key extracted in 10 minutes

# Even without source code:
$ strings firmware.bin | grep -E "[A-Za-z0-9+/]{40,}="
# Base64-encoded keys found immediately
```

**Why obfuscation fails:**
- 🔴 Static analysis tools reverse it automatically
- 🔴 Debugging reveals key at runtime
- 🔴 Just makes code harder to maintain (not more secure)
- 🔴 False confidence leads to poor security decisions

---

## The Solution: Hardware Root of Trust

### What Makes Hardware Different?

**Fundamental Principle:** Private key **NEVER** exists in software-accessible memory

```mermaid
graph LR
    subgraph "MCU (Untrusted Zone)"
        APP[Application]
        FLASH[Flash Memory]
        RAM[RAM]
    end

    subgraph "OPTIGA Trust M (Trusted Zone)"
        KEY[Private Key]
        CRYPTO[Crypto Engine]
        SECURE[Secure Storage]
    end

    APP -->|"Sign this data"| CRYPTO
    CRYPTO --> KEY
    KEY --> SECURE
    CRYPTO -->|"Here's the signature"| APP

    APP -.->|"❌ Cannot access"| KEY
    FLASH -.->|"❌ Cannot access"| KEY
    RAM -.->|"❌ Cannot access"| KEY

    style KEY fill:#ff6b6b
    style SECURE fill:#ff6b6b
    style CRYPTO fill:#4ecdc4
    style APP fill:#95e1d3
```

---

### OPTIGA Trust M Architecture

```mermaid
graph TB
    subgraph "OPTIGA Trust M Chip"
        subgraph "Secure Key Storage"
            SLOT0[Key Slot 0xE0F0<br/>Device Identity Key]
            SLOT1[Key Slot 0xE0F1<br/>User Key 1]
            SLOT2[Key Slot 0xE0F2<br/>User Key 2]
            SLOT3[Key Slot 0xE0F3<br/>User Key 3]
        end

        subgraph "Crypto Engine"
            ECDSA[ECDSA Sign/Verify<br/>P-256, P-384, P-521]
            ECDH[ECDH Key Exchange]
            AES[AES Encrypt/Decrypt<br/>128, 192, 256 bits]
            HASH[SHA-256/384/512]
            TRNG[True Random<br/>Number Generator]
        end

        subgraph "Security Features"
            TAMPER[Tamper Detection]
            ZERO[Key Zeroization]
            COUNTER[Usage Counters]
            ACCESS[Access Control]
        end

        SLOT0 --> ECDSA
        SLOT1 --> ECDSA
        SLOT2 --> ECDH
        SLOT3 --> AES
    end

    subgraph "I²C Interface"
        I2C[I²C Bus<br/>Data & Commands Only<br/>❌ Keys NEVER transmitted]
    end

    ECDSA --> I2C
    ECDH --> I2C
    AES --> I2C
    TRNG --> I2C

    MCU[MCU<br/>PSoC™ Edge E84] <--> I2C

    style SLOT0 fill:#e74c3c
    style SLOT1 fill:#e74c3c
    style SLOT2 fill:#e74c3c
    style SLOT3 fill:#e74c3c
    style TAMPER fill:#f39c12
    style ZERO fill:#f39c12
```

**Critical Security Boundaries:**

1. **Private keys NEVER leave secure storage**
   - Generated inside OPTIGA
   - Stored in tamper-resistant memory
   - Used only internally for crypto operations

2. **Only results cross the boundary**
   - Signatures (not the signing key)
   - Ciphertext (not the encryption key)
   - Public keys (safe to share by definition)

3. **Hardware-enforced access control**
   - Each key slot has usage policy (sign-only, encrypt-only, etc.)
   - Cannot be bypassed by software
   - Even OPTIGA firmware can't extract keys

---

### How OPTIGA Prevents Attacks

| Attack Type | Software-Only | OPTIGA Trust M | Explanation |
|-------------|---------------|----------------|-------------|
| **Flash Dump** | ❌ Key exposed | ✅ No key in flash | Key only exists in OPTIGA hardware |
| **RAM Dump** | ❌ Key in RAM during crypto | ✅ Never enters MCU RAM | All crypto happens inside OPTIGA |
| **Debug Probe (JTAG/SWD)** | ❌ Can read key from memory | ✅ JTAG can't access OPTIGA internals | Separate chip with isolated debug interface |
| **Power Analysis (SCA)** | ❌ Software AES vulnerable | ✅ Hardened crypto engine | OPTIGA designed to resist side-channel attacks |
| **Firmware Reverse Engineering** | ❌ Key embedded in binary | ✅ No key in firmware | Firmware contains only key ID (just a number) |
| **Chip Desoldering** | ❌ Flash can be read externally | ✅ Tamper detection + zeroization | Physical attacks trigger key erasure |
| **Firmware Update Exploit** | ❌ Update could leak key | ✅ Key remains in OPTIGA | Update never touches key storage |
| **Supply Chain Attack** | ❌ Manufacturer can extract key | ✅ Keys provisioned after manufacturing | Device-unique keys generated in field |

<!--
📸 IMAGE NEEDED: images/diagrams/optiga-attack-prevention.png
Description: Table/matrix visualization with green checkmarks for OPTIGA
Size: 1000x600px
Can be generated from the table above
-->

---

## How OPTIGA Trust M Works

### Secure Key Lifecycle

#### Phase 1: Key Generation (In Factory or First Boot)

```mermaid
sequenceDiagram
    participant M as MCU Application
    participant O as OPTIGA Trust M

    Note over M,O: Key Generation Request

    M->>O: Generate ECDSA P-256 key in slot 0xE0F1
    activate O

    Note over O: 1. True Random Number Generator<br/>generates private key
    Note over O: 2. Derive public key from private key
    Note over O: 3. Store private key in secure memory<br/>(marked as non-exportable)

    O->>M: Public key: 04 a1 b2 c3 ... (65 bytes)
    deactivate O

    Note over M: ✅ MCU receives public key only<br/>❌ MCU NEVER sees private key<br/>(not even during generation!)
```

**Key Security Properties:**

✅ **Private key never leaves OPTIGA**
- Generated by OPTIGA's internal TRNG
- Stored in tamper-resistant memory
- Marked as "non-exportable" in key metadata

✅ **Public key safe to share**
- Transmitted to MCU over I²C
- Can be stored in flash, cloud, certificates
- Used for signature verification (not a secret)

✅ **Even OPTIGA firmware can't export private key**
- Hardware access control blocks export operations
- Attempting to read private key returns error
- Only crypto operations (sign/decrypt) allowed

---

#### Phase 2: Signing Operation

```mermaid
sequenceDiagram
    participant M as MCU Application
    participant O as OPTIGA Trust M Crypto Engine
    participant K as Secure Key Storage

    Note over M,O: Sign Data Request

    M->>O: Sign hash 0xAB12CD... with key 0xE0F1
    activate O

    O->>K: Load private key from slot 0xE0F1
    activate K
    K->>O: Private key (internal only)
    deactivate K

    Note over O: ECDSA signature generation<br/>(private key used internally)

    O->>M: Signature: 30 45 02 21 ... (71-72 bytes)
    deactivate O

    Note over M: ✅ MCU receives signature<br/>❌ Private key stayed in OPTIGA

    Note over O: Private key is zeroized<br/>from working memory after use
```

**Why This is Secure:**

🔒 **Private key never transmitted over I²C**
- I²C bus only carries: hash (input) and signature (output)
- Even if attacker probes I²C lines, no key exposed

🔒 **Key remains in secure storage**
- Loaded into OPTIGA's crypto engine (internal RAM)
- Used for ECDSA calculation
- Zeroized from working memory after signing

🔒 **Signature proves authenticity**
- Only device with private key can generate valid signature
- Public key can verify signature (math guarantees)
- Signature doesn't reveal private key (ECDSA property)

---

### Real Code Comparison

#### ❌ Without OPTIGA (Insecure)

```c
// Software-only implementation (INSECURE)

// Private key stored in flash
const uint8_t device_private_key[32] = {
    0xC9, 0x0F, 0xDA, 0xA2, ...  // ❌ EXPOSED!
};

void sign_message_insecure(const char *message) {
    // 1. Hash message
    uint8_t hash[32];
    sha256(message, strlen(message), hash);

    // 2. Sign hash using software ECDSA
    uint8_t signature[64];

    // ❌ Private key loaded into RAM here!
    ecdsa_sign_p256(device_private_key, hash, signature);

    // 3. Send signature
    send_to_server(message, signature);

    // ⚠️ Vulnerabilities:
    // - device_private_key visible in flash dump
    // - Private key visible in RAM during execution
    // - Debug probe can extract key
    // - No tamper protection
}
```

**Attack Surface:**
```
Flash: [... device_private_key: 0xC9 0x0F ...] ❌ Attackable
       ↓ (loaded into RAM)
RAM:   [... 0x20001000: 0xC9 0x0F ...] ❌ Attackable
       ↓ (used in calculation)
CPU:   [ECDSA computation] ❌ Side-channel vulnerable
```

---

#### ✅ With OPTIGA (Secure)

```c
// OPTIGA Trust M implementation (SECURE)

#include "psa/crypto.h"

// Key ID (not the actual key!)
#define OPTIGA_DEVICE_KEY_ID 0xE0F1  // Just a number, not sensitive

void sign_message_secure(const char *message) {
    psa_status_t status;

    // 1. Hash message
    uint8_t hash[32];
    size_t hash_len;
    status = psa_hash_compute(
        PSA_ALG_SHA_256,
        (uint8_t*)message, strlen(message),
        hash, sizeof(hash),
        &hash_len
    );

    // 2. Sign hash using OPTIGA key
    uint8_t signature[72];
    size_t signature_len;

    // ✅ Key never enters MCU - signing happens in OPTIGA!
    status = psa_sign_hash(
        OPTIGA_DEVICE_KEY_ID,  // Key ID (reference, not key itself)
        PSA_ALG_ECDSA(PSA_ALG_SHA_256),
        hash, hash_len,
        signature, sizeof(signature),
        &signature_len
    );

    // 3. Send signature
    send_to_server(message, signature);

    // ✅ Security guarantees:
    // - Private key never in flash (only in OPTIGA)
    // - Private key never in MCU RAM (stays in OPTIGA)
    // - Debug probe can't extract key (isolated chip)
    // - Tamper detection protects key
}
```

**Secure Architecture:**
```
MCU Flash: [... OPTIGA_DEVICE_KEY_ID: 0xE0F1 ...] ✅ Safe (just ID)
           ↓ (PSA API call)
MCU RAM:   [... hash: 0xAB 0x12 ...] ✅ Safe (not the key)
           ↓ (I²C command: sign hash)
I²C Bus:   [Command: SIGN, KeyID: 0xE0F1, Hash: ...] ✅ Safe
           ↓
OPTIGA:    [Load key from secure storage] 🔒 Inaccessible
           [ECDSA sign internally] 🔒 Inaccessible
           [Return signature] ✅ Safe (not the key)
           ↓
MCU RAM:   [... signature: 0x30 0x45 ...] ✅ Safe
```

**Key Difference:** At no point does the private key enter MCU-accessible memory!

---

## OPTIGA vs Alternatives

### Detailed Comparison Matrix

| Feature | Software Crypto | TrustZone-M | OPTIGA Trust M | Secure MCU (Built-in SE) | HSM |
|---------|----------------|-------------|----------------|--------------------------|-----|
| **Key Storage Location** | Flash/RAM | Secure Flash | Dedicated chip | On-die secure memory | Dedicated module |
| **Physical Isolation** | ❌ No | ⚠️ Logical only | ✅ Yes (separate chip) | ✅ Yes (on-die) | ✅ Yes (separate hardware) |
| **Extractable via JTAG?** | ✅ Yes | ⚠️ If misconfigured | ❌ No | ❌ No | ❌ No |
| **Side-Channel Resistance** | ❌ Low | ⚠️ Medium | ✅ High (hardened) | ✅ High | ✅ Very High |
| **Tamper Detection** | ❌ No | ⚠️ Software-based | ✅ Hardware (zeroization) | ✅ Hardware | ✅ Hardware (advanced) |
| **Certifications** | None | PSA Level 1-2 | **CC EAL6+**, PSA Level 3 | Varies (usually CC EAL4+) | FIPS 140-2 Level 3+ |
| **Performance (ECDSA)** | Fast (~5ms) | Fast (~5ms) | Medium (~50ms I²C overhead) | Fast (~10ms) | Slow (network/PCIe latency) |
| **Cost (per unit)** | $ | $$ | **$$$** | $$$$ | $$$$$ |
| **Integration Effort** | Low | Medium (TrustZone config) | **Low (with TF-M)** | Medium-High | High |
| **Field Lifetime** | N/A | 10+ years | **15+ years** | 10-15 years | 20+ years |
| **Use Case** | Prototypes only | Moderate security | **High-security IoT** | Integrated solution | Data centers, servers |

<!--
📸 IMAGE NEEDED: images/diagrams/comparison-chart.png
Description: Visual comparison chart with color coding:
- Red (poor), Yellow (medium), Green (good)
- Highlight OPTIGA row
Size: 1200x800px
-->

---

### When to Choose Each Option

#### Software-Only Crypto (mbedTLS, etc.)

**✅ Use when:**
- Lab prototypes and development only
- No network connectivity (local-only device)
- Extremely cost-sensitive (< $0.10 BOM difference matters)
- Short lifetime (< 1 year disposable devices)

**❌ Avoid when:**
- Device connects to cloud/network
- Compliance requirements exist
- Device handles sensitive data
- Risk of physical access

**Example:** Internal R&D prototype, never deployed

---

#### TrustZone-M Only (No Secure Element)

**✅ Use when:**
- Moderate security requirements
- Very cost-constrained (but need basic protection)
- Performance critical (no I²C overhead acceptable)
- PSA Level 2 certification sufficient

**❌ Avoid when:**
- High-value devices or data
- Regulatory compliance requires CC EAL6+
- Physical tampering is a credible threat
- Device lifetime > 10 years with key rotation

**Example:** Smart home sensors with basic encryption, low attack incentive

---

#### OPTIGA Trust M (Recommended for IoT)

**✅ Use when:**
- IoT devices connecting to cloud (AWS IoT, Azure IoT, etc.)
- Device lifetime 5-15 years
- Compliance requirements (medical, automotive, industrial)
- Physical deployment in field (risk of tampering)
- High-value devices or data
- Device cloning is a business risk

**❌ Consider alternatives when:**
- Ultra-low-cost consumer devices (millions of units)
- Performance requirements incompatible with I²C latency
- Already using secure MCU with built-in SE

**Example:** Industrial IoT sensors, medical wearables, smart locks, automotive telematics

**PSoC™ Edge E84 Advantage:**
OPTIGA pre-integrated on evaluation board + TF-M integration = lowest effort implementation!

---

#### Secure MCU with Built-in SE

**✅ Use when:**
- Need integrated solution (MCU + SE on same die)
- Higher volumes justify NRE costs
- Vendor lock-in acceptable
- Moderate-to-high security requirements

**❌ Avoid when:**
- Need flexibility (separate SE allows MCU changes)
- Cost extremely sensitive
- Vendor diversity required

**Example:** Payment terminals, access control systems (high volume)

---

#### HSM (Hardware Security Module)

**✅ Use when:**
- Server/gateway/edge gateway (not IoT endpoint)
- FIPS 140-2 Level 3+ required
- Key management for fleet of devices
- High transaction volume

**❌ Avoid when:**
- IoT endpoint device (too expensive, large, power-hungry)
- Embedded system (form factor mismatch)

**Example:** Cloud server managing IoT device certificates, not the IoT devices themselves

---

## When You Need OPTIGA

### ✅ Strong Indicators You Should Use OPTIGA

#### 1. Network-Connected Devices

**If your device connects to:**
- Cloud platforms (AWS IoT, Azure IoT Hub, Google Cloud IoT)
- MQTT/CoAP brokers
- RESTful APIs over HTTPS
- Wi-Fi, Cellular, LoRaWAN, Zigbee networks

**Why OPTIGA matters:**
Network = Attack surface. Compromised credentials allow:
- Device impersonation (send false data)
- Command injection (control your device)
- Lateral movement (attack other devices in fleet)

**Example:**
Smart building HVAC controller → connects to building management system
- Without OPTIGA: Attacker dumps firmware → clones controller → shuts down all HVAC
- With OPTIGA: Each controller has unique hardware identity → cloning impossible

---

#### 2. Long-Lifetime Devices (> 5 Years)

**If your device:**
- Expected lifetime > 5-10 years
- Deployed in field (difficult/expensive to retrieve)
- Receives firmware updates over lifetime

**Why OPTIGA matters:**
- Threat landscape evolves (vulnerabilities discovered)
- Firmware updates increase attack surface
- Hardware keys remain secure even if firmware compromised

**Example:**
Smart meters deployed for 15 years
- Firmware vulnerabilities discovered in year 8
- Update required, but update mechanism needs secure authentication
- OPTIGA keys validate updates, prevent malicious firmware

---

#### 3. Compliance & Certification Requirements

**If your industry requires:**
- Medical: FDA 510(k), IEC 62304, HIPAA
- Automotive: ISO 26262, UNECE WP.29
- Industrial: IEC 62443, NIS Directive
- Payment: PCI DSS
- Government: FIPS 140-2/3, Common Criteria

**Why OPTIGA matters:**
Common Criteria EAL6+ certification provides:
- Third-party validation of security claims
- Assurance for regulators and customers
- Reduced liability risk

**Example:**
Medical wearable monitoring device
- HIPAA requires protection of patient health data
- OPTIGA provides cryptographic evidence of data protection
- Hardware key protection satisfies regulatory audits

---

#### 4. High-Value Assets

**If your device:**
- Controls physical systems (locks, valves, motors)
- Handles financial transactions
- Manages confidential data
- Failure causes safety risk or significant cost

**Why OPTIGA matters:**
Attack ROI calculation: If attack benefit > attack cost, you will be attacked
- OPTIGA raises attack cost to physical chip-level attacks (expensive, detectable)

**Example:**
Smart door locks
- Value of unauthorized entry: High (theft, privacy violation)
- Attack incentive: High
- OPTIGA makes cloning impractical (each lock has unique hardware key)

---

#### 5. Deployed in Untrusted Environments

**If your device is:**
- Accessible to public (street sensors, public transit)
- In remote locations (oil rigs, farms, pipelines)
- Handled by untrusted parties (shipping, contract manufacturing)

**Why OPTIGA matters:**
Assume physical access → attacker can:
- Dump firmware (software keys compromised)
- Attach debug probes (extract RAM keys)
- **BUT cannot extract OPTIGA keys** (hardware protected)

**Example:**
Agricultural IoT sensors in fields
- Anyone can physically access sensor
- Firmware dump is trivial with $10 programmer
- OPTIGA ensures keys remain secure despite physical access

---

### ⚠️ Cases Where OPTIGA May Be Overkill

#### 1. Closed-Network Devices

**If your device:**
- Never connects to external network
- Operates in controlled facility (no public access)
- Communication is local-only (no cloud)

**Consider:** TrustZone-M may be sufficient (but OPTIGA still provides defense-in-depth)

---

#### 2. Extremely Cost-Sensitive Consumer Devices

**If your device:**
- Target BOM < $2
- Volume > 10 million units
- Security is not a differentiator

**Consider:** Evaluate risk vs. cost. Note: Even "low-value" IoT devices are hijacked for botnets (Mirai, etc.)

**Alternative:** Negotiate with secure MCU vendor for built-in SE at volume pricing

---

#### 3. Short-Lifetime Disposables

**If your device:**
- Lifetime < 1 year
- Disposable (not worth attacking)
- No sensitive data

**Consider:** Software-only may be acceptable (but check compliance requirements)

---

## Technical Specifications

### OPTIGA Trust M Cryptographic Capabilities

#### Asymmetric Cryptography

| Algorithm | Key Sizes | Operations | Performance (PSoC™ E84) |
|-----------|-----------|------------|------------------------|
| **ECDSA** | NIST P-256, P-384, P-521 | Sign, Verify | Sign: ~50ms, Verify: ~80ms |
| **ECDSA** | Brainpool P-256r1, P-384r1, P-512r1 | Sign, Verify | Sign: ~55ms, Verify: ~85ms |
| **ECDH** | NIST P-256, P-384 | Key Agreement | ~60ms |
| **RSA** | 1024, 2048 bits | Sign, Verify, Encrypt, Decrypt | Sign: ~200ms (2048-bit) |

#### Symmetric Cryptography

| Algorithm | Key Sizes | Modes | Performance |
|-----------|-----------|-------|-------------|
| **AES** | 128, 192, 256 bits | ECB, CBC, CTR | ~1ms per 16-byte block |
| **AES-GCM** | 128, 192, 256 bits | AEAD | ~2ms per 16-byte block |
| **AES-CCM** | 128, 192, 256 bits | AEAD | ~2ms per 16-byte block |

#### Key Derivation & Hashing

| Function | Algorithms | Use Cases |
|----------|------------|-----------|
| **HMAC** | SHA-256, SHA-384, SHA-512 | Message authentication |
| **HKDF** | Based on HMAC-SHA-256 | Key derivation for secure channels |
| **TLS PRF** | TLS 1.2 PRF | TLS session key derivation |
| **Hash** | SHA-256, SHA-384, SHA-512 | General hashing, attestation |

#### Random Number Generation

| Type | Specification | Entropy Source |
|------|---------------|----------------|
| **TRNG** | NIST SP 800-90A/B/C compliant | Hardware entropy source |
| **Output** | Up to 256 bytes per request | High-quality randomness |

---

### Key Storage & Lifecycle

#### Key Slots

| Slot ID | Purpose | Key Type | Default Configuration |
|---------|---------|----------|----------------------|
| 0xE0F0 | Device Identity | ECC P-256 | Pre-provisioned (on some variants) |
| 0xE0F1 | User Key 1 | ECC/RSA | User-configurable |
| 0xE0F2 | User Key 2 | ECC/RSA | User-configurable |
| 0xE0F3 | User Key 3 | ECC/RSA | User-configurable |

#### Key Metadata (Configurable per Slot)

| Property | Options | Purpose |
|----------|---------|---------|
| **Key Usage** | SIGN, VERIFY, ENCRYPT, DECRYPT, DERIVE | Restrict key operations |
| **Algorithm** | ECDSA, ECDH, RSA, AES | Lock to specific algorithm |
| **Exportable** | YES / NO | Prevent private key export |
| **Change Access** | PIN, Session-based | Protect metadata updates |
| **Usage Counter** | 0 - 100,000+ | Track/limit key operations |

---

### Physical & Environmental Specifications

| Parameter | Specification | Notes |
|-----------|--------------|-------|
| **Package** | USON10 (3mm x 3mm) | Compact for space-constrained designs |
| **Operating Temperature** | -40°C to +105°C | Industrial grade |
| **Storage Temperature** | -40°C to +125°C | |
| **Data Retention** | 15+ years | At 25°C |
| **Endurance** | 100,000 sign/verify cycles | Per key slot |
| **Interface** | I²C (up to 1 MHz) | 400 kHz typical, 1 MHz capable |
| **Power Supply** | 1.62V - 5.5V | Wide range for various MCUs |
| **Power Consumption** | < 5 mA active, < 3 µA sleep | Energy-efficient |

---

### Security Certifications & Standards

| Certification | Level | Scope |
|--------------|-------|-------|
| **Common Criteria** | EAL6+ (high assurance) | Chip security evaluation |
| **IEC 62443-4-2** | Compliant | Industrial automation security |
| **AEC-Q100** | Grade 2 | Automotive qualification (-40°C to +105°C) |
| **PSA Certified** | Level 2/3 Ready | ARM Platform Security Architecture |
| **NIST Algorithms** | Validated | Cryptographic algorithm compliance |

---

## Summary

### Key Takeaways

**1. Software crypto ≠ Secure crypto**
- Private keys in software are fundamentally vulnerable
- Firmware dumps, debug probes, side-channels all expose keys
- Obfuscation provides false sense of security

**2. OPTIGA Trust M = Hardware isolation**
- Private keys generated and stored in tamper-resistant chip
- Keys NEVER enter MCU memory (Flash, RAM, debug interfaces)
- Crypto operations performed inside secure boundary

**3. PSoC™ Edge E84 = Complete platform**
- OPTIGA Trust M pre-integrated on evaluation board
- TF-M provides standard PSA Crypto API
- No manual I²C driver development required
- Production-ready security architecture out-of-box

**4. When to use OPTIGA:**
- ✅ Network-connected IoT devices
- ✅ Long lifetime (> 5 years) deployments
- ✅ Compliance/certification requirements
- ✅ High-value devices or data
- ✅ Untrusted physical environments

---

### Next Steps

**Return to main page**

👉 **[Back to README →](../README.md)**

---

**© 2025 TESA Training Program**
**Platform: Infineon PSoC™ Edge E84 with OPTIGA Trust M**
