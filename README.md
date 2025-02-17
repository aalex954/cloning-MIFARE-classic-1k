# Cloning MIFARE Classic 1k Cards
A technical walkthrough for extracting authentication keys and either emulating or cloning MIFARE classic 1k cards.

> **Legal Disclaimer:**  
> This guide is intended solely for educational purposes and for use in authorized security testing. Any unauthorized use, cloning, or tampering with RFID systems is illegal and unethical. Always obtain proper authorization before performing any penetration tests.

In today's threat landscape, legacy RFID systems, particularly the MIFARE Classic 1K, continue to remain prevelant despite decades of known vulnerabilities. This guide distills both theoretical and hands-on expertise into a technical walkthrough for extracting authentication keys and either emulating or cloning these cards. It covers historical context, inherent vulnerabilities, advanced attack methodologies, and defensive mitigations, all while underscoring the importance of maintaining up-to-date security practices.

> **Note:** MIFARE Classic 1K cards operate at **13.56 MHz**

---

## Executive Summary

This guide provides a comprehensive overview of techniques for cloning MIFARE Classic 1K cards, a legacy RFID technology still widely deployed despite its vulnerabilities. It details the process of extracting authentication keys and emulating or cloning the card using tools like the Flipper Zero and custom firmware. Additionally, it discusses alternative tools such as the Proxmark3 for handling specific scenarios (e.g., “Static Encrypted” cards), outlines multiple attack methodologies, and recommends robust mitigation strategies.

**Key Highlights:**

- **Background & Vulnerabilities:**  
  MIFARE Classic 1K cards, structured into 16 sectors and operating at 13.56 MHz, utilize the proprietary CRYPTO1 algorithm. Its limited key space and predictable challenge-response mechanism, combined with the frequent use of default keys and fixed UIDs, have led to the emergence of UID changeable “magic” cards.

- **Attack Techniques:**  
  The guide explores several advanced attack methodologies:
  - **Dictionary Attack:** Uses a precompiled list of common keys to unlock each sector.
  - **Mfkey32 Attack:** Captures cryptographic nonces during the reader’s handshake, then processes them, often combining statistical and brute-force methods to derive additional keys.
  - **Nested Attack:** Exploits partially authenticated sessions by reusing residual nonces from previous interactions.
  - **KDF Attack:** Utilizes dedicated plugins to crack specific key derivation functions used by some card brands.

- **Emulation & Cloning:**  
  Once all sector keys are recovered, the guide explains how to emulate the card using the Flipper Zero or clone its data onto a physical “magic” card. It also addresses challenges such as timing discrepancies in emulation, recommending firmware updates and other adjustments to resolve these issues.

- **Mitigation Strategies:**  
  To counter these vulnerabilities, the guide advocates upgrading to modern, cryptographically secure systems, enforcing strong key management, and adopting multi-factor authentication. It further recommends incorporating network segmentation and physical security measures, as well as conducting regular security audits and firmware/software updates.

---

## 1. Background: MIFARE Classic 1K Technology

MIFARE is a series of integrated circuit (IC) chips used in contactless smart cards and proximity cards. The brand includes proprietary solutions based on various levels of the ISO/IEC 14443 Type-A 13.56 MHz contactless smart card standard. It uses AES and DES/Triple-DES encryption standards, as well as an older proprietary encryption algorithm, Crypto-1. According to NXP, 10 billion of their smart card chips and over 150 million reader modules have been sold.

The MIFARE Classic IC is a basic memory storage device, where the memory is divided into segments and blocks with simple security mechanisms for access control. They are ASIC-based and have limited computational power. Due to their reliability and low cost, those cards are widely used for electronic wallets, access control, corporate ID cards, transportation or stadium ticketing. It uses an NXP proprietary security protocol (Crypto-1) for authentication and ciphering.

The MIFARE Classic with 1K memory offers 1,024 bytes of data storage, split into 16 sectors; each sector is protected by two different keys, called A and B. Each key can be programmed to allow operations such as reading, writing, increasing value blocks, etc.

MIFARE Classic 1K cards operate at 13.56 MHz and are divided into 16 sectors, each typically comprising 4 blocks (with block 0 reserved for the immutable Unique Identifier (UID) and manufacturer data). Authentication relies on the proprietary CRYPTO1 algorithm. Over time, extensive research has exposed several weaknesses:

- **Weak Cryptography:**  
  The CRYPTO1 algorithm’s limited key space makes brute-force, dictionary, and nested attacks feasible.

- **Default & Reused Keys:**  
  Many deployments still use default keys (e.g., `FFFFFFFFFFFF`), analogous to default admin credentials, greatly simplifying unauthorized access.

