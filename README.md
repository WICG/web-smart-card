# Web Smart Card Explainer

## Introduction

The objective of this API is to enable [smart card](https://en.wikipedia.org/wiki/Smart_card) ([PC/SC](https://en.wikipedia.org/wiki/PC%2FSC)) applications to move to the Web platform. It gives them access to the PC/SC implementation (and card reader drivers) available in the host OS.

See [background.md](background.md) for background information.

There's also the [Unofficial draft of the Specification](https://wicg.github.io/web-smart-card/).

## Motivating Use Cases

### Identification and Authentication

While there are other APIs that provide the right level of abstraction and security properties for identity on the Web, such as WebAuthn, there are domain-specific functions which can't be captured by such higher-level APIs. Take German ID cards for example:  The [AusweisApp2](https://www.ausweisapp.bund.de/en/home) needs [native code](https://github.com/Governikus/AusweisApp2/tree/community/src/card/pcsc) to talk to the platform's PC/SC stack to access a USB smart card reader. It can issue control commands to the reader to establish and destroy [PACE](https://de.wikipedia.org/wiki/Password_Authenticated_Connection_Establishment) channels between the reader and the card (if the reader supports this feature). Websites from the German government then have to interact with the user's ID card via that native companion application.

### Remote Access Applications

A remote access (aka "remote desktop") web app letting the remote machine access the host's card reader as if it were directly connected to it. Enabling PC/SC applications on that remote machine to work without modification, unaware that the card reader is not local.

### Badge reading kiosks

A web-based kiosk could read even simple RFID badges via PC/SC and then display relevant information on a display. It's also not uncommon for such readers to need control commands to put them into the proper state for reading the particular type of card the application supports.

### Enable new and niche smart card use cases 

You can see smart cards as a platform to communicate with applications (also known as applets) running on small, resource-constricted, computers with tamper-proof memory. Thus, being an application-indepentent platform, there are in theory no restrictions to its possible use cases.

While a higher level API is preferable, safer, more convenient or even necessary (think WebAuthN for FIDO2 applets), there's also a variety of both standardized and proprietary applets for which there's not enough "general usefulness" to justify a specific, higher-level, Web API for it. Or it's simply a new applet for a new use case.

## Non-goals

* Provide an API that is at a higher level than PC/SC.
* Solve all inconsistencies between the different native PC/SC stacks.

## Transmitting a command to a smart card present in a reader

This shows all the steps necessary to transmit a command to a smart card. This includes the prerequisite steps of listing available readers and establishing a connection to the smart card present in a given reader.

```js
try {
  let context = await navigator.smartCard.establishContext();
  let readers = await context.listReaders();

  // Assume there's at least one reader present.
  let reader = readers[0];

  console.log("Connecting to the smart card inside " + reader);
  // A permission prompt might be displayed before the connection is established.
  let connectResult = await context.connect(reader, "shared",
      {preferredProtocols: ["t0", "t1"]});

  console.log(`Connected with protocol ${connectResult.activeProtocol}`);

  let connection = connectResult.connection;

  // Send an APDU (application protocol data unit) and get the card's response
  let command = new Uint8Array([0x00, 0xA4, 0x04, 0x00, 0x0A, 0xA0, 0x00, 0x00,
      0x00, 0x62, 0x03, 0x01, 0x0C, 0x06, 0x01]);

  let responseData = connection.transmit(command);
  console.log("Card responded with: " + responseData);

  await connection.disconnect();
} catch(ex) {
  console.warn("A Smart Card operation failed: " + ex);
}
```

## Transmissions in a transaction

If a connection is established in `"shared"` access mode, to ensure consecutive transmissions are not disrupted by concurrent use from other applications, they must be done inside a transaction.

```js
try {
  await connection.startTransaction(async () => {
    // A transaction has successfully started.
    let firstCommand = new Uint8Array([0x00, 0xA4, 0x04, 0x00, 0x0A, 0xA0, 0x00,
        0x00, 0x00, 0x62, 0x03, 0x01, 0x0C, 0x06, 0x01]);
    let firstResponse = await connection.transmit(firstCommand);
    …
    await connection.transmit(someOtherCommand);
    …
    await connection.transmit(finalCommand);

    // Reset the card when ending the transaction.
    return "reset";
  });
} catch (ex) {
  // Either the transaction failed to start, the callback
  // has thrown or the transaction failed to end.
}
```

Upon calling `startTransaction()`, a transaction will be started with the card related to this connection. When successful, the given callback will be called. All interactions with the card done inside this callback will then be part of this transaction. The transaction is ended once the `Promise` returned by the callback settles. If the settled `Promise` is fulfilled, the transaction is ended with the card disposition it was fulfilled with. If that value is `undefined` or invalid, a `"leave"` card disposition is assumed. A rejected `Promise` also causes the transaction to be ended with a `"leave"` card disposition.

## Reacting to reader availability

A common use case is reacting when a new card reader becomes available. On most platforms one can be notified of that using the special reader name `\\?PnP?\Notification`:

```js
try {
  let context = await navigator.smartCard.establishContext();

  let oldReaders = await context.listReaders();

  let allReaders = await context.getStatusChange(
      [{readerName: "\\\\?PnP?\\Notification", currentState: {}}],
      {timeout: 10000})
        .then(
          (statesOut) => {
            // A new reader was added.
            return context.listReaders();
          }
        );

  let newReaders = allReaders.filter(x => !oldReaders.includes(x));
  console.log("Readers added: " + newReaders);
} catch(ex) {
  console.warn("A Smart Card operation failed (or timed out): " + ex);
}
```

## Security and Privacy Considerations

Smart cards offer tamper-resistant storage for private keys and other personal information as well as isolation for security-critical computations. Such devices can be used in identification and authentication applications.

The security-sensitive nature of smart card applications and the fact that PC/SC is a rather low level API, which makes it powerful while at the same time hard (or impossible) to differentiate legitimate from malicious use, it is recommended that it is made available only to [Isolated Web Apps](https://github.com/reillyeon/isolated-web-apps/blob/main/README.md). Note that despite this recommendation, this API is described independently from it.

### Risks

The scenarios described here refer to API access by websites in general and can be read as a justification for the recommendation of having it available only to Isolated Web Apps.

#### Accessing sensitive information

A malicious website might want to read information from the user's ID card (assuming it's inserted in a reader), to reveal their identity and obtain other private information. Such a card normally requires a PIN to proceed with this operation. Thus besides the user having to explicitly give permission for this website to establish connections with smart cards, that website will also have to lure the user into providing their card's PIN. Alternatively that website could simply try out guessed PINs against the inserted card, but that card would then go into a blocked state and would no longer be usable.

There are cards however, or parts of a card, which are not PIN protected. But if it was decided not to protect them with a PIN one could assume that this information wasn't considered to be sensitive enough.

#### Permanently disabling a smart card

A malicious website either intentionally or by exhausting guess attempts could permanently disable a card after failing to verify its PIN consecutively.

#### Misleading user into authenticating against a malicious website

A malicious website could mislead the user into assuming they're using their smart card to authenticate against a legitimate service. This exposes the user to a man-in-the-middle attack, potentially granting the attacker access to the legitimate website the user thought they were accessing.
WebAuthN is not susceptible to this attack. But with PC/SC being a lower-level API, the browser has no way to know this is happening since it cannot tell that the challenges being issued to the card don't actually come from that website. Strictly speaking, all the browser knows is that [APDU](https://en.wikipedia.org/wiki/Smart_card_application_protocol_data_unit)s are being transmitted to the card.

#### Cross-origin communication

Some smart cards have the ability to store arbitrary data without requiring PIN entry, thus different origins could share information by reading and writing data into the same card. But for this to be possible, the user would have to explicitly grant permission to both origins to access smart cards.

#### Changing the state of a reader device

The `SmartCardConnection.control` and `SmartCardConnection.setAttribute` methods are able to change the state and configuration of a card reader.
Some readers have the ability of having their firmware updated via some vendor-specific control codes. An attacker could theoretically exploit this feature to upload a malicious firmware that intercepts or stores PINs contained in APDUs sent by legitimate PC/SC applications in order to later gain access to that card using those PINs.
Alternatively, an attacker could aim to simply render the card reader unusable, either temporarily by setting an invalid mode that requires a reboot, or permanently by uploading a nonfunctional firmware.
The user would have to explicitly grant permission for the malicious website to establish connections with card readers to make this possible.

### Fingerprinting

The names of the available smart card readers provide an [active fingerprinting](https://www.w3.org/TR/fingerprinting-guidance/#dfn-active-fingerprinting) surface. This is made available via the `navigator.smartCard.getReaders()` API call and does not require user consent.
The name of a reader is also dependent on its driver, which is platform-specific. Ie, the same device when plugged on a Windows machine may show a different name than when plugged on a macOS, as the corresponding PC/SC stacks also come with their own different reader drivers and the reader name comes from its driver.

The `SmartCardReader.getATR()` method extends that fingerprinting surface by allowing one to also know the model and, in some cases,  issuer of the smart card inserted in that reader. Eg: the SIM card of a particular phone carrier is inserted in a particular card reader model. Because of that, this method is protected by the `"smartcard"` permission check, thus requiring user consent before this information is made available.

Some devices such as YubiKeys show up as both a smart card and its own card reader (ie, a card reader that has always the same card inserted). In this situation the ATR doesn't provide additional entropy as there is a one-to-one mapping between the reader name its constant ATR.

#### Entropy

Smart card readers are traditionally used in corporate or institutional machines, whereas their use in private computers by the general public is rare. Thus the mere presence of a smart card reader can already give out the category of its user (professional vs. private/leisure use).

#### Persistence

It persists as long as the same set of readers and inserted cards remains unchanged.

#### Availability

Only isolated web apps may access this fingerprinting surface.

#### Scope

The surface is consistent across origins.

#### Mitigation

A possible user-level mitigation is to simply remove readers (or cards) when not in active use (assuming they are not built in), which removes that fingerprinting surface.

A browser might want to move the `"smartcard"` permission check to be done on the `SmartCardResourceManager` methods, which are all `Promise` based and hence suitable for permission prompts and rejections. A non-API-breaking change would be needed to this spec and some applications might need to be updated to consider or handle the new rejections types at this stage.

## Considered alternatives

### Letting the user choose which card reader a website has access to

Device APIs like WebUSB, Web Serial, Web Bluetooth and WebHID all implement a chooser model in which a website can only see and enumerate the devices that the user has explicitly allowed them to (via a device picker GUI).

In the case of the Web Smart Card API, the privacy and security sensitive interface is essentially `SmartCardConnection`, not `SmartCardReader`. Freely enumerating all `SmartCardReader` devices does open an active fingerprinting surface but nothing more than that. Therefore it was decided to have the permission prompting at the point a website tries to establish a connection with a card (and/or with the reader itself, as reader control commands and reader attributes can then be sent/read). This is the point where that powerful interface would be made available to the website. Ie, the [powerful feature](https://w3c.github.io/permissions/#dfn-powerful-feature) is `SmartCardConnection` and not the entire Web Smart Card API.

Having said that, the asynchronous design of the `getReaders()` and `watchForReaders()` methods does provide optionality for implementing a permission prompt before any reader information is provided to the page, but no implementation is currently pursuing this route.

### Access permissions based on card type

The entity whose access needs permission is ultimately the smart card, as it contains sensitive data, and not the smart card reader. Thus the User Agent would grant/prompt/deny access based on the type of card that is inserted in a reader. Eg: one could end up with a card permissions configuration like the following (built by prompting for permission at different times): "Allow Web App Foo to access identification cards from maker X but deny it access to SIM cards from provider Y".

But unlike USB devices, smart card type identification is very irregular. It's an array of up to 32 bytes (called ATR, standing for Answer to Reset) containing a varying combination of *optional* information about the card type, including data in proprietary format. Thus card type identification is essentially about [*fingerprinting* its ATR byte array](http://ludovic.rousseau.free.fr/softwares/pcsc-tools/smartcard_list.txt). This means that the User Agent cannot reliably display a human-readable string to the user when prompting for permission. All it can do is display something close to "Do you want to give permission for accessing the card type currently inserted on reader X?". Note that the ATR identifies the card type, not the individual card at hand (ie, it's not a unique identifier).

Another limitation of the ATR fingerprinting is that the mapping between cards and their ATR strings is not well defined. Ie, cards that for all effects and purposes are of the same type may reply with different ATRs due to the manufacturer or the issuer/provider deciding to include different information. This would lead to an inconsistent user experience: it would be hard for the user to predict (or reason about) whether he would be prompted for permission when inserting a different card. One cannot expect the user to be aware of the peculiarities of ATR arrays that ultimately led to the User Agent decision.

### Access permissions based on card readers

Unlike USB devices which have well defined vendor and product IDs, card readers cannot be reliably identified in PC/SC. Card readers are identified only by their names (even though they're actually USB devices as well). The name string is meant to be human readable and its only requirement is that it's unique among the currently attached readers.  For a given reader device, the value of its PC/SC name string depends both on the PC/SC implementation and on the PC/SC reader driver at hand. This is particularly problematic when there are multiple readers of the same model connected to the host as their name strings will be prefixed or suffixed with an index to avoid collisions. Thus it's possible that the same reader, on the same host, will get a different name string when reattached to the host.

It's possible to get the USB vendor and product IDs of a reader by sending a `GET_TLV_PROPERTIES` control command to that reader driver. But that doesn't solve our problem since a reader driver might not support this feature and even if it does, it requires establishing a direct connection to that reader and sending a control command through it. Those operations are exposed in the Web API and therefore it's up to the Web API user to request them (and handle its results and possible errors) and not the Web API implementation itself.

### SCardConnection.status as an attribute with a onstatuschange event handler

The `SmartCardConnection.status()` method of this Web API exposes the information provided by the `SCARDCOMM::Status` method in the PC/SC spec (or `SmartCardStatus` function in winscard.h and PCSClite). There's no event-based procedure to get this information. Thus if `SmartCardConnection.status` was a changeable attribute in this Web API it would mean that its implementation would necessarily have to do polling of the aforementioned PC/SC function.

### Abstracting away the concept of a card reader

Even though the purpose of this API is to enable websites to read smart cards, and thus the readers are merely enablers or channels for that, it's still useful to expose card readers in the API. One use case is that some readers need to have control commands sent to them to put them in a particular mode where they can read the types of cards that an application is interested in or supports. Ie, for configuring, initializing, or setting up a card reader.

## Acknowledgements
* Reilly Grant &lt;reillyg@google.com>
* Domenic Denicola &lt;domenic@google.com>
* Maksim Ivanov &lt;emaxx@google.com>
