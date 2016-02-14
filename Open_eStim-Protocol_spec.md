# Open eStim Protocol Specification

This protocol is used in communication over UART or over the air using a HopeRF RFM22b Mobule

Default buadrate for UART connections is 9600 8N1

## Packet format

             +---+---+----+----+-----+----------+-----+----------+------+----+---+----------+-------------+------------+
    Offset   | 0 | 1 |  2 |  3 |  4  |     5    |  6  |    7     |   8  |  9 | A |    B..   |  C+(Length) | D+(Length) |
             +---+---+----+----+-----+----------+-----+----------+------+----+---+----------+-------------+------------+
    Field    | Magic | Version | SRC | SRC Mod. | DST | DST Mod. | Type | Length | Payload  |         Checksum         |
             +-------+---------+-----+----------+-----+----------+------+-------------------+--------------------------+
    Size     |   2   |    2    |  1  |     1    |  1  |    1     |   1  |    2   | (Length) |             2            |
             +-------+---------+-----+----------+-----+----------+------+--------+----------+--------------------------+
    Data     | "oe"  | 00   00 | ... |    ...   | ... |   ...    |  ... |   ...  |    ...   |           CRC16          |
             +-------+---------+-----+----------+-----+----------+------+--------+----------+--------------------------+

Fields:

 * Magic
   2 Bytes, MUST BE `ASCII "oe" (0x6F, 0x65)`
 * Version
   2 Bytes, MUST BE `0.0` (0x00 0x00)` as of this spec version
   Endpoints using the same major version (1. byte) should be considered compatible, second byte marks minor changes
 * SRC
   1 Byte, packet source address (if sending device can't receive answer packets, set this to 0)
 * SRC Module
   1 Byte, packet source module address, for addon modules (the host is always 0)
 * DST
   1 Byte, packet destination address (all devices should listen to their assigend address and to 0x00=broadcast address unless configured otherwise)
 * DST Module
   1 Byte, packet destination module address, for addon modules (the host is always 0)
 * Type 
   1 Byte, packet type
 * Length
   2 Bytes, payload length not including any other fields (payload length != packet length)
   *Field may be omitted in special cases, see below*
 * Payload
   (Length) Bytes packet payload
   *Field may be omitted if Length is allso omitted, see below*
 * Checksum
   2 Bytes, CRC16 checksum of the packet including the complete header


### Packet Types

#### 0x01: ACK
Low level positive respone for the last received packet.

    Type: 01
    Length: omit
    Payload: omit

#### 0x02: NACK
Low level negative respone for the last received packet.

    Type: 02
    Length: omit
    Payload: omit

#### 0x03: PING
Ping

    Type: 03
    Length: 1
    Payload: Sequence Number

#### 0x04: PONG
Ping Reply

    Type: 04
    Length: 2
    Payload: 
                 +----------+------+
        Offset   |    0     |  1   |
                 +----------+------+
        Field    | Sequence | RSSI |
                 |  Number  |      |
                 +----------+------+
        Size     |    1     |   1  |
                 +----------+------+

        Sequence Number: 1 Byte, the seuqence number in the assoiated ping request
        RSSI: 1 Byte, the rssi of the received ping request 0=unknowen, 1..ff RSSI in 0.5 db resolution per bit
              Wired devices set this to 0x00

#### 0x05: Plaintext Payload
The Payload consits of a commands and a sequence of arugments with vairable length, limited by the size of the length field.
For a list of valid commands and arguments see below

#### 0x06: Encrypted Payload Key 1
Same as the plaintext payload, just that it is encrypted using AES-128 in CBC mode with the MASTER KEY  
For details about the Encryption see below

#### 0x07: Encrypted Payload Key 2
Same as the plaintext payload, just that it is encrypted using AES-128 in CBC mode with the TEMPORARY KEY  
For details about the Encryption see below

#### 0x08-0xBA: Reserved For Future Use

#### 0xC0-0xFF: Reserved For User Applications


## Commands
Each command is prefixed with a 2Byte Sequence Number and a 1 Byte command id, followd by a packed struct speciffic to the command.  
The command may be followed by a number of argument structs, the number of beeing defined in the command struct

             +--------+--------+------------+----------------+------------------+
    Offset   |    0   |    1   |     2      |      3...      | 4+(CmdStructLen) |
             +-----------------+------------+----------------+------------------+
    Field    | Sequence Number | Command ID | Command Struct | Command Args     |
             +-----------------+------------+----------------+------------------+
    Size     |        2        |      1     | (CmdStructLen) |   (CmdArgsLen)   |
             +-----------------+------------+----------------+------------------+
    Value    |    uint16++     |     ...    |       ...      |       ...        |
             +-----------------+------------+----------------+------------------+

    Sequence Number: Replay protection, 


### 0x01: Beacon

    struct structCmdBeacon {
    	uint8 numArgs; //Allways 0
    } typedef cmdBeacon;

### 0x02: Temporary Key Initialisation

    struct structInitTempKey {
    	uint8 numArgs;
    	char salt [16];     //used to generate key
    	uint16 keyTimeout; //in 10sec increments
    	char sig [32];     //HMAC SHA256 Signature

    } typedef initTempKey;

*TODO*

## Encryption
The Encryprion uses AES with a 128bit keysize in CBC mode.
The receiver whaits for a vailid Packet with an encypted payload to setup its IV.
The Transmitter is only allowed to change the IV when it received an ACK for the last packet.
The Transmitter has to remeber an IV for each receiver.

### Master Key
The Masterkey is a key with is loaded to the receiver via usb, it's also the secet for the temporary key generation function  
The Masterkey may be used to controll all aspects of the device and to activate a new temporary key.

### Temporary Key
The Temporary is a key with is activated via the initTempKey command.  
The Temporary may be used to controll all or a limited subset of the available commands of the device.  
The Temporary Key has an timeout assoiated to it, after the set timeout expires its invalideted.  
If the outputs are still active when this happens, they a shut off in an controlled manner (ie. ramp down over 10sec)

*TODO*

## Attribution
Parts inspired by the protocol of nSST (https://github.com/nonchip/nSST)