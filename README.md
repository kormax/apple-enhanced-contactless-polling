# Apple Enhanced Contactless Polling (ECP)


<p float="left">
 <img src="./assets/PN532.ECP.DEMO.webp" alt="![ECP Access Home with PN532]" width=250px>
 <img src="./assets/FLIPPER.ECP.DEMO.webp" alt="![ECP Transit Clipper with Flipper Zero]" width=250px>
</p>


# Overview


Enhanced Contactless Polling/Protocol (ECP) is a proprietary extension to the ISO/IEC 14443 (A/B) standard developed by Apple.  

It defines a custom data frame that a reader transmits during the polling sequence, giving an end device contextual info about the reader field, allowing it to select an appropriate applet even before any communication starts.  

This extension:
- Helps to make sure that end device will only start communication with the reader if it has something useful to do with it, avoiding error beeps and card clashing;
- Increases privacy and security as it complicates brute force scanning for available passes on the device in a single tap.
- Allows automatic usage of non ISO7816-compliant passes:
  * DESFire in native mode and on card-level instead of app-level;
  * Passes without application id: Mifare Plus, Ultralight, Classic etc.
- Helps with conflict resolution when there are multiple passes with the same AID:
  * For instance, both Gymkit and ISO18013 use NDEF AID for BLE handover. ECP allows to differentiate between them in advance.
- May serve as a form of NFC DRM, requiring reader manufacturers to pay licensing fees in order to be able to use this feature and provide better experience for Apple users.


# Use cases


Express mode for most passes (apart from NFC-F and CATHAY) is implemented using ECP. That includes:
- Credit cards (For transit fallback);
- Transit cards;
- Access passes:
  - University;
  - Office badges;
  - Venue (Theme parks);
  - Apartment;
  - Hotel.
- Keys:
  - Car;
  - Home.

