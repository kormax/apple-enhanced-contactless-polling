# Broadcast frames with nfcpy library

## Terminology

Broadcast frame is an NFC frame that can be sent during the polling loop. It may or may not be responded to. It can be used in conjunction with a custom-developed NFC target (i.e. a bracelet/watch/phone). Any arbitrary piece of data may serve as a broadcast payload, for instance `48656c6c6f20776f726c64`. For data integrity, CRC is appended as defined in ISO14443A/B.  
Broadcast frame may be used:
1. As a unidirectional way of conveying information (reader -> device) very quickly, without needing to do full anticollision sequence and NDEF/proprietary data read;  
2. As a way of conveying other information in a non-standard, proprietary manner, in order to implement a communication flow without interrupting NFC operation for incompatible devices. A great example would be magic WUPC frames for Magic Mifare tags, that are used to activate hidden functionality, but are also used by some readers to prevent the use of magic tags;  
3. As a form of device authentication before the transaction start. 

## Overview

This example implements example support for NFC broadcast frames with the use of [nfcpy](https://nfcpy.readthedocs.io) Python library.

Current implementation uses the `BroadcastFrameContactlessFrontend` class that inherits from the `ContactlessFrontend`, with `sense` method overriden to add support for broadcast frames. Broadcast frame is passed into the method as an optional `broadcast` keyword argument.

Due to a way the library is structured, method overrides require copying over a lot of repeating boilerplate code. To help with understanding, new code blocks inside of overriden method are denoted by `Modified code BEGIN` and `Modified code END` comments.

A fully "correct" implementation would require to implement `sense_broadcast` method for each device driver, thus moving all support checks inside, but this was not done in order to keep the example code simple.

## Support

Current example supports following modules:
- PN532;

Support for other modules and chips is not implemented, but is possible to do in a similar fashion as provided.  
Modules like ACR122 are based on PN532, so implementation should be very simillar. It was not added due to a lack of a test device to verify functionality on.


## Usage 

### Prerequisites

1. A device with a functioning operating system;
2. Valid Python 3 installation;
3. Drivers for serial/usb adapters used to connect NFC module to the device;
4. A supported NFC module.

### Installation

<sub> Note that command examples were done on MacOS. It is not mandatory to use the commands exactly as presented. They are given solely to help understanding what needs to be done.</sub>

1. Copy the contents of this folder to your device, enter the folder;
2. (Optional) use the Python virtual environment:
    - Create the environment `python3 -m venv ./venv`;
    - Activate the environment `source ./venv/bin/activate`.
3. Install the required dependencies (only nfcpy) `python3 -m pip install nfcpy` or using the provided requirements file `python3 -m pip install -r requirements.py`;
4. Open the `main.py` file using any code editor:
    - Modify path to your device:
        - In MacOS, `ls /dev/tty.*` command returns `/dev/tty.usbserial-0001` for connected device, so the value of `path` variable should be `"usbserial-0001"`;
        - In Linux, done simillarly to MacOS;
        - For windows, COM port number can be viewed in device manager.
    - (Very rare cases) modify `interface` variable;
    - Set the `broadcast` variable value to HEX string of data that should be sent during the polling loop. CRC is added automatically, so you should omit it from input.

<sub> If you encounter any mistakes in the code or tutorial - feel free to notify me about them.</sub>  
<sub> For general python-related questions, consult Google/Bing/ChatGPT </sub>


### Running the code

To run the code, use `python3 main.py` command.

Example code doesn't do anything else apart from sending broadcast frames and printing out the information of found remote contactless targets. Change as you wish.


### Common issues

1. If you're running nfcpy on Linux, it may try to increase the baudrate of PN532 connection together with the baudrate of your serial device. Some poorly-made/cloned devices might have problems operating on speeds higher than 115200 baud.  
   Issue manifests as a timeout error, but only after the initialzation, when board version is printed to the console.  
   To fix that issue:
   1. Use another UART adapter;
   2. Go to library install location, `/site-packages/nfc/clf/pn532.py`.  
      At line 390, replace `change_baudrate = True` with `change_baudrate = False`.
2. If you're getting frame index/malformation errors, try increasing broadcast frame response timeout in file `broadcast_frame_contactless_frontend.py` at line 86.  
   Beware that timeouts larger than 0.3 seconds are not adviced for optimal performance. 
3. "Everything runs but no magic happens". Verify that you've changed broadcast frame to the value that you need. Beware that default value is just `Hello world` in encoded form.

## License notice

Example code uses and overrides parts of [nfcpy](https://nfcpy.readthedocs.io) library code.
Assume that code inside of this folder is licensed with the same license as the library at the moment of writing: `EUPL-1.1 license`.

## Notes

- If you manage to create a functioning implementation for other module types or have made improvements to the example code, feel free to create a pull request.