#   NXP i.MX8M Plus: Secure and Encrypted Boot


##  Reference Manual

-   NXP, AN4581, i.MX Secure Boot on HABv4
-   NXP, AN12056, Encrypted Boot on HABv4
-   NXP, AN12714, i.MX Encrypted Storage using CAAM
-   NXP, Code-Signing Tool User Guide
-   NXP, U-Boot Source Code Docs:<br>
    url: https://source.codeaurora.org/external/imx/uboot-imx<br>
    branch: lf-5.10.35-2.0.0<br>
    path: doc/imx/habv4


##  General Info

-   Just follow U-Boot docs, they work. CST 3.1.0 and 3.3.1 shall both okay.
    -   introduction_habv4.txt
    -   mx8m_secure_boot.txt
    -   mx8m_encrypted_boot.txt

-   The images seems been Authenticated and Decrypted by BootROM.

        BootROM -> U-Boot SPL + DDR firmware -> BootROM -> ATF, TEE, U-Boot FIT

-   If uBoot uses hab_auth_img to authenticate images (secure boot), e.g. Linux Kernel, scripts in mx8m_secure_boot.txt work.

-   If uBoot uses hab_auth_img to authenticate and decrypt images (secure and encrypted boot), patches for ATF and TEE are required. They shall be included in the newer released code base with `encrypted kernel support` commit log.

    If the patches are not applied, U-Boot HAB call reports error and hangs.


##  Image layout for U-Boot Authentication and Encryption

Typical image layout provided by mx8m_secure_boot.txt is:
```
        ------- +-----------------------------+ <-- *load_address
            ^   |                             |
            |   |                             |
     Signed |   |            Image            |
      Data  |   |                             |
            |   |                             |
            |   +-----------------------------+
            |   |    Padding to Image size    |
            |   |          in header          |
            |   +-----------------------------+ <-- *ivt
            v   |     Image Vector Table      |
        ------- +-----------------------------+ <-- *csf
                |                             |
                | Command Sequence File (CSF) |
                |                             |
                +-----------------------------+
                |     Padding (optional)      |
                +-----------------------------+
```
In U-Boot, hab_auth_img requires `<img_addr> <img_size> <ivt_offset>` parameters.

To set `<ivt_offset>` to 0, the image layout can be set to:
```
        ------- +-----------------------------+ <-- *load, *ivt
        Signed  |  Image Vector Table (IVT)   |
        ------- +-----------------------------+ <-- *csf
                |                             |
                | Command Sequence File (CSF) |
                |                             |
                +-----------------------------+
                |             ...             |
                +-----------------------------+
                |          DEK Blob           |
        ------- +-----------------------------+ <-- *data
            ^   |                             |
            |   |                             |
        Singed  |                             |
          and   |            Image            |
     Encrypted  |                             |
            |   +-----------------------------+
            v   |       Image Padding         |
        ------- +-----------------------------+
```

In this new image layout,
-   Image is load to predefined address. *load, *ivt, *csf, *data are known.

    IVT could be pre-generated as binary blob.

    CSF + DEK could be pre-allocated and *data aligns to 0x1000 boundary.

-   So, hab_auth_img parameters, <img_addr> is known and <ivt_offset> is set to 0.

    In U-Boot, <img_size> is known if loading from filesystem, or, could be get through uImage.

-   IVT Blob, DEK Blob in CSF file shall be set as absolute address / offset.

Futhermore, U-Boot hab_auth_img command supports uImage or FIT image, but its usage and required image layout need detailed code review.


##  Code-Signing Tool (CST)

-   CSF file for CST is a script file. It will be converted to binary blob by CST and runs by CAAM from beginning to end.

    CST 3.3.1 provides hab_csf_parser, which can decode CSF binary blob for user investigation.

-   `Blocks` in `Authenticate Data` or `Decrypt Data` can cover multiple sections.

    For Secure Boot, IVT header and Payload shall all be covered. But they are not required to be contiguously layed out.

-   `Authenticate Data` section:
    -   CST reads data referenced by the `Blocks` line.
    -   Result is written out to CST -o output file.
    -   Source file is *not changed*.

-   `Decrypt Data` section:
    -   CST reads data referenced by the `Blocks` line.
    -   Encrypted data is written back to its original position.
    -   Source file is *modified*.
    -   Each data section shall be 16 Bytes aligned.

-   `--dek --skip` could be applied for reusing generated dek.bin and dek_blob.bin.

    At default, CST always generate a new dek.bin. And a new dek_blob.bin shall be obtained from target processor for decryption.

##  CST, 3.1.0

-   Needs re-compile if encrypted boot is used.

##  CST, 3.3.1

-   Seems fully open source, cross-compile to arm/arm64 possible.

-   Encrypted boot is already compiled in.

-   CSF file format issue:
    ```
    # supported by both 3.1.0 & 3.3.1
    Blocks = xxx yyy zzz "flash.bin", \
             xxx yyy zzz "flash.bin"

    # supported by 3.1.0, but not 3.3.1
    Blocks = xxx yyy zzz "flash.bin", xxx yyy zzz "flash.bin"
    ```
