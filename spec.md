# DIDComm over Bluetooth

This document aims to describe an interoperable method in which different agents, from different vendors can perform DIDComm based communication using Bluetooth as a means of transport.

There are multiple versions of Bluetooth, and each version of Bluetooth supports multiple protocols, profiles and modes. Because of the wide range of methods to connect and exchange data, defining an interoperable method to exchange data over Bluetooth is not as straightforward as defining the transport for, lets say, HTTP. Bluetooth can not be treated as being on the same layer of HTTP, but more as TCP, where HTTP is a protocol that uses TCP. We need to define the specific Bluetooth protocols and methods to use to achieve interoperable communication.

This document describes how DIDComm based communication can be achieved using Bluetooth Low Energy. Bluetooth Low Energy (BLE) is developed by Bluetooth SIG that builds on the lessons learned from building lots of Bluetooth Classic devices. Compared to Bluetooth Classic, BLE consumes less power, requires less time and effort to pair devices, and provides lower connection speeds.

Although Classic Bluetooth has a bigger range and higher throughput, BLE shines at interoperability and the ease of connecting.

> Some methods in this document describe some well known BLE practices. In the future it may be best to reduce the size of the document to a required minimum, referencing other documents where needed.

## Bluetooth Low Energy Protocol Stack

There are two mechanisms a BLE device can use to communicate to the outside world: broadcasting or connecting. These mechanisms are subjected to the Generic Access Profile (GAP) guidelines. GAP defines how BLE-enabled devices can make themselves available and how two devices can communicate directly with each other.

