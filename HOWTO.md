# How to use the Web Smart Card API

The Web Smart Card API has been shipped in Chrome on ChromeOS as of M143.

1) Provision a ChromeOS device and ensure it's updated to Chrome 143 or above.
2) Install the [Smart Card Connector] extension (this is the PC/SC provider on ChromeOS).
3) The API should be available for Isolated Web Apps now. You may wish to try out the [provided example][Smart Card Demo app].

A rudimentary flow is similar to the native PC/SC libraries. Examples of code can be found in the explainer—for example [here][transmitting to a reader] is a section on transmitting a command to a smart card reader—and in the aforementioned [demo application][Smart Card Demo app].

[Smart Card Demo app]: https://github.com/GoogleChromeLabs/web-smartcard-demo
[Smart Card Connector]: https://chromewebstore.google.com/detail/smart-card-connector/khpfeaanjngmcnplbdlpegiifgpfgdco
[transmitting to a reader]: https://github.com/WICG/web-smart-card?tab=readme-ov-file#transmitting-a-command-to-a-smart-card-present-in-a-reader
