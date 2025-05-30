# AMD

## AMD SecureBoot

AMD Secure Boot is designed to ensure that a system boots using only software trusted by the Original Equipment Manufacturer (OEM). It establishes a chain of trust starting from the hardware, preventing unauthorized code from being executed during the boot process. This chain of trust is anchored in the AMD Secure Processor (ASP), also called the Platform Security Processor (PSP), a dedicated security subsystem within AMD processors.

### Bootflow

The host firmware boot process typically consists of multiple phases. The initial code executed during boot is called the **Initial Boot Block (IBB)**. The IBB initializes critical hardware components, such as the Trusted Platform Module (TPM), to establish a secure environment. In UEFI systems, the boot process starts with the SEC and PEI phases ([UEFI PI Boot Flow](https://github.com/tianocore/tianocore.github.io/wiki/PI-Boot-Flow)). Similarly, in coreboot, the process begins with the [bootblock and verstage](https://doc.coreboot.org).

When AMD Secure Boot is enabled, the AMD Secure Processor (PSP) verifies the integrity of the IBB to ensure it has not been tampered with. This means AMD Secure Boot is executed **before** the initial host firmware code, the IBB. Once verified, the IBB extends the chain of trust using vendor-specific mechanisms, such as UEFI Secure Boot or coreboot's measured boot.

Below is a visual representation of the bootflow with AMD Secure Boot enabled (image source: [IOActive Labs](https://labs.ioactive.com/2024/02/exploring-amd-platform-secure-boot.html)):

![Bootflow on AMD System with Secure Boot enabled](./images/Arch.png)

### Platform Security Processor Structures

The PSP is a critical component of AMD’s security architecture. It acts as the root of trust for the entire system and is responsible for initializing the Secure Boot process. Sadly the PSP - or rather its firmware source code - is not open-source. However quite some development and reverse engineering happend around the PSP. The PSP includes:

1. **Immutable ROM Code**: This is the initial code executed by the PSP. It cannot be modified and contains the root of trust logic.
2. **PSP Firmware**: Loaded after the ROM code, the PSP firmware performs additional security checks and initialization tasks.

The PSP supports various interfaces in order to retrieve and/or send information from/to the PSP. The vast majority of information are going through:

1. **Memory-Mapped (MMIO) Registers**: Certain MMIO registers are exposed to provide access to PSP status, configurations, and control mechanisms.
    - **PSB Status Register (Offset 0x10998)**: This register provides information about the status of the Platform Secure Boot (PSB). A value of `0x00` indicates no errors, while a non-zero value indicates specific error codes.
2. **Mailbox-Interface**: The Mailbox Interface is the primary mechanism for communicating with the PSP. It allows the system firmware, operating system, or software tools to send commands and receive responses from the PSP.
3. **Model-Specific Registers (MSRs)**: MSRs provide a low-level mechanism to configure and query specific features related to the PSP. Interesting MSRs are e.g.:
    - `MSR_AMD64_SYSCFG`
    - `MSR_AMD64_SEV`
    - `SEV_STATUS`


### Key Hierarchy

The key hierarchy in AMD Secure Boot involves several layers, ensuring that each stage of the boot process is verified before proceeding:

1. **Root Key**
   - The highest-level key, embedded in the hardware, which is used to verify the initial PSP firmware.

2. **Firmware Keys**
   - *PSP Firmware Key*: Used to sign the PSP firmware, ensuring its authenticity and integrity.
   - *BIOS/UEFI Firmware Key*: The BIOS/UEFI firmware is signed using a private key corresponding to the PSP's public key. This key is verified during the Secure Boot process.

### Verification Process

The Secure Boot process involves multiple steps to verify the authenticity and integrity of the boot components:

1. Loading PSP ROM code.
2. Loading PSP firmware.
3. Verification of the IBB.
4. IBB extends the chain of trust.

## AMD MemoryGuard

AMD MemoryGuard provides full memory encryption capabilities to protect data stored in DRAM from physical attacks, such as cold boot attacks or direct memory access (DMA) attacks. This technology encrypts system memory transparently using a dedicated hardware encryption engine integrated within the processor. It operates at the memory controller level, ensuring that all data written to DRAM is encrypted and decrypted seamlessly without requiring changes to software.

### Technical Details

- Hardware Implementation:

    - AMD MemoryGuard uses a hardware-based AES encryption engine within the memory controller to perform real-time encryption and decryption.

    - The encryption keys are generated by the PSP and are stored in a secure location inaccessible to the operating system or user.

- System Management:

    - Memory encryption is managed through Secure Memory Encryption (SME), which can be enabled or disabled via BIOS/UEFI settings or system firmware.

    - Encrypted memory pages are marked in page table entries (PTEs) with a special memory encryption bit, ensuring proper management by the operating system.

- Performance Considerations:

    - MemoryGuard operates with minimal performance overhead, typically in the range of 1-3%, depending on the workload.

    - Optimizations in the hardware encryption engine ensure minimal latency in memory transactions.

- Security Features:

    - Protects against cold boot attacks by ensuring memory contents are encrypted at all times.

    - Ensures encryption keys are ephemeral and regenerated during system reboots.

### Memory Read/Write Flow with MemoryGuard

Below is an ASCII representation of the memory read/write flow with AMD MemoryGuard:

```
+---------------------+        +---------------------+
|                     |        |                     |
|      CPU Core       |        |    Memory Controller|
|                     |        |                     |
+---------+-----------+        +----------+----------+
          |                               |
          |  Plaintext Data               |
          |                               |
          v                               v
+---------+-----------+        +----------+----------+
|   Encryption Engine  | <----> |  DRAM (Encrypted)   |
|   (AES Encryption)   |        |                    |
+----------------------+        +---------------------+

```

- **Write Flow**: Data written by the CPU core to the memory controller is first encrypted using the AES encryption engine before being stored in DRAM.
- **Read Flow**: Data read from DRAM is decrypted by the encryption engine before being delivered back to the CPU core as plaintext.

For more information on AMD MemoryGuard, check out the [white paper from AMD](https://www.amd.com/content/dam/amd/en/documents/products/processors/ryzen/7000/amd-memory-guard-white-paper.pdf).


## AMD SEV

AMD Secure Encrypted Virtualization (SEV) enhances the security of virtual machines by encrypting the memory of each VM with a unique encryption key. SEV is built into the processor and managed through the hypervisor, providing isolation between virtual machines and the hypervisor itself. This technology helps prevent unauthorized access and tampering by restricting the ability of the hypervisor to access encrypted virtual machine memory, ensuring that sensitive data remains protected even in a compromised system.

### Technical Details

1. **Encryption Mechanism**:
   - Each VM is assigned a unique encryption key by the PSP.
   - All data in the VM’s memory is encrypted before being written to DRAM and decrypted when read back.
   - The hypervisor cannot access or manipulate encrypted VM memory.

2. **Key Management**:
   - Encryption keys are managed entirely by the PSP, ensuring they are not exposed to the operating system or hypervisor.
   - Keys are generated when the VM is initialized and are destroyed when the VM is terminated.

3. **Secure Memory Paging**:
   - Memory pages swapped out to disk are encrypted, ensuring no plaintext data is exposed during paging operations.

4. **Integration with Hypervisors**:
   - SEV is supported by hypervisors like KVM and VMware.
   - Commands to the PSP, such as key provisioning and management, are issued through hypervisor APIs.

### Advanced Features

- **SEV-ES (Encrypted State)**: Extends SEV by encrypting the CPU register state when transitioning between the guest VM and the hypervisor, ensuring the hypervisor cannot access sensitive CPU state information.

## AMD SEV-SNP

AMD SEV-SNP builds upon SEV technology by adding strong memory integrity protection capabilities. SEV-SNP extends the memory encryption provided by SEV to include measures that prevent malicious modification of memory, such as replay or remapping attacks. This is achieved by using a hardware-based approach to validate the integrity and proper allocation of memory pages when they are accessed. SEV-SNP ensures that the system remains secure even if the hypervisor is compromised, providing an additional layer of security for virtualized environments.

### Technical Details

1. **Memory Integrity Protection**:
   - SEV-SNP validates the integrity of memory pages accessed by the VM.
   - It ensures that memory pages are mapped correctly and are not being reused, preventing replay attacks.

2. **Nested Paging Mechanism**:
   - Uses a hardware-based nested page table (NPT) with integrity checks for secure memory allocation.
   - NPT entries include metadata, such as Guest Physical Address (GPA) hashes, to validate memory mappings.

3. **Measurement and Attestation**:
   - SEV-SNP includes a **measurement feature** that allows guest VMs to verify the integrity of their initial state.
   - Attestation reports generated by the PSP contain details such as firmware versions, memory configurations, and platform features.

4. **Firmware and TCB Management**:
   - SEV-SNP tracks the **Trusted Computing Base (TCB)** version and provides mechanisms for updating and verifying firmware.
   - Secure Firmware Updates are cryptographically signed and verified by the PSP before being applied.

---

# Preliminary Test Plan

## AMD Platform Secure Boot

1. **Get PSP MMIO Base Address**:
    - `0x13E102E0` for families 17h, model 30h/70h or family 19h, model 20h.
    - `0x13B102E0` for all other models.

2. **Read `FUSE_PLATFORM_SECURE_BOOT_EN` from `PSB_STATUS`**:
    - Address: PSP MMIO Base Address + `0x10994` ([reference](https://github.com/mkopec/psb_status)).
    - Bit 24: If `1`, PSB is fused.

3. **Other Bits to Check in `PSB_STATUS`**:
    - Bit 0-7: `Platform Vendor ID`
    - Bit 8-11: `Platform Model ID`
    - Bit 12-15: `BIOS Key Revision`
    - Bit 16-19: `Root Key Select`
    - Bit 25: `Anti-Rollback`
    - Bit 26: `Disable AMD Key`
    - Bit 27: `Disable Secure Debug`
    - Bit 28: `Customer Key Lock`

4. **Check PSB Status Register (`0x10998`)**:
    - Bit 0-7: `PSB Status` — `0` if no errors occurred, non-zero otherwise.

**Tests:**
- Verify PSB is enabled (`PSB_Status`).
- Confirm PSB has been fused (`Customer Key Lock` enabled).
- Verify `Disable AMD Key` prevents booting AMD-signed images.
- Ensure `Disable Secure Debug` is enabled.
- Validate `PSP Status` is non-zero.
- Check `Platform Vendor ID`, `Platform Model ID`, and `BIOS Key Revision` are non-zero.

## AMD MemoryGuard

1. **Check SME and SEV Support**:
    - Read CPUID `0x8000001f[eax]`.
    - Bit[0]: SME support.
    - Bit[1]: SEV support.

2. **Verify Memory Encryption**:
    - Read `MSR_AMD64_SYSCFG` (`0xc0010010`):
      - Bit[23]: `1` if SME is enabled.
    - Read `MSR_AMD64_SEV` (`0xc0010131`):
      - Bit[0]: `1` if SEV is active.

## AMD SEV

1. **Check SEV Features**:
    - Read MSR `SEV_STATUS` (`C001_0131`).
    - Validate enabled SEV features based on Table 15-34.

2. **Check SPEC_CTRL**:
    - Read MSR `SPEC_CTRL` for additional SEV-related controls.

## AMD SEV-SNP

1. **Read SEV-SNP Metadata**:
    - Read `MDATA` and `TCB_VERSION` for detailed firmware information.
    - Reference: [AMD EPYC Technical Specifications](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf).

---

# Conclusion

AMD’s Secure Boot, MemoryGuard, SEV, and SEV-SNP technologies collectively provide robust security for modern systems, addressing threats from boot-time attacks to virtualized environment vulnerabilities. The provided test plans offer a framework for verifying these features’ implementation and functionality, ensuring compliance and resilience against advanced threats.

