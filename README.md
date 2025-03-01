## Overview
Petyaware is a complete rewrite of the original Red Petya ransomware, designed to function on both MBR (Master Boot Record) and GPT (GUID Partition Table) disks. To operate on GPT disks, the system's UEFI firmware must support Legacy Boot mode.

## MBR Disk Infection Process
1. Detects if the target disk uses MBR.
2. Reads the original MBR from sector 0, encrypts it using XOR 0x37, and writes the encrypted MBR to sector 56.
3. Encrypts sectors 1-33 using XOR 0x37.
4. Generates configuration data for the Red Petya kernel, including:
   - Salsa10 encryption key
   - 8-byte random nonce
   - Personal decryption code (Salsa10 key encrypted with secp192k1 public key cryptography and Base58 encoded).
5. Writes verification data by filling sector 55 with 0x37.
6. Constructs the Red Petya bootloader:
   - Copies the disk ID and partition table from the original MBR (bytes 440-510) into its bootloader.
   - Writes the new bootloader to sector 0 and the 16-bit kernel to sectors 34-50.
7. Triggers a system crash using the undocumented `NtRaiseHardError` API with error code 0xc0000350.

## GPT Disk Infection Process
1. Identifies GPT disk structure and locates the Backup GPT Header by reading the Primary GPT Header at sector 1.
2. Encrypts the Backup GPT Header (last sector - 33 sectors) using XOR 0x37.
3. Encrypts the Primary GPT Header (sector 1-33) using XOR 0x37.
4. Modifies the disk ID to `7777` and constructs a partition table entry that represents the entire drive, facilitating BIOS-based CSM booting.
5. Executes the same actions performed on MBR disks.

## Personal Decryption Code Generation
1. Generates a 16-byte Salsa10 key using `CryptGenRandom` from a Base54 character set.
2. Generates an ECDSA key pair (victim private key and victim public key) on the secp192k1 curve.
3. Computes a shared secret using ECDH: `shared_secret = ECDH(victimpriv, Januspub)`.
4. Derives an AES-256 key: `AESKEY = SHA512(shared_secret)`.
5. XORs the Salsa10 key with the victim's public key and encrypts the result using AES-256-ECB with `AESKEY`.
6. Creates a buffer containing the victim's public key and encrypted Salsa10 key.
7. Encodes the buffer using Base58.
8. Computes a SHA-256 hash of the Base58-encoded data.
9. Generates a 90-byte final personal decryption code containing:
   - Base58-encoded victim public key and encrypted Salsa10 key.
   - Two verification bytes (`check1` and `check2`) derived from the SHA-256 hash.
10. Stores the final encoded personal decryption code in sector 54 at offset `0xA9`.

## Master File Table (MFT) Encryption Process
1. After a system crash and reboot, Petyaware's bootloader loads into memory and transfers control to its kernel.
2. The kernel reads sector 54 and verifies if MFT encryption is pending.
3. If unencrypted, it marks the sector as `MFT Encrypted` (0x01), stores the Salsa10 key temporarily, and clears its storage in sector 54.
4. Encrypts sector 55 using Salsa10 with the stored key and nonce.
5. Identifies NTFS partitions and determines MFT sector ranges.
6. Encrypts MFT clusters using Salsa10 in 8-sector batches, updating sector 57 with the encryption progress.
7. Upon completion, the system is forced to reboot using `INT 19h`.
8. On subsequent boot, Petyaware displays a blinking skull and presents ransom instructions, including onion URLs and the personal decryption code.

## Master File Table (MFT) Decryption Process
1. The MFT remains encrypted with Salsa10, and the encryption key is erased from sector 54.
2. The victim is prompted to enter a 16-byte decryption key.
3. The input is validated against the Base54 character set and expanded into a 32-byte key.
4. The nonce is retrieved from sector 54.
5. The provided key is used to decrypt sector 55.
6. If successful (sector 55 contains only `0x37` values), MFT decryption proceeds; otherwise, the key is rejected.
7. The decrypted 32-byte key is stored in sector 54 (flagged as `MFT Decrypted` - 0x02), and decryption begins.
8. The original MBR is restored by decrypting sector 56 using XOR 0x37 and writing it to sector 0.
9. If the disk ID is `0x37373737`, additional decryption processes restore the Backup GPT Header.

## Disclaimer
This software is intended for educational and cybersecurity research purposes only. Unauthorized use, deployment, or distribution of ransomware is illegal and punishable under various cybersecurity laws. The author does not endorse or encourage malicious use of this software and/or information.