- **UID Immutability:**  
  Although a fixed UID enhances security by ensuring consistent identity, it complicates cloning, which has led to the development of “magic” or UID changeable cards—often exploiting [undocumented backdoors(https://www.keysight.com/blogs/en/tech/nwvs/2024/08/27/security-highlight-backdoor-key-found-in-mifare-classic-cards).

**Security Controls & Mitigations:**
- **Key Diversification:** Employ unique keys for each card and sector.
- **Strong Key Management:** Replace default keys with cryptographically secure alternatives.
- **Multi-Factor Authentication:** Supplement card-based access with additional verification methods.
- **Firmware & Protocol Enhancements:** Regularly update and patch readers and systems to counter timing, replay, and other vulnerabilities.

---

## 2. Required Tools

- **Flipper Zero:** A versatile, open-source multi-tool for interacting with various digital protocols.
- **Unleashed Firmware:** Custom firmware that unlocks advanced NFC functionalities on the Flipper Zero.
- **13.56 MHz Writable Cards/FOBs:** Often known as “magic” cards, these devices have an unlocked sector zero and allow UID modification.
- **Alternative Tools:**  
  Tools such as the Proxmark3 can be used for specialized tasks (e.g., processing “Static Encrypted” cards) and may complement the methods described here.

---

## 3. Step-by-Step Process

### Step 1: Identify the Card

#### **Objective:**  
Confirm that the target is a MIFARE Classic 1K (or variant) and gather initial intelligence.

<img src="https://github.com/user-attachments/assets/d2800075-8a14-4531-b2b4-c5ef65077179" alt="drawing" width="200" height="200" align="right"/>
<img src="https://github.com/user-attachments/assets/61721bfa-7833-4ea6-bb52-a07506c21225" alt="drawing" width="200" height="200" align="right"/>


#### **Methodology:**  

- **Physical Inspection & Reverse Image Search:** Examine the card for logos, markings, or printed information. Reverse image searches can reveal the card’s model and known vulnerabilities.
- **Flipper Zero Scanning:** Use the device’s NFC scanning capabilities to read basic information (e.g., UID and manufacturer data), laying the groundwork for subsequent key extraction.

---

### Step 2: Cracking the Keys

The aim is to recover the secret keys from each sector. Without these keys, the card reader’s authentication handshake will fail, preventing access to the card’s data.

#### 2.1 Dictionary Attack

##### **Rationale:**  
Many cards utilize default or commonly reused keys. A dictionary attack leverages a precompiled list of these keys to authenticate sectors.

##### **Procedure:**
1. **Initiate the NFC Read:** Position your Flipper Zero near the card and activate the NFC “Read” function.
2. **Automated Key Testing:** The device cycles through a dictionary of known default keys and any previously captured keys.
3. **Validation:**  
   - For standard cards, a successful read reveals all 16 sectors with valid keys.
   - For extended variants (e.g., MIFARE Classic EV1), all corresponding keys must be recovered.

##### **Considerations:**  
- **Patience is Essential:** The attack can take several minutes; avoid interrupting the process.
- **Iterative Approach:** If only a subset of keys is recovered, proceed with more advanced techniques.

---

#### 2.2 Mfkey32 Attack

##### **Rationale:**  
When the dictionary attack is only partially successful, the Mfkey32 method captures cryptographic nonces during the reader’s handshake and processes them—often via a combination of statistical and brute-force methods—to deduce additional keys.

##### **Mechanics & Justification:**

- **Cryptographic Material Exposure:**  
  During the handshake, the reader issues a nonce (a random challenge) to the card. The card’s response, computed using CRYPTO1 and its secret key, may inadvertently leak exploitable intermediate data if the reader's implementation is weak.

- **Exploiting CRYPTO1 Weaknesses:**  
  The limited key space and predictable challenge-response behavior of CRYPTO1 mean that once nonces are captured, they can be leveraged to reconstruct the secret keys using advanced cryptanalysis.

- **Randomization Deficiencies:**  
  Poor nonce randomization or predictable patterns reduce the computational effort needed for key recovery.

- **Session Aggregation:**  
  In cases where the reader reuses certain values or fails to adequately protect intermediate data, cumulative nonces from multiple sessions can eventually reveal the complete key material.

##### **Procedure:**
1. **Reader Interaction:**  
   - On your Flipper Zero, navigate to **NFC → Detect Reader** (labeled “Extract MF Keys” in OFW 1.0.0).
   - Hold the device near the operational reader.
2. **Nonce Collection:**  
   - If the reader’s handshake is disrupted by weak UID checks or other flaws, it will emit nonces.
3. **Key Extraction:**  
   - **Computer-Based:** Connect via USB-C, access [Flipper Lab](https://lab.flipper.net/), and use the “GIVE ME THE KEYS” option or run `mfkey32v2` from GitHub.
   - **Mobile App:** Utilize the Flipper Mobile App (iOS/Android) over Bluetooth to execute the Mfkey32 attack.
   - **On-Device Execution:** Launch the MFKey app from the Flipper Zero main menu (if memory issues occur, a reboot may help).
4. **Reevaluation:** Re-scan the card with newly recovered keys to unlock additional sectors.

---

#### 2.3 Nested Attack

##### **Rationale:**  
A Nested attack takes advantage of partially authenticated sessions and previously obtained keys by leveraging residual nonces from earlier interactions to unlock remaining sectors.

##### **Procedure:**
1. **Install FlipperNested:** Download the FlipperNested app from the App Hub on the Flipper Mobile App.
2. **Follow the Guide:** Refer to detailed instructions on the [FlipperNested Wiki](https://github.com/AloneLiberty/FlipperNested/wiki/Usage-guide) for optimal positioning and data capture.
3. **Attack Variants:**
   - **Static Nested Attack:** Effective when nonces remain constant; can be run on-device or using FlipperNestedRecovery on a PC.
   - **Full/Hard Nested Attack:** Requires more data and computational power, best performed on a computer.
4. **Analysis:**  
   - If additional keys are found, reinitiate the scan.
   - For “Static Encrypted” cards, consider using specialized tools (e.g., a Proxmark3 running the `run fm11rf08s_recovery` script).

---

#### 2.4 KDF Attack

##### **Rationale:**  
Some card manufacturers employ specific key derivation functions (KDFs) that can be exploited via dedicated plugins, allowing for rapid key recovery when vulnerabilities exist.

##### **Procedure:**
1. **Download Plugins:** Visit the [Flipper KDF GitHub page](https://github.com/noproto/flipper_kdf/releases/latest) and download the latest `plugins.zip`.
2. **Deploy Plugins:** Extract the `.fal` files and copy them to the SD card directory at `/apps_data/nfc/plugins/`.
3. **Rescan the Card:** Use the NFC app on your Flipper Zero to re-scan the card; the plugins will attempt to derive any missing keys.
4. **Verification:**  
   - Success is achieved when all expected keys (e.g., 32/32 for a standard layout) are recovered.

> **Note:** If certain keys remain elusive, revert to the Mfkey32 method. Currently, there is no known process to crack the residual keys without further exploiting system vulnerabilities.

---

### Step 3: Emulation and Cloning

After successfully extracting all sector keys, you can proceed to either emulate the card using the Flipper Zero or clone its data onto a physical “magic” card.

#### Emulation

1. **Load the Card Profile:**  
   - Navigate to **NFC → Saved → [Your Card’s Name]** on the Flipper Zero.
2. **Initiate Emulation:**  
   - Select **Emulate** and hold the device near the target reader.
3. **Troubleshooting:**  
   - If the emulated card is rejected—often due to timing issues—upgrade your Flipper Zero firmware to version 0.94.0 or later, which includes an improved NFC stack. *(Note: Some tools like FlipperNested might not support firmware above 0.93.0 yet.)*

#### Cloning to a Physical Card

1. **Select a Magic Card:**  
   - **For a 4-byte UID:** Choose Gen1a or Gen2 Magic Cards (4-byte UID variant).  
   - **For a 7-byte UID:** Opt for Gen2 (7-byte UID variant) or Gen4/Ultimate Magic Cards.
2. **Clone the Data:**  
   - Follow the [official guide for writing to magic cards](https://docs.flipper.net/nfc/magic-cards) to transfer both the recovered keys and data sectors.
3. **Verification:**  
   - Test the cloned card on the original reader. Note that some advanced readers may implement countermeasures to detect cloned or “magic” cards.

---

## 4. Final Thoughts and Mitigation Strategies

Cloning MIFARE Classic 1K cards starkly exposes the vulnerabilities inherent in legacy access control systems. While the techniques described here mirror the current state-of-the-art in penetration testing, they also serve as a call to action for better security measures.

**Recommendations:**
- **Upgrade Your Systems:** Transition from legacy RFID technologies to modern, cryptographically secure solutions.
- **Enforce Strong Key Management:** Replace default keys and ensure that keys are diversified across sectors.
- **Implement Multi-Factor Authentication:** Combine card-based access with additional verification layers.
- **Enhance Physical and Network Security:** Utilize network segmentation and improve physical security controls to further protect access systems.
- **Continuous Security Auditing:** Regularly test and update system firmware and software to defend against evolving threats.

> **Ethical Reminder:**  
> This guide is provided for educational purposes and authorized security testing only. Unauthorized cloning or tampering with RFID systems is both illegal and unethical.


By integrating these insights and best practices, security professionals can significantly bolster the resilience of access control systems against modern threats.

## Additional reading and sources
- [Wikipedia](https://en.wikipedia.org/wiki/MIFARE)
- [Flipper Forums](https://forum.flipper.net/t/how-to-clone-mifare-classic-1k-4byte-uid-card/8903)
- [noproto Gist](https://gist.github.com/noproto/63f5dea3f77cae4393a4aa90fc8ef427#mifare-classic)
- [nfc-tools mfoc](https://gist.github.com/noproto/63f5dea3f77cae4393a4aa90fc8ef427#mifare-classic)
- [Gavin Johnson-Lynn Proxmark 3, Cloning a Mifare Classic 1K](https://www.gavinjl.me/proxmark-3-cloning-a-mifare-classic-1k/)
- [Flipper Docs MFKey32 Attack](https://docs.flipper.net/nfc/mfkey32)
- [Dark Flipper Unleashed Firmware](https://github.com/DarkFlippers/unleashed-firmware)
- [equipter - MFKey32v2](https://github.com/equipter/mfkey32v2)
- [Jasper van Woudenberg - Backdoor Key Found in MIFARE Classic Cards](https://www.keysight.com/blogs/en/tech/nwvs/2024/08/27/security-highlight-backdoor-key-found-in-mifare-classic-cards)
