# Web Smart Card - Background

The [PS/SC Workgroup specification](https://pcscworkgroup.com/) defines what a compliant API for smart card access must look like. The existing compliant implementations are [winscard](https://docs.microsoft.com/en-us/windows/win32/api/winscard/) on Windows (which is the de facto canonical  implementation of PC/SC) and [PCSClite](https://pcsclite.apdu.fr/), a free-software implementation providing an API compatible with winscard.h. The [CCID project](https://ccid.apdu.fr/) provides PCSClite with free software smart card and card reader drivers.

PC/SC support in platforms is as follows:
*   Microsoft Windows: ships with winscard (out of the box).
*   macOS: ships with a forked version of PCSClite (out of the box).
*   Linux: PCSClite can be installed.
*   ChromeOS: Available via the [Smart Card Connector](https://chrome.google.com/webstore/detail/smart-card-connector/khpfeaanjngmcnplbdlpegiifgpfgdco?hl=en) extension, which bundles PCSClite and CCID compiled to PNaCl/WebAssembly.

Even though the existing PC/SC implementations share pretty much the same C API, meaning that a C application (when properly written) can be compiled against both winscard (on Windows) and PCSClite (on macOS and Linux) with little to no modifications, their actual behaviors differ significantly ([some examples](behavioral-differences.md)). Such differences cannot be ignored or be transparent to users of a Web API that is at the same abstraction level, providing the same concepts and features, as that C PC/SC API.
