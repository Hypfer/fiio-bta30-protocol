
## Motivation

The FiiO BTA30 is a Hi-Fi-grade DAC and Bluetooth Transceiver, featuring USB-C for Data and Power, RCA analog output and
also SPDIF in and out via both TOSLINK and Coax. According to the marketing material, a Qualcomm CSR8675 Bluetooth Chip is used.

While it does work out of the box, there are some additional tunables available via the official "FiiO Control" Android App.
These include things such as the power-on-behaviour, whether the output level should be locked or which codecs should be used.

The app itself works fine, however it is unclear how long it will stay that way given that it is an Android app.
Thus, here's some documentation on the protocol.

This may also be adaptable to other similar FiiO devices, however I only have a BTA 30.

## Protocol

The FiiO BTA 30 uses Bluetooth Low Energy (BLE) for all its configuration needs. As with most BLE appliances,
the implementation could be described as _strange_.

It seems that all FiiO-specific functionality is just piggybacked on top of the generic `Qualcomm CSR102x OTAU` functionality,
which is probably some sample code provided by Qualcomm and it was just easier that way? Who knows.

Said OTAU characteristic is documented by Qualcomm in a PDF named `csr102x_otau_overview.pdf`, which I won't mirror here
due to copyright concerns. At the time of writing, it is available for download off the official Qualcomm servers:
[https://developer.qualcomm.com/qfile/34081/csr102x_otau_overview.pdf](https://developer.qualcomm.com/qfile/34081/csr102x_otau_overview.pdf)

As the name suggest, there also seems to be some over-the-air-firmware-update functionality available, however it seems like that doesn't apply to the BTA30 (yet?)

Firware updates would likely be stored somewhere below `http://fiio-bluetooth.oss-cn-beijing.aliyuncs.com/`.
There currently (2022-05-19) is a subfolder named BTA30, however it doesn't seem to contain a new firmware image.

### Characteristics

We have one service with UUID `00001100-d102-11e1-9b23-00025b00a5a5` named `CSR GAIA` according to the Qualcomm docs.

This service features three characteristics:

1. `00001101-d102-11e1-9b23-00025b00a5a5` (`CSR GAIA Command Endpoint`)
    Commands will be written to this characteristic

2. `00001102-d102-11e1-9b23-00025b00a5a5` (`CSR GAIA Response Endpoint`)
    Replies to commands will be provided via this characteristic as notifications.
    It needs to be subscribed first before any commands are sent

3. `00001103-d102-11e1-9b23-00025b00a5a5` (`CSR GAIA Data Endpoint`)
    No idea what this does


### Commands

#### Requests

Command requests are written to the `CSR GAIA Command Endpoint` characteristic.

These consist of at least four bytes but may be longer if there is a payload.

The first 20 bits are a magic while the next 12 bits contain the command ID.
After that, there may be 0-n bytes of payload.

##### Examples

Let's look at some examples to make this easier.

###### No Payload

This is an example without any payload

`00 0a 04 3d`

- First, there's the magic: `0x000a0`
- Then, there's the command: `0x43d`

That's it.
Command `0x43d` is the `GET_LED_OFF` command.
On success, you will get the following response: `00 0a 84 3d 00 01`.
If your LED_OFF setting is false, then the 0x01 at the end will of course be `0x00`.

###### Payload

Here's an example that features a payload:

`00 0a 04 0b 01`

- First, there's the magic: `0x000a0`
- Then, there's the command: `0x40b`
- Finally, there's the payload: `0x01`

Command `0x41c` is the `Set Boot Mode` command and `0x01` means true.
On success, you will get the following response: `00 0a 84 1c 00 01`.

#### Replies

Command replies are provided via notifications of the `CSR GAIA Response Endpoint` characteristic.

These consist of at least five bytes but may be longer if there is a payload.
The first 20 bits are a magic while the next 12 bits contain the command ID.
This is followed by a single null byte. After that, there may be 0-n bytes of payload.

##### Examples

Let's look at some examples to make this easier.

###### No Payload

This is an example without any payload:

`00 0a 84 25 00`

- First, there's the magic: `0x000a8`
- Then, there's the command: `0x425`
- Finally, there's the null byte `0x00`

Command `0x425` is the `Power Off` command, which just turns of the device.
Hence, there's no need for a payload. The reply confirms the successful receive of the command.

###### Payload

Here's an example that features a payload:

`00 0a 84 1c 00 01`

- First, there's the magic: `0x000a8`
- Then, there's the command: `0x41c`
- Then, there's the null byte `0x00`
- Finally, there's the payload: `0x01`

Command `0x41c` is the `Boot Mode` command and `0x01` means true.
Thus, this message means that the BTA 30 will turn on as soon as it receives power.
If this was `0x00` you'd still have to manually press the power button.

### List of Commands

Here are all available commands. Some of them may be read-only. Others can be written

#### Global Commands

##### Version

**Description:**<br/>
The Device Version + Type

**GET Command**:<br/>
0x418

**Response**:<br/>
3 Byte

1. Major Version
2. Minor Version
3. Device Type (Always `D4` for the BTA30)


##### Current Connection Codec

**Description:**<br/>
The Codec used by the current connection

**Command**:<br/>
0x416

**Payload**:<br/>
One Byte

Decimal:
- 0 = A2DP OFF
- 1 = SBC
- 3 = AAC
- 5 = APTX
- 7 = APTX HD
- 10 = LDAC




##### Boot Mode

**Description:**<br/>
Whether the Device should turn on as soon as it receives power

**GET Command**:<br/>
0x41C

**SET Command**:<br/>
0x40B

**Payload**:<br/>
Either `0x00` or `0x01`

##### LED Mode

**Description:**<br/>
Whether the Device LEDs should be on or off

**GET Command**:<br/>
0x43D

**SET Command**:<br/>
0x43E

**Payload**:<br/>
Either `0x00` or `0x01`

##### LED Pattern

**Description:**<br/>
Whether the LED flashes red-green or red-blue during connection setup

**GET Command**:<br/>
0x44A

**SET Command**:<br/>
0x44B

**Payload**:<br/>
Either `0x00` for red-green or `0x01` for blue-red


##### TX LDAC Quality Setting

**Description:**<br/>
Fetch the current TX LDAC Quality Setting

**GET Command**:<br/>
0x44C

**SET Command**:<br/>
0x44D

**Payload**:<br/>
1 Byte
- Audio quality first (`0x01`)
- Standard (`0x02`)
- Connection first (`0x03`)
- Adaptive (`0x04`)

##### SPDIF Volume Adjustment Setting

**Description:**<br/>
Whether the SPDIF Output level should be controlled by the volume knob

**GET Command**:<br/>
0x452

**SET Command**:<br/>
0x453

**Payload**:<br/>
Either `0x00` or `0x01`

##### Upsampling Setting

**Description:**<br/>
Whether the DAC should upsample the audio data to 384KHz

**GET Command**:<br/>
0x450

**SET Command**:<br/>
0x451

**Payload**:<br/>
Either `0x00` or `0x01`

##### Volume Setting

**Description:**<br/>
The current volume setting.
This is usually controlled by the knob on the device.

**GET Command**:<br/>
0x412

**SET Command**:<br/>
0x402

**Payload**:<br/>
1 Byte `0x00`-`0x3C`
Decimal 0-60

##### DAC Lowpass Setting

**Description:**<br/>
The current DAC lowpass setting

**GET Command**:<br/>
0x411

**SET Command**:<br/>
0x401

**Payload**:<br/>
1 Byte
- Sharp Roll-Off Filter (`0x00`)
- Slow Roll-Off Filter (`0x01`)
- Short Delay Sharp Roll-Off Filter (`0x02`)
- Short Delay Slow Roll-Off Filter (`0x03`)

##### Output Balance

**Description:**<br/>
The current balance setting.

Values can be between L12 and R12 with 0 being R0 `0x0200`

**GET Command**:<br/>
0x413

**SET Command**:<br/>
0x403


**Payload**:<br/>
2 Byte
1. L (`0x01`) or R (`0x02`)
2. 0-12 Decimal Left or Right (`0x00`-`0x0C`)


##### Name

**Description:**<br/>
The bluetooth name advertised by the device. 30 Bytes UTF-8 max

**GET Command**:<br/>
0x445

**SET Command**:<br/>
0x446

**Description**:<br/>
Because the name can be up to 30 UTF-8 bytes long, this command behaves a bit differently from the others.
One `Set Name` command may consist of up to four writes. You may even use emoji

###### Basic Example SET

This example contains the short name `Hallo` and therefore needs no splitting as it fits in a single message.

`00 0a 04 46 ff 05 48 61 6c 6c 6f ff`

- First, there's the magic: `0x000a0`
- Then, there's the command: `0x446`
- Then, there's the start of payload marker: `0xff`
- Then, there's the length of the name payload: `0x05`
- Then, there are the UTF-8 Bytes of `Hallo`: `0x48616c6c6f`
- Finally, there's the end of payload marker: `0xff`

###### Advanced example SET

This example contains the 30 character long name `KlausKlausKlausKlausKlausKlaus`, which was split into four messages.

1. `00 0a 04 46 ff 1e 4b 6c 61 75 73 4b 6c 61` - `KlausKla`
2. `00 0a 04 46 75 73 4b 6c 61 75 73 4b 6c 61` - `usKlausKla`
2. `00 0a 04 46 75 73 4b 6c 61 75 73 4b 6c 61` - `usKlausKla`
3. `00 0a 04 46 75 73 ff` - `us`

In this example, the `0x1e` represents the length of 30 characters.

##### Example GET

The same logic applies to the get name response. Here, the name is `FiiO BTA30`.

1. `00 0a 04 46 ff 0a 46 69 69 4f 20 42 54 41` - Length: 0x0a (10), `FiiO BTA`
1. `00 0a 04 46 33 30 ff` - `30`


#### Mode-dependant Commands

##### Input Sources

**Description:**<br/>
Fetch/select which sources are enabled for the mode

**GET Command**:<br/>
0x448

**SET Command**:<br/>
0x449

**GET Payload**:<br/>
1 Byte
`0x01` for RX, `0x02` for TX

**GET Response/SET Payload**:<br/>
2 Bytes
1. `0x01` for RX, `0x02` for TX
2. Settings
    - USB (`0x01`)
    - Coax (`0x02`)
    - USB & Coax (`0x03`)

##### GET Volume Mode

**Description:**<br/>
Fetch/select the volume mode setting for the mode

**GET Command**:<br/>
0x44E

**SET Command**:<br/>
0x44F

**GET Payload**:<br/>
1 Byte
`0x01` for RX, `0x02` for TX

**GET Response/SET Payload**:<br/>
2 Bytes
1. `0x01` for RX, `0x02` for TX
2. Settings
    - Adjustable (`0x01`)
    - Fixed at 30% (`0x02`)
    - Fixed at 50% (`0x03`)
    - Fixed at 70% (`0x04`)
    - Fixed at 100% (`0x05`)

##### GET Enabled Codecs

**Description:**<br/>
Fetch/set the enabled codecs for the mode
APTX-LL seems to only be available in TX?

**GET Command**:<br/>
0x417

**GET Command**:<br/>
0x407

**GET Payload**:<br/>
1 Byte
`0x01` for RX, `0x02` for TX

**GET Response/SET Payload**:<br/>
2 Bytes
1. `0x01` for RX, `0x02` for TX
2. Sum of supported codecs
    - AAC (`2`)
    - LDAC (`4`)
    - APTX (`8`)
    - APTX-LL (`16`)
    - APTX-HD (`32`)

#### Special SET-only commands

##### Factory reset

**Description:**<br/>
Does a factory reset

**Command**:<br/>
0x404

**Response**:<br/>
None

##### Power Off

**Description:**<br/>
Turns off the BTA30

**Command**:<br/>
0x425

**Response**:<br/>
None

##### Clear Pairing

**Description:**<br/>
Delete all pairing information from the BTA30

**Command**:<br/>
0x443

**Response**:<br/>
None