![Bluetooth Low Energy protocol stack](https://www.mdpi.com/futureinternet/futureinternet-11-00194/article_deploy/html/images/futureinternet-11-00194-g006-550.jpg)
_Bluetooth Low Energy protocol stack. ([Source](https://www.mdpi.com/1999-5903/11/9/194/htm))_

## Identifiers

Bluetooth uses UUIDs to advertise services. This allows DIDComm enabled devices to advertise and search for other DIDComm enabled devices, as long as all devices use the same service identifier. The "DIDComm Service UUID" is intended for exchanging DIDComm encrypted JSON messages (similar to the `application/didcomm-encrypted+json` media type, except that for BLE data exchange we use buffers). The write and read characteristic UUIDs are explained in [Separating Read from Write](#Separating-Read-from-Write) below.

- **DIDComm Service UUID**: `d2f195b6-2e80-4ab0-be24-32ebe761352f`
- **Write DIDComm message characteristic UUID**: `c3103ded-afd7-477c-b279-2ad264e20e74`
- **Read DIDComm message characteristic UUID**: `e6e97879-780a-4c9b-b4e6-dcae3793a3e8`

:::info
The UUIDs listed above are randomly generated. Commonly used UUIDs can be registered at the Bluetooth Special Interest Group (SIG). Besides public recognition this saves some bytes during transport as the UUID will be 16bit instead of 128.
:::

## Connecting

Broadcasting only allows for one way communication, where the broadcaster broadcasts **public** advertising data packets. No connection is required.

When connecting, the devices must first perform a handshake to transfer data. This is the more common usage of BLE. Connecting also allows for two way communication.

There are two roles when connecting:

- **Peripheral**: A device that advertises its presence so central devices can establish a connection. After connecting, peripherals no longer broadcast data to other central devices and stay connected to the device that accepted the connection request.
- **Central**: A device that initiates a connection with a peripheral device by first listening to the advertising packets. A central device can connect to many other peripheral devices.

### Steps

1. One DIDComm enabled system will start as the peripheral role. It will advertise the DIDComm service listed above using the corresponding UUID.
2. Another DIDComm enabled system will start as the central role and begin scanning for a peripheral device that is advertising the DIDComm service.
3. Once the central device finds a peripheral that advertises the DIDComm service, it will connect to this service.

## Exchanging messages

Once a connection is created the devices are able to exchange data between each other. However we need to standardize the exchange process as BLE doesn't allow you to just "send" data between devices.

### Chunking

Just like with TCP, BLE deals with something called a Maximum Transmission Unit (MTU). This indicates the maximum size in bytes that can be sent at once. If you want to send messages larger than the MTU, data would need to be split and send in chunks.

The MTU differs greatly between device, Bluetooth version and Bluetooth chip. We could state a static, very low MTU that would work on all devices, but this would limit througput for higher end devices. Devices can discover the maximum MTU of the connected Bluetooth device and should always take the lowest MTU between your own device and the other device. This is standard practice.

1. Stringify JSON and transform to bytes.
1. Take lowest MTU (lMTU) between your device and other device.
1. Split data into chunks of size lMTU.
1. Update the characteristic value in a loop until all chunks are shared.
1. Update the characteristic value to `EOM` to indicate the end of message.

#### Acknowledgement of packets (chunks)

There are different ways to acknowledge packets:

- No Response: packets are not acknowledged, data loss is possible.
- Indication: packet is received and acknowledged by the application on the other end.
- Notification: packet is acknowledged on the link layer (see _Bluetooth Low Energy protocol stack_ diagram above for difference)

Indication will drastically reduce the throughput as each packet must be acknowledged by the user application before the next packet can be sent. When using notifications, an acknowledgement will still be sent, but just that the data is received, not whether the data is accepted. Because of the chunking of messages, an individual packet doesn't have a lot of meaning anyway.

Therefore using notifications seems like the best approach. This assures all packets are successfully transmitted. In case of problems with the message itself, a problem report or something similar can be sent.

### Separating Read from Write

The identifiers section above mentions a separate characteristic UUID for reading and writing messages. This is to separate between sending messages from Central to Peripheral (writing), and sending messages from Peripheral to Central (reading).

All data is stored on the peripheral device. Therefore the operations are written as actions taken by the central. The central sends a message by writing to the peripheral (using "Write DIDComm message characteristic UUID"), and the central receives a message by reading from the peripheral ("Read DIDComm message characteristic UUID").

As messages need to be chunked and each attribute can only contain a single value at a time, it is easiest to separate the sending from the receiving of messages.

### Requiring encryption (Pairing)

By default BLE connections are not secure. Even though DIDComm messages are encrypted themselves, messages can still be intercepted.

Pairing allows two Bluetooth devices to pair with each other and establish an encrypted connection. The security of the encrypted exchange depends on the pairing method used.

Bluetooth supports four different pairing methods:

- **Just Works**: Use pin 000000. Doesn't protect against MITM attacks.
- **Numeric Comparison**: Confirm that two codes are identical.
- **Passkey Entry**: Require the user to enter a code.
- **Out-of-Band (OOB)**: Make use of other mediums to exchange some verification data.

Because supported pairing methods differ per device type we can't define the standard method to use. During the connection phase the supported methods are exchanged. For example iOS automatically pairs when an encrypted attribute is used, and doesn't allow to specify which method to use. If the other device supports entering a code it will use the "Passkey Entry" method, otherwise it will use the "Just Works" method.

For more info, The Bluetooth website contains a 5 part blog series about Bluetooth pairing: https://www.bluetooth.com/blog/bluetooth-pairing-part-1-pairing-feature-exchange/

### Using DIDComm Out-of-Band

Two devices can create a Bluetooth connection without first sending an Out-of-Band (OOB) message. This means the first message exchanged over Bluetooth can't be sent encrypted. I see two possible methods to overcome this:

1. Define a separate service / characteristic for exchanging plaintext JSON messages (like `application/didcomm-plain+json`) or signed JSON messages (like `application/didcomm-signed+json`). This could be used for sending the initial OOB message. See [Supporting other formats and encodings](#Supporting-other-formats-and-encodings) for more information.
1. Exchange OOB using other method than Bluetooth (e.g. QR). In the case of QR, the Peripheral creates an OOB QR, The Central scans the QR, Connects to the Peripheral and sends and encrypted message using the info from the OOB.
   - Encoding an OOB message in a QR has some limitations with size. Url shorteners can be used, but this means Bluetooth messaging can't be done without an internet connection.
   - However using an OOB in the form of, for example, a QR does give some extra verification. This allows to exchange some data to verify you're connecting to the correct Bluetooth device (and not someone around the corner. This is still not secure, as the QR can be scanned from a distance). This is like the OOB Bluetooth pairing method, and can also be achieved using one of the other pairing steps.

## Throughput

There are a quite a lot of variables that impact the throughput over a BLE connection, and theoretical throughput often doesn't conform to practical throughput. Bluetooth 4.2 introduced Data Packet Length Extension (DLE) which drastically improves throughput ([more info](https://punchthrough.com/maximizing-ble-throughput-part-3-data-length-extension-dle-2/)) by allowing bigger packet sizes. Dependant on the Bluetooth library / API used, this is enabled automatically when both devices support it (e.g. Android + iOS). The article mentioned above states that based on their testing that **~50 KB/s** data throughput is achievable on newer Android and iOS devices with DLE enabled on the default LE 1M PHY. Higher rates should be possible.

### Example

A JSON packed DIDComm message (taken from DIDComm v1) can easily reach 7000 bytes (7 KB). This is an `offer-credential` message with indy offer for two attributes. If we assume a data throughput of 50 KB/s, 7KB will take approx. 0.14 seconds to exchange. If higher throughput is desirable, see [Reducing Size of Messages](#Reducing-Size-of-Messages) below for more information.

## Reducing Size of Messages

The document mainly focusses on exchange of stringified JSON messages. As mentioned above, this can achieve fairly reasonable throughput of messages. However making use of different encodings and compression formats can drastically reduce the size of a message, thus also decreasing the time it will take to exchange messages.

Protobuffers is a topic already discussed in the DIDComm WG, however this adds complexity that maybe not everyone wants to support. Other compression formats can achieve sizes similar to protobuffers, but will take up more CPU resources. Based on a [comparison](https://nilsmagnus.github.io/post/proto-json-sizes/) between plain JSON, protobuf and gzipped JSON the size _could_ be reduced to the sizes listed below. However this can differ drastically based on message structure and size, and in the case of gzip also compression level.

| size raw json | size protobuf | size gzipped json | size gzipped protobuf |
| ------------- | ------------- | ----------------- | --------------------- |
| 6578          | 2413          | 1970              | 1629                  |

## Supporting other formats and encodings

Other encodings and compression formats could be supported using different service or characteristic UUIDs. One method is creating a separate service UUID for each encoding / format, keeping the read / write characteristic UUIDs the same. E.g.:

- **DIDComm JSON service UUID**: `d2f195b6-2e80-4ab0-be24-32ebe761352f`
- **DIDComm Protobuffer service UUID**: `16728cdc-0593-465f-bc8f-e40f4f32cdbe`
- **DIDComm JSON Gzipped service UUID**: `ee45f789-23c2-4e68-ae9e-937be3d99eb4`
- ...etc

Another method could be having the same service UUID for all encodings and formats (a general "DIDComm Service UUID"), and creating a separate Characteristic UUID for each encoding / format. E.g.:

- **Write DIDComm JSON message characteristic UUID**: `c3103ded-afd7-477c-b279-2ad264e20e74`
- **Read DIDComm JSON message characteristic UUID**: `e6e97879-780a-4c9b-b4e6-dcae3793a3e8`
- **Write DIDComm Protobuffer message characteristic UUID**: `2a68c38c-1909-4431-9570-56373faf0ba0`
- **Read DIDComm Protobuffer message characteristic UUID**: `97ec509f-7d9c-45da-aced-acf55a08e324`
- ...etc

> TODO: add method using descriptors. Descriptors allow to set metadata about a characteristic. We could create a custom 'encoding/format' descriptor that describes what should used. To save data a number could be used.
> 1 --> DIDComm Encrypted JSON
> 2 --> DIDComm Plaintext JSON
> ...

## Remaining

- Most other DIDComm transports make use of `serviceEndpoint`. Bluetooth works through discovery of a predefined service UUID. How should this be handled?
- DIDComm is simplex by default. Should we disallow the peripheral sending messages to the central until the central sends a message containing the ["DIDComm Messaging Return-Route" header](https://github.com/decentralized-identity/didcomm-messaging/blob/a8d6d48b13cb1d4a02131d35c056f185ab09d34f/extensions/return_route/main.md)?
- Exchange of Peer DIDs (not specific to Bluetooth transport)
- RSSI (should we specify or recommend a minimum signal strength?)
- Connection reuse
- Diagrams

## Resources

- [Classic Bluetooth vs Bluetooth Low Energy - A Round by Round Battle](https://www.semiconductorstore.com/blog/2014/Classic-Bluetooth-vs-Bluetooth-Low-Energy-A-Round-by-Round-Battle/860/)
- [Comparing JSON to Proto](https://nilsmagnus.github.io/post/proto-json-sizes/)
- [BLE Throughput](https://interrupt.memfault.com/blog/ble-throughput-primer)
- [BLE Guides](https://github.com/chrisc11/ble-guides)
- [Maximizing BLE Throughput - Data Length Extension (DLE)](https://punchthrough.com/maximizing-ble-throughput-part-3-data-length-extension-dle-2/)
- [Maximizing BLE Throughput - iOS and Android](https://punchthrough.com/maximizing-ble-throughput-on-ios-and-android/)
- [Maximizing BLE Throughput - Use Larger ATT MTU](https://punchthrough.com/maximizing-ble-throughput-part-2-use-larger-att-mtu-2/)
- [BLE - A Primer](https://interrupt.memfault.com/blog/bluetooth-low-energy-a-primer)
- [Bluetooth 5 Speed Maximum Throughput](https://www.novelbits.io/bluetooth-5-speed-maximum-throughput/)
- [Apple Accesory Design Guidelines](https://developer.apple.com/accessories/Accessory-Design-Guidelines.pdf)
- [Apple Bluetooth Security](https://support.apple.com/en-in/guide/security/sec82597d97e/web)
- [Bluetooth Site](https://www.bluetooth.com/)
- [Bluetooth Bottleneck](https://devzone.nordicsemi.com/f/nordic-q-a/613/bottleneck#post-id-1929)
- [Android BLE Issues](https://github.com/iDevicesInc/SweetBlue/wiki/Android-BLE-Issues)
- [Android BLE Documentation](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter.html)
- [Android BLE Guide](https://developer.android.com/guide/topics/connectivity/bluetooth-le)
- iOS / Android BLE showcases
  - GitGarage
    - https://github.com/GitGarage/BLEMingleiOS
    - https://github.com/GitGarage/BLEMingleDroid
  - https://github.com/itanbp/android-ble-peripheral-central
  - https://developer.apple.com/documentation/corebluetooth/transferring_data_between_bluetooth_low_energy_devices
  - https://github.com/android/connectivity-samples/tree/main/BluetoothLeChat
  - https://github.com/android/connectivity-samples/tree/main/BluetoothLeGatt
