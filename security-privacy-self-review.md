## Responses to the [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for the [Web Smart Card API](https://github.com/WICG/web-smart-card)


1. **What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

    This API exposes the PC/SC interface, including the list of available smart card readers and potentially all the possible information that one can get via controlling them. This is for the purpose of providing smart card functionality in isolated contexts. 

2. **Do features in your specification expose the minimum amount of information necessary to enable their intended uses?**

    Yes. The specification provides no more information than is needed for generic use of the PC/SC-compliant devices.

3. **How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**

    This specification deals with generic information (byte arrays) without interpretting or storing it. That being said, considering the use cases of the smart cards, it is certain that it will be used to deal with highly sensitive PII. 

4. **How do the features in your specification deal with sensitive information?**

    Same as in #3.

5. **Does data exposed by your specification carry related but distinct information that may not be obvious to users?**

    No. That being said, it would be a reasonable assumption that an average user won't understand the full scope of the data exposed by this API (due to lack of the domain knowledge).

6. **Do the features in your specification introduce new state for an origin that persists across browsing sessions?**

    Yes. If a user grants persistent permission to an origin, that grant is a state that persists across sessions. This allows the origin to connect to the specified smart card reader in the future without prompting the user again.

7. **Do the features in your specification expose information about the underlying platform to origins?**

    Possibly. This API is essentially a proxy to system native PC/SC, which might handle things differently on different systems. An example of this are reader names, which [might be computed differently on different platforms](https://blog.apdu.fr/posts/2010/05/what-is-in-pcsc-reader-name/).

8. **Does this specification allow an origin to send data to the underlying platform?**

    Yes. The main point of this feature is sending data to (and receiving it from) the platform's native PC/SC component.

9. **Do features in this specification enable access to device sensors?**

    Only if the sensors are compliant with PC/SC standard (show up to system as smart card readers)—which is not likely.

10. **Do features in this specification enable new script execution/loading mechanisms?**

    No.

11. **Do features in this specification allow an origin to access other devices?**

    Yes. It allows an origin to communicate with PC/SC-compliant smart card readers.

12. **Do features in this specification allow an origin some measure of control over a user agent’s native UI?**

    No.

13. **What temporary identifiers do the features in this specification create or expose to the web?**

    None.

14. **How does this specification distinguish between behavior in first-party and third-party contexts?**

    By default, permissions policy prevents use from third-party contexts.

15. **How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**

    It does not. This API is intended to be used in Isolated Web Apps (which do not support either mode).

16. **Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**

    Yes. The specification now includes a comprehensive, normative 'Security and Privacy Considerations' section. It details threats such as fingerprinting, device tampering, and authentication spoofing, and defines normative requirements for user agents to mitigate them, centered around explicit user consent.

17. **Do features in your specification enable origins to downgrade default security protections?**

    Yes. It is technically possible for one origin to store some data on a smart card for other origins to read it—or even share access to one smart card reader with them. However that is hardware-dependent and unavoidable.

18. **How does your feature handle non-"fully active" documents?**

    All the existing connections and contexts are natively disposed of. It is an important consideration for user agents to ensure that this termination happens reliably when a document is no longer fully active to prevent background applications from holding connections to sensitive hardware without the user's immediate awareness.

19. **Does your spec define when and how new kinds of errors should be raised?**

    Yes. Most of the possible errors are a direct translation of the error codes provided by the native PC/SC.

20. **Does your feature allow sites to learn about the user’s use of assistive technology?**

    No.

21. **What should this questionnaire have asked?**

    N/A.

