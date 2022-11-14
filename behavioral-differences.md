# Behavioral differences between PC/SC native implementations

Tests made with a "Yubico YubiKey OTP+FIDO+CCID" (Vendor ID `1050`, Device ID `0407`) and a Gemalto IDBridge CT30 (Vendor ID `08e6`, Device ID `3437`). When not specified, it's implied that a YubiKey was used.

Behavioral differences can be attributed to both different PC/SC implementations and to different smartcard reader drivers.


## SmartCardReader.connect()


<table>
  <tr>
   <td colspan="2">SmartCardReader.connect
   </td>
   <td>MS Windows
   </td>
   <td>PCSClite macOS
   </td>
  </tr>
  <tr>
   <td colspan="4">if a transaction is ongoing:
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>shared mode
   </td>
   <td colspan="2">blocks
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>exclusive mode
   </td>
   <td colspan="2">fails with SCARD_E_SHARING_VIOLATION
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>direct mode
   </td>
   <td>fails with SCARD_E_SHARING_VIOLATION (behaves as exclusive)
   </td>
   <td>works (behaves as shared)
   </td>
  </tr>
  <tr>
   <td colspan="4">if another connection has exclusive mode:
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>shared mode
   </td>
   <td colspan="2">SCARD_E_SHARING_VIOLATION
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>exclusive mode
   </td>
   <td colspan="2">SCARD_E_SHARING_VIOLATION
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>direct mode
   </td>
   <td>SCARD_E_SHARING_VIOLATION
   </td>
   <td>works, but there's a protocol mismatch on transmission
   </td>
  </tr>
  <tr>
   <td colspan="4">direct mode, without concurrent connections:
   </td>
  </tr>
  <tr>
   <td colspan="2"><p style="text-align: right">
preferredProtocols: T0|T1</p>

   </td>
   <td>activeProtocol T1
   </td>
   <td>activeProtocol T0|T1 (ie, transmitting will fail)
   </td>
  </tr>
  <tr>
   <td colspan="2"><p style="text-align: right">
preferredProtocols undefined</p>

   </td>
   <td>activeProtocol T1
   </td>
   <td>error "invalid value given"
   </td>
  </tr>
  <tr>
   <td colspan="2"><p style="text-align: right">
preferredProtocols raw</p>

   </td>
   <td>SCARD_E_NOT_READY
   </td>
   <td>activeProtocol raw
   </td>
  </tr>
</table>



## SmartCardConnection.startTransaction()

On Microsoft Windows, winscard automatically resets the card if a transaction is kept idle (ie, without any commands being issued) for more than 5 seconds. PCSClite has no such behavior (an idle transaction can be kept indefinitely).


## SmartCardConnection.status()

Note that the status in native PC/SC APIs by flags that can be combined and have overlapping meaning (specific ⊂ powered ⊂ present ), unlike in the Web API where status is an enumeration.

This table shows the content of an SCardStatus call after a SCardConnect using the parameters specified in the leftmost column.


<table>
  <tr>
   <td>device
   </td>
   <td>connect() parameters: shared mode, preferred protocols
   </td>
   <td>MS Windows
   </td>
   <td>PCSClite macOS
   </td>
  </tr>
  <tr>
   <td>YubiKey
   </td>
   <td>shared, T0|T1
   </td>
   <td>present | powered, T1
   </td>
   <td>present | powered | specific, T1
   </td>
  </tr>
  <tr>
   <td>Gemalto (with card)
   </td>
   <td>shared, T0|T1
   </td>
   <td>present | powered, T1
   </td>
   <td>present | powered | specific, T1
   </td>
  </tr>
  <tr>
   <td>Gemalto (card removed after connection established)
   </td>
   <td>shared, T0|T1
   </td>
   <td colspan="2">SCARD_W_REMOVED_CARD
   </td>
  </tr>
  <tr>
   <td>Gemalto (card removed after connection established and then reinserted)
   </td>
   <td>shared, T0|T1
   </td>
   <td>SCARD_W_REMOVED_CARD
   </td>
   <td>SCARD_W_RESET_CARD
   </td>
  </tr>
  <tr>
   <td>Gemalto (reader removed after connection established)
   </td>
   <td>shared, T0|T1
   </td>
   <td>SCARD_E_SERVICE_STOPPED
   </td>
   <td>SCARD_E_READER_UNAVAILABLE
   </td>
  </tr>
  <tr>
   <td>YubiKey
   </td>
   <td>direct, undefined
   </td>
   <td>absent | powered, undefined
   </td>
   <td>Error: invalid value
   </td>
  </tr>
  <tr>
   <td>Gemalto (with card)
   </td>
   <td>direct, undefined
   </td>
   <td>present, undefined
   </td>
   <td>Error: invalid value
   </td>
  </tr>
  <tr>
   <td>Gemalto (without card)
   </td>
   <td>direct, undefined
   </td>
   <td>absent, undefined
   </td>
   <td>Error: invalid value
   </td>
  </tr>
  <tr>
   <td>Gemalto (card inserted after connection established)
   </td>
   <td>direct, undefined
   </td>
   <td>absent | present, undefined
   </td>
   <td>N.A.
   </td>
  </tr>
  <tr>
   <td>Gemalto (card inserted after connection established)
   </td>
   <td>direct, raw
   </td>
   <td>N.A.
   </td>
   <td>0x7fb7, raw
<p>
then SCARD_W_RESET_CARD
   </td>
  </tr>
  <tr>
   <td>YubiKey
   </td>
   <td>direct, raw
   </td>
   <td>SCARD_E_NOT_READY
   </td>
   <td>present | powered | specific, raw
   </td>
  </tr>
</table>


On Windows, when a reader device is removed after a connection is established not only the connection handle is invalidated but also the context handle (resource manager context) as well. Meaning that one has to establish a new resource manager context to continue operation, whereas in PCSClite the context remains unaffected by the reader removal.

