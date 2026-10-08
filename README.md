# NetRPay - Network-Resilient Payment

A Spring Boot backend that demonstrates **offline payment packets routed through a Bluetooth-style mesh network**.

Imagine a basement with no internet connection. A sender creates a payment, encrypts it, and broadcasts the packet to nearby devices. The packet can hop from device to device until a bridge device reaches an internet connection and uploads it to the backend.

The backend decrypts, validates, deduplicates, and settles the payment.

This repository contains the **server side of the system**, along with a software simulator that allows the complete payment flow to be demonstrated on a single laptop without real Bluetooth hardware.

---

## Table of Contents

1. What this demo proves
2. How to run it
3. Demo flow
4. Architecture
5. Three hard problems and their solutions
6. File-by-file walkthrough
7. API reference
8. Tests
9. What's not real
10. Honest limitations

---

## What this demo proves

The system demonstrates three things working end to end:

1. **Secure payment transport through untrusted intermediaries**

   Payment data is encrypted using hybrid RSA-OAEP and AES-256-GCM encryption. Intermediate devices carry the encrypted packet without being able to read the payment details.

2. **Exactly-once settlement under duplicate delivery**

   The same payment can reach the backend simultaneously through multiple bridge nodes and still settle only once. This is handled using idempotency based on an atomic claim of the ciphertext hash.

3. **Tamper and replay protection**

   Tampered packets are rejected during authenticated decryption, while replayed packets are rejected using freshness validation and idempotency.

---

## How to run it

### Prerequisites

* JDK 17 or newer
* No external database required
* No Redis required
* Maven installation is not required because the Maven wrapper is included

Check Java:

```cmd
java -version
```

### Run on Windows

Open a terminal in the project folder:

```cmd
mvnw.cmd spring-boot:run
```

### Run on Mac/Linux

```bash
./mvnw spring-boot:run
```

### Open the dashboard

Once the application starts, open:

```text
http://localhost:8080
```

The dashboard provides controls for simulating the complete payment flow.

### Stop the server

Press:

```text
Ctrl+C
```

### Run the tests

```cmd
mvnw.cmd test
```

---

## Demo flow

### Step 1 - Send a payment

Choose the sender, receiver, amount, and PIN.

Click **Send Payment**.

The backend:

1. Creates a `PaymentInstruction`.
2. Generates a unique nonce and timestamp.
3. Encrypts the payment using hybrid encryption.
4. Creates a `MeshPacket`.
5. Assigns the packet a TTL.
6. Injects the packet into the simulated network.

The packet is initially held by `phone-alice`.

### Step 2 - Run gossip rounds

Click **Run Gossip Round**.

Devices holding packets broadcast them to other simulated devices within range.

The simulator treats all virtual devices as being within Bluetooth range.

The packet TTL decreases as it travels through the network.

### Step 3 - Bridge reaches the internet

Click **Bridges Upload to Backend**.

`phone-bridge` is configured as the simulated device with internet access.

It uploads packets to:

```text
POST /api/bridge/ingest
```

The backend then:

1. Calculates the SHA-256 hash of the ciphertext.
2. Attempts to claim the hash in the idempotency cache.
3. Decrypts the packet if the claim succeeds.
4. Validates packet freshness.
5. Performs the debit and credit inside a database transaction.
6. Records the transaction in the ledger.

### Step 4 - Demonstrate idempotency

The automated concurrency test demonstrates the strongest duplicate-delivery scenario.

Run:

```cmd
mvnw.cmd test -Dtest=IdempotencyConcurrencyTest#singlePacketDeliveredByThreeBridgesSettlesExactlyOnce
```

The test delivers the same packet simultaneously from three threads and verifies:

* exactly one `SETTLED`
* two `DUPLICATE_DROPPED`
* the sender is debited exactly once

---

## Architecture

```text
                     SENDER PHONE
                          |
                   PaymentInstruction
                          |
                  Hybrid Encryption
                          |
                      MeshPacket
                          |
                  Bluetooth-style
                      gossip
                          |
             +------------+------------+
             |            |            |
         Device 1      Device 2      Bridge
                                      |
                                Internet access
                                      |
                                   HTTPS
                                      |
                                      v
             +--------------------------------------+
             |          SPRING BOOT BACKEND         |
             |                                      |
             |       /api/bridge/ingest             |
             |                |                     |
             |                v                     |
             |       SHA-256 ciphertext hash        |
             |                |                     |
             |                v                     |
             |       IdempotencyService             |
             |                |                     |
             |                v                     |
             |       HybridCryptoService            |
             |                |                     |
             |                v                     |
             |       Freshness validation            |
             |                |                     |
             |                v                     |
             |       SettlementService              |
             |                |                     |
             |        Debit + Credit + Ledger        |
             +--------------------------------------+
```

---

## The three hard problems and how they're solved

### Problem 1: Untrusted intermediaries

A random device may carry the transaction packet.

How do we prevent it from reading or modifying the payment?

### Solution: Hybrid encryption

The system uses:

* RSA-OAEP
* AES-256-GCM

The sender encrypts the payment payload using the server's public key.

The implementation:

1. Generates a fresh AES-256 key.
2. Encrypts the payment JSON using AES-256-GCM.
3. Encrypts the AES key using RSA-OAEP.
4. Combines the encrypted key, IV, and ciphertext into the final packet.

AES-GCM also provides authentication. If the ciphertext is modified, authentication fails and the packet is rejected.

See:

```text
HybridCryptoService.java
```

---

### Problem 2: Duplicate delivery

Multiple bridge devices may upload the same payment at almost the same time.

Without idempotency, the same payment could potentially be processed more than once.

### Solution: Atomic ciphertext-hash claim

The backend calculates:

```text
SHA-256(ciphertext)
```

and attempts to claim the resulting hash before processing the payment.

The implementation uses:

```java
ConcurrentHashMap.putIfAbsent()
```

This operation is atomic.

Only one concurrent request can claim a particular hash.

The other requests are returned as:

```text
DUPLICATE_DROPPED
```

In a distributed production deployment, this JVM-local mechanism could be replaced with a distributed store such as Redis using an equivalent atomic `SET NX EX` operation.

The transaction table also has a unique constraint on `packet_hash` as an additional defense.

---

### Problem 3: Replay attacks

An attacker could capture an encrypted packet and attempt to replay it later.

### Solution: Freshness + idempotency

The encrypted payment payload cont