Other features use ECP as well:
- Value Added Services ([VAS](https://github.com/kormax/apple-vas)):  
  Allows reader to select the VAS applet and try to get pass in advance (although failing to do so), causing pass to appear on a screen for authentication or under a payment card if one is selected.  
  <img src="./assets/VASONLY.BEFORE.DEMO.webp" alt="![Image showing VAS]" width=200px>
  <img src="./assets/VASANDPAY.BEFORE.DEMO.webp" alt="![Image showing VAS]" width=200px>
  <img src="./assets/VASANDPAY.AFTER.DEMO.webp" alt="![Image showing VAS]" width=200px>
- GymKit:  
  Makes apple watch act as an NDEF tag for BLE handover in order to connect to supported gym equipment.  
  <img src="./assets/GYMKIT.DEMO.webp" alt="![Image showing GymKit setup]" width=200px>
- Identity:  
  Makes apple device act as an NDEF tag for BLE handover in order to connect to a ISO18013 verfier.  
  <p float="left">
   <img src="./assets/ISO18013.REQUEST.DEMO.webp" alt="![ISO18013 Request promt]" width=200px>
  </p>
- CarKey Setup:  
  Tells the device what car brand it is, causing a car key setup popup to appear on a screen.  
  <p float="left">
   <img src="./assets/CARKEY.SETUP.IOS16.DEMO.webp" alt="![Image showing IOS16 CarKey pairing prompt]" width=200px>
   <img src="./assets/CARKEY.SETUP.IOS17.DEMO.webp" alt="![Image showing IOS17 CarKey pairing prompt]" width=200px>
  </p>
- Field ignore:  
  Makes apple devices not react (by react meaning displaying a default payment card) to a field generated by other apple devices.
- AirDrop:  
  Replaces field ignore in IOS17 for background reading, used to negotiate an AirDrop session. NameDrop is a special case of AirDrop. Triggers a warp animation.  
  <p float="left">
   <img src="./assets/AIRDROP.DEMO.webp" alt="![AirDrop warp animation preview]" width=200px>
  </p>
- HomeKit:  
  Allows appliances with an NFC reader that lack card emulation mode to convey pairing info and bring up a pairing prompt when a user device is brought near to it. 
  

# Device support


Reader side:
* Can be implemented in software on most devices, provided that a low-level access to NFC hardware is available. In some cases it is required to reimplement parts of the protocol stack in software when doing so.  
HALs/Libraries for most popular chips contain separate confidential versions that include ECP support and are given to approved partners only, but homebrew solution is easy to implement.  
  Proof of concept was successfuly tested using PN532, PN5180, ST25R3916(B) chips;
* IOS has special reader APIs that make the device emit specific ECP frames:
  *  NFCVASReaderSession, PaymentCardReaderSession for VAS;
  *  MobileDocumentReaderSession for Identity;
  *  When using other derivatives of NFCReaderSession, device emits Ignore frame so that other apple devices don't react to it.
* Android does not have an API for ECP, although some android-based handheld reader manufacturers have implemented this feature in their software.  
  
Device side:
* Implemented using a customized CRS applet.
* Can be implemented on chips that allow reading raw frames in emulation mode even before selection.


# Decision logic


Upon entering a loop, device does not answer to the first polling frame it sees, instead opting to wait and see what other technologies does the field poll for, allowing it to make a fully informed decision on what applet to select later.

When device makes a decision, it is mostly, although not in all cases (excluding keys) signified by a card image appearing along with a spinner.

Even though ECP is sent during the polling loop, device does not answer to it. Instead it responds to a polling frame related to technology of the pass that the device had decided to use.


<img src="./assets/EM.DECISION.webp" alt="![Image showing express mode animation after decision]" width=250px>


When device enters the loop initially:
* In case of a full polling loop (A,B,F) it waits through one full iteration before making a decision on what applet to select:  
```
(ENTRY) -> A -> ECP_A -> B -> ECP_B -> F -> (DECISION) -> A -> (RESPONSE)
```

```
A -> ECP_A -> (ENTRY) -> B -> ECP_B -> F -> A -> ECP_A -> (DECISION) -> B -> (RESPONSE)
```


* In case of partial or wierdly-ordered polling loop, behavior is different. For example:

```
(ENTRY) -> A -> ECP_A -> A -> ECP_A -> (DECISION) -> A -> (RESPONSE)
```

```
(ENTRY) -> F -> B -> ECP_B -> A -> F -> B -> (DECISION) -> ECP_B -> A -> (RESPONSE)
```

```
(ENTRY) -> A -> ECP_B -> F -> A -> ECP_B -> (DECISION) -> F -> A -> (RESPONSE)
```

```
(ENTRY) -> F -> F -> F -> (DECISION) -> F -> (RESPONSE)
```

```
(ENTRY) -> A -> A -> A -> (DECISION)
```

```
(ENTRY) -> A -> ECP_A -> F -> A -> ECP_A -> F -> (DECISION) -> A -> (RESPONSE)
```

<sub>Characters A, B, and F were used in examples as a shorthand for full polling frame names: WUPA, WUPB, SENSF_REQ respectively. ECP frame has different values depending on a use case _A/B suffix refers to modulation used. </sub>

In conclusion, it seems that if reader is polling for:
* 1 technology, decision is made after third poll, response is given on the fourth;
* 2 technologies, decision is made after the second polling loop, while the response is given on the third.
* 3 technologies, decision is made after the first loop, response is given on the second.

Tests were conducted using very big intervals between polling frames. IRL if polling is faster device might respond after more frames than shown, presumably because of internal processing delay.  

Although not possible during normal operation, if a reader is polling for multiple cards using express mode that use different technology qualifiers for selection, following technology priority will be applied:
1. ECP 
2. NFC-F
3. CATHAY  
  
(BUG) If polling for both ECP and NFC-F, device will display NFC-F card in animation while actually selecting and emulating NFC-A/NFC-B applet. 

(NOTE) In IOS17 new AirDrop frame does not follow the beforementioned rules, as device reacts to it on first iteration in all cases.


# Structure

## Frame format

Each ECP frame consists of a header, version, payload and CRC:

```
     6A         XX          XX...     XX XX
  [Header]  [Version]  [Payload (n)]  [CRC]
```
- Header byte has a constant value of (HEX) 6A;
- Version number can be either 0x01 or 0x02;
- Payload: Version-dependant;
- CRC (Calculated via ISO14443A/B algorithm, according to the modulation used).

## Payload

For V1 payload consists only of a single TCI:  
  ```
    XX XX XX
     [TCI]  
  ```
- TCI is a 3 byte long identifier. More info below.
  

For V2 payload contains terminal configuration, terminal type, terminal subtype, and data:  
  ```
       XX       XX        XX         XX...
    [Config]  [Type]  [Subtype]  [Data (n)]
  ```
  - Configuration byte has a following binary format:  
    ```
          1        X        0 0      X X X X
      [Unknown]  [Auth]  [Unknown]  [Length ]  
    ```
    - Auth: 0b1 if authentication not required, 0b0 otherwise.  
      If auth is required pass will be presented on a screen for manual authentication when brought near to the field.
    - Length: defines a length of data.
  - Type contains terminal type:
    - 0x01: Transit;
    - 0x02: Access;
    - 0x03: Identity (Handoff);
    - 0x05: AirDrop.
  - Subtype depends on type. In most cases it has a value of 0x00;
  - Data. Its content and availability depend on terminal type and subtype. Detailed description below.

## Data

Data is a part of payload in V2, it contains TCIs and extra data:

```
  XX XX XX...        XX..
  [TCIs (n)]   [Extra data (n)]
``` 
- TCIs define an array of 3 byte long indentifiers. Standard allows for 0-n long TCI arrays to be conveyed depending on terminal type and subtype;
- Extra data contents depend on terminal type, subtype, and TCIs:
  * For access/key readers it may contain a 8 byte long unique reader group identifier, which allows to differentiate between them for passes of the same type;
  * For HomeKit it contains pairing information;
  * For NameDrop it carries a 6 byte long BLE MAC address;
  * For AirDrop it carries a 6 byte long zeroed out value.

## TCI

TCI, also referred to as Terminal Capabilities Identifier, is an arbitrary three-byte-long value that establishes reader relation to a particular pass type (Home key, Car key, Transit) or system feature (Ignore, GymKit, AirDrop, NameDrop).

The following restrictions apply to the use of TCI:
- Some TCIs are bound to a reader with particular type and subtype (which requires V2), while others trigger for all types (support V1). It is not known if this behavior is a bug or was intentional;
- TCIs intended for V1 cannot be used with V2.

TCI format is arbitrary, although several patterns related to grouping of similar functionality can be established:
- VAS: grouped with the last byte having a value of 0x00, 0x01, 0x02, 0x03 depending on mode;
- Access (Car/Home/University/Office/Venue): First byte is static, other two link to a particular pass provider;
- Transit: First byte is static, other two link to a particular transit agency (and their pass);
- CarKey: usually grouped by car manufacturer, consequent values signal readers on front/back doors,charging pad, etc. First byte is always 0x01. Can be seen in wallet configuration json hosted at [smp-device-content.apple.com](https://smp-device-content.apple.com/static/region/v2/config.json).


# Configuration examples

Note that CRC A/B, ECP Header, Configuration bytes are omitted from this table.
<sub> NA - not applicable; XX - any; ?? - unknown </sub>

| Name                          | Version | Type | Subtype | TCI      | Data                    | Description                                                                                                                                            |
| ----------------------------- | ------- | ---- | ------- | -------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| VAS or payment                | 01      | NA   | NA      | 00 00 00 | NA                      |                                                                                                                                                        |
| VAS and payment               | 01      | NA   | NA      | 00 00 01 | NA                      |                                                                                                                                                        |
| VAS only                      | 01      | NA   | NA      | 00 00 02 | NA                      |                                                                                                                                                        |
| Payment only                  | 01      | NA   | NA      | 00 00 03 | NA                      | Serves as anti-CATHAY                                                                                                                                  |
| Ignore                        | 01      | NA   | NA      | cf 00 00 | NA                      |                                                                                                                                                        |
| Transit                       | 02      | 01   | 00      | XX XX XX | XX XX XX XX XX          | TCI refers to a transit agency, Data is a mask of allowed EMV payment networks for fallback                                                            |
| Transit: Ventra               | 02      | 01   | 00      | 03 00 00 | ?? ?? ?? ?? ??          |                                                                                                                                                        |
| Transit: HOP Fastpass         | 02      | 01   | 00      | 03 04 00 | ?? ?? ?? ?? ??          |                                                                                                                                                        |
| Transit: WMATA                | 02      | 01   | 00      | 03 00 01 | ?? ?? ?? ?? ??          | Will select a Smart Trip card                                                                                                                          |
| Transit: TFL                  | 02      | 01   | 00      | 03 00 02 | 79 00 00 00 00          | Allows Amex, Visa, Mastercard, Maestro, VPay                                                                                                           |
| Transit: LA Tap               | 02      | 01   | 00      | 03 00 05 | ?? ?? ?? ?? ??          |                                                                                                                                                        |
| Transit: Clipper              | 02      | 01   | 00      | 03 00 07 | ?? ?? ?? ?? ??          |                                                                                                                                                        |
| Access                        | 02      | 02   | XX      | XX XX XX | XX XX XX XX XX XX XX XX | TCI refers to a pass provider, Data is reader group identifier                                                                                         |
| Access: Home Key              | 02      | 02   | 06      | 02 11 00 | XX XX XX XX XX XX XX XX | Having more than one key breaks usual ECP logic                                                                                                        |
| Access: Car Pairing           | 02      | 02   | 09      | XX XX XX |                         | TCI refers to a combination of car manufacturer + reader position                                                                                      |
| Access: Car Pairing: Mercedes | 02      | 02   | 09      | 01 02 01 |                         |                                                                              |
| Identity                      | 02      | 03   | 00      | NA/00    | NA/00                   | Only ECP frame found IRL that lacks a full TCI. Could this mean that TCI length is variable or it could be missing and the extra byte is data instead? |
| AirDrop                       | 02      | 05   | 00      | 01 00 00 | 00 00 00 00 00 00       | Sent only after device sees a NameDrop frame                                                                                                           |
| NameDrop                      | 02      | 05   | 00      | 01 00 01 | XX XX XX XX XX XX       | Data part contains a BLE MAC-address                                                                                                                   |


# Full frame examples

Examples contain full frames with CRC calculated for ISO14443-A;

- VAS or payment:  
  `6a01000000f6f1`  
  ```
       6a         01      000000   f6f1
    [Header]  [Version]   [TCI]   [CRC-A]
  ```

- Ignore  
  `6a01cf0000abb1`  
  ```
       6a         01      cf0000   abb1
    [Header]  [Version]   [TCI]   [CRC-A]
  ```

- NameDrop:  
  `6a02890500010001deadbeef6969a390`
  ```
       6a         02        89       05      00      010001  deadbeef6969  a390
    [Header]  [Version]  [Config]  [Type] [Subtype]  [TCI]     [Data]     [CRC-A]

    1000        1001
    [NA]  [Payload length]
  ```

- Access: Car Pairing: Mercedes:  
  `6a02c30209010201530b`
  ```
       6a         02        c3       02      09      010201   530b
    [Header]  [Version]  [Config]  [Type] [Subtype]  [TCI]  [CRC-A]
  ```

Note that for examples to work 8-bit byte setting should be set in case of NFC-A.


# Contributing

Best way to help is to provide more samples of ECP frames and TCIs.  
Especially interesting (missing) are the following:
- TCIs of transit agencies that use EMV only:
  - France:
    - Dreux;
    - Lyon;
    - Montargis;
    - Nevers;
    - Tarbes-Lourdes.
  - Estonia:
    - Tartu.
  - Finland:
    - Turku.
  - Sweeden:
    - Malmo.
- Access readers (There might be many unknown variations, so any samples would be welcome):
  - University;
  - Office;
  - Venues;
  - Hotels.
- HomeKit pairing;
- Identity (Real device).

<sub>Frames missing from the example table but not mentioned above were collected but not yet analyzed and publicized.</sub>

One way to get this information is via a sniffing functionality of a device like Proxmark (Easy or RDV2/4) connected to a Proxmark client inside of Termux running on an Android phone. 
A couple of tidbits encountered:
- First time using the app I've encountered an issue connecting to Proxmark3 directly as Termux did not connect a device, TCPUART app had to be installed to forward serial connection over the local network to be used in Proxmark client inside of Termux;
- Some Android phones won't power Proxmark properly through direct connection. Connecting via a USB-C to USB-A dongle can help to overcome this issue.

More info on installing and running Proxmark client on your Android device [here](https://github.com/RfidResearchGroup/proxmark3/blob/master/doc/termux_notes.md).

The command needed to collect traces is `hf 14a sniff`, after activating the command hold the Proxmark near a reader for a couple of seconds. In some cases it is needed to tap/touch the reader in order to wake it up as it might not poll to save energy.

After that, press a button on a device, traces will be downloaded and can be viewed with a `hf 14a list` command. You'll know which ones are the ones.  

Some other devices might also be able to sniff the frames, but due to a lack of personal experience I cannot recommend any.


# Notes

- This document is based on reverse-engineering efforts done without any access to original documentation. Consider all information provided here as an educated guess that is not officially cofirmed.
- If you find any mistakes/typos or have extra information to add, feel free to raise an issue or create a pull request.


# References

* Resources that helped with research:
  - Code analysis:
    - [IOS16 Runtime Headers](https://developer.limneos.net/?ios=16.3);
  - Apple resources:
    - [Apple Developer Documentation](https://developer.apple.com/documentation/);
    - [Apple Wallet configuration json](https://smp-device-content.apple.com/static/region/v2/config.json);
    - [Apple mention of ECP as Enhanced Contactless Protocol](https://developer.apple.com/videos/play/wwdc2020/10006/?time=1023);
  - Forums:
    - [NXP mention that ECP HALs or docs are only given to licensed partners](https://community.nxp.com/t5/NFC/Do-CLRC66302HN-and-CLRC66303HN-support-Apple-s-ECP-Enhanced/m-p/1445260#M9362);
    - [ST mention that ECP docs can be provided only after certification](https://community.st.com/t5/st25-nfc-rfid-tags-and-readers/st25r3917b-technical-support-apple-ecp-guide/td-p/81953);
  - Device operation manuals:
    - [HID mention of TCI for reader configuration](https://www3.hidglobal.com/sites/default/files/resource_files/plt-03683_b.7_-_hid_reader_manager_app_user_guide_ios.pdf) [(Archive)](https://web.archive.org/web/20230630195103/https://www3.hidglobal.com/sites/default/files/resource_files/plt-03683_b.7_-_hid_reader_manager_app_user_guide_ios.pdf);
  - Chip brochures (with ECP mentions):
    - [PN7150X](https://www.nxp.com/docs/en/brochure/PN7150X_LF.pdf) [(Archive)](https://web.archive.org/web/20210920054718/https://www.nxp.com/docs/en/brochure/PN7150X_LF.pdf);
    - [ST25](https://www.st.com/resource/en/product_presentation/st25_product_overview.pdf) [(Archive)](https://web.archive.org/web/20230109135439/https://www.st.com/content/ccc/resource/sales_and_marketing/presentation/product_presentation/group1/a9/5d/77/96/be/9a/48/7e/ST25_NFC_RFID_product_overview/files/ST25_product_overview.pdf/jcr:content/translations/en.ST25_product_overview.pdf).
    - [SM-4XXX](https://www.legic.com/fileadmin/user_upload/Flyer_Broschueren/SM-4200_4500_flyer_en.pdf).
  - Chip datasheets:
    - [PN532](https://www.nxp.com/docs/en/nxp/data-sheets/PN532_C1.pdf) [(Archive)](https://web.archive.org/web/20230401225452/https://www.nxp.com/docs/en/nxp/data-sheets/PN532_C1.pdf);
    - [PN5180](https://www.nxp.com/docs/en/data-sheet/PN5180A0XX-C1-C2.pdf) [(Archive)](https://web.archive.org/web/20221127182441/http://www.nxp.com/docs/en/data-sheet/PN5180A0XX-C1-C2.pdf);
    - [ST25R3916](https://www.st.com/resource/en/datasheet/st25r3916.pdf) [(Archive)](https://web.archive.org/web/20230124020718/https://www.st.com/resource/en/datasheet/st25r3916.pdf).
* Devices and software used for analysis:
  - Proxmark3 Easy - used to sniff ECP frames out. Proxmark3 RDV2/4 can also be used;
  - [Proxmark3 Iceman Fork](https://github.com/RfidResearchGroup/proxmark3) - firmware for Proxmark3;
  - [RFID Tools app](https://play.google.com/store/apps/details?id=com.rfidresearchgroup.rfidtools) - app that can used to control OFW Proxmark RDV4 from an Android device while in field;
  - [Termux](https://github.com/termux/termux-app) - can be used to run Iceman Fork Proxmark client in field;
  - [TCPUART transparent Bridge](https://play.google.com/store/apps/details?id=com.hardcodedjoy.tcpuart) - used to connect Proxmark to a client running in Termux;
  - PN532, PN5180, ST25R3916 - chips used to test homebrew ECP reader implementation.
