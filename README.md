# Apple Enchanced Contactless Polling (ECP)

---

![Video demonstrating ECP](/assets/ECP.DEMO.webp)


Enhanced Contactless Polling/Protocol (ECP) is a proprietary extension to the ISO14443 standard developed by Apple.   

It defines a custom data frame that a reader transmits during the polling sequence, giving an end device contextual info about the reader field, allowing it to select an appropriate applet even before any communication starts.  

This extension:
- Helps to make sure that end device will only start communication with the reader if it has something useful to do with it, avoiding error beeps and card clashing;
- Allows automatic usage of non ISO7816-compliant passes:
  * DESFire in native mode or on card-level instead of app-level;
  * (Theoretically) Passes without application id: Mifare Plus, Ultralight etc.
- Helps with conflict resolution when there are multiple passes with the same AID:
  * For instance, VENTRA and HOP use Mastercard private label with the same AID, ECP allows both passes to be active for express mode simultaniously.


Express mode for almost all** passes (apart from NFCF and REDACTED) is implemented using ECP. That includes:
- Credit cards (For transit fallback);
- Transit cards;
- Access passes:
  - University;
  - Office badges;
  - Venue (Theme parks).
- Keys (that are actually a subset of access passes):
  - Car;
  - Home;
  - Apartment;
  - Hotel.


Other features (Gymkit, CarKey setup, VAS, Ignore other iphone NFC field) use ECP as well.

More info will be added here soon©.
