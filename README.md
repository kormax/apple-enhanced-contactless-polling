# Apple Enchanced Contactless Polling (ECP)

---

![Video demonstrating ECP](/assets/ECP.DEMO.webp)


Enhanced Contactless Polling/Protocol is a proprietary extension to the ISO14443 standard developed by Apple.   

It defines a custom data frame that a reader transmits during the polling sequence, giving an end device contextual info about the reader field, allowing it to select an appropriate applet even before any communication starts.

This approach:
- Allows automatic usage of non ISO7816-compliant passes:
  * DESFire in native mode;
  * (Theoretically) MFP or MFU.
- Helps with conflict resolution when there are multiple passes with the same AID.  
  * For instance, VENTRA and HOP use Mastercard private label with the same AID, ECP allows both passes to be active for express mode simultaniously).


Express mode for almost all** passes (apart from NFCF and REDACTED) is implemented using ECP.  
Other features (Gymkit, CarKey setup, VAS, Ignore other iphone NFC field) use ECP as well.

Detailed info will be added here in coming units of time.
