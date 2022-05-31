# Windows Blue Pill

An examination of braking security without breaking SecureBoot

This is based on observations in the wild, and hypothesises how Windows machines may be comprimised without breking the Kernel device driver and SecureBoot security model, giving an effective rootkit without the need for an exploit.

## tl;dr

Loop the EFI SecureBoot chain around from `bootx64.efi` to a different-signed `bootmgfw.efi` file, boot Windows into a Hyper-V instance and use Hyper-V local debugging to allow the host slice to lobotimize the guest OS, while passing through the graphics and HID to the guest.

## Windows Kernel Debugger and the Hyper-V

As with all blue pill designs, the best API / contract with the guest operating system is always to use OS kernel debug functionality.  This is the case becasue its highly stable between versions, well documented and can usually be passed off as a expected feature of the OS, making it uncommon that detections occur.

Windows 10 began installing and using Hyper-V in all cases when Virtualization Backed Security was introduced, meaning that a "vanilla" install of Windows provides the required virtualization components, and hardware manufacturers ship with Intel's VT/VTd extensions emnabled.

## Using Restore to Bypass Signing

One highlighed way to bypass the secure boot chain is to hybernate a machine that has already had its security state lowered.

## Windows Boot Components

The first component booted is the `bootx64.efi` binary, that acts as an "entry" into the Windows boot manager.  This is signed with the Microsoft PCA root, and is therefore trusted by the SecureBoot portion of the boot process.  This is the only component verified by the EFI secure boot chain, and if the SecureBoot `dbx` (revoked siguatures) database is not updated with prior versions, means an older copy with known vulnerabilities can easily be used in place of the common one.  In the authors opinion the certififcates need a monotonic incrementer for EFI to keep track of "latest" and deny all prior versions to be secure.  (being possible to disable via EFI policy of course as this would break older versions of Windows from booting after the increment).

The job of `bootx64.efi` is to hand off to the `bootmgr.efi` binary under the Microsoft hive.  

### `boot.stl`

This is an ASN.1 encoded set of data that spcifies the trust configuration (aka what certificates are trusted in a SecureBoot configuration)

### 

### Aside - Surface EFI

From a quick analysis, the Surface devices seem to have built and derived their configuration and UI around the exact Microsoft technologies such as the Boot Configuration Database (BCD) that are in use for later in the boot chain.  More data is needed about where one config ends and the other begins.


## Weaknesses in Windows Boot Architecture Identified

* SecureBoot can be rolled back to any prior verson
* Insufficant WIM signatures, where Windows PE should alert "This is a customized version of Windows from <SIGNER>" to indicate modified Windows images.  Hardware manufacturers should provide the attestation key in EFI runtime services to allow for the verification that the signer is the expected.  If it is other then Microsoft and the Manufacturer, big danger warnings should occur.
* Microsoft should fully document configurable settings in `BCD` that affect the boot security, as well as provide a baseline for assessment of a secure or insecure boot configuration
  * Example: `allowedinmemorysettings` is a value (in my case `0x15000075`) which doesn't specify what the intent of the setting is.
  * Full documentation of `ReAgent.xml`
  * `custom:21000026` is a BCD property that has no documentation
  * `custom:46000010` is a BCD property that has no documentation
  * The differences between `bootmgr.efi` and `bootmgfw.efi` should be documented
  * More data is required on if `partition=C:` could be ambigious by making another volume letter asignment prior to the intended disk
  * Debugger's `debugtype=Local` on install seems to either be a massive surface area bug or an indicator of comprimise.
    * Combined with `hypervisordebugtype`/`hypervisordebugport`/`hypervisorbaudrate`
  * System seems to indicate that there's both a `boot.sdi` as well as a `winre.wim` - clarifiactoin is needed about which is used in which context
  * `displaymessageoverride` seems abusiable as it can (and in my case does) have a value of "Recovery"
    * Similar: `
