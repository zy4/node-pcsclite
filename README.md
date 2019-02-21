# node-pcsclite

[![npm](https://img.shields.io/npm/v/@pokusew/pcsclite.svg)](https://www.npmjs.com/package/@pokusew/pcsclite)
[![node-pcsclite channel on discord](https://img.shields.io/badge/discord-join%20chat-61dafb.svg)](https://discord.gg/bg3yazg)

Bindings over pcsclite to access Smart Cards. It works in **Linux**, **macOS** and **Windows**.

> **Looking for library to work easy with NFC tags?**  
take a look at [nfc-pcsc](https://github.com/pokusew/nfc-pcsc) which offers easy to use high level API for detecting / reading and writing NFC tags and cards


## Content

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Installation](#installation)
- [Example](#example)
- [Behavior on different OS](#behavior-on-different-os)
- [API](#api)
  - [Class: PCSCLite](#class-pcsclite)
    - [Event:  'error'](#event--error)
    - [Event:  'reader'](#event--reader)
    - [pcsclite.close()](#pcscliteclose)
  - [Class: CardReader](#class-cardreader)
    - [Event:  'error'](#event--error-1)
    - [Event:  'end'](#event--end)
    - [Event:  'status'](#event--status)
    - [reader.connect([options], callback)](#readerconnectoptions-callback)
    - [reader.disconnect(disposition, callback)](#readerdisconnectdisposition-callback)
    - [reader.transmit(input, res_len, protocol, callback)](#readertransmitinput-res_len-protocol-callback)
    - [reader.control(input, control_code, res_len, callback)](#readercontrolinput-control_code-res_len-callback)
    - [reader.close()](#readerclose)
- [FAQ](#faq)
  - [Can I use this library in my Electron app?](#can-i-use-this-library-in-my-electron-app)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Installation

In order to install the package you need to **have installed in the system the
pcsclite libraries**.

In **macOS** and **Windows** you **don't have to install** anything.

> For example, in Debian/Ubuntu:
> ```bash
> apt-get install libpcsclite1 libpcsclite-dev
> ```
> To run any code you will also need to have installed the pcsc daemon:
> ```bash
> apt-get install pcscd
> ```

Once you have all needed libraries, you can install using npm:

```bash
npm install @pokusew/pcsclite --save
```

## Example

```javascript
const pcsclite = require('@pokusew/pcsclite');

const pcsc = pcsclite();

pcsc.on('reader', (reader) => {

    console.log('New reader detected', reader.name);

    reader.on('error', err => {
        console.log('Error(', reader.name, '):', err.message);
    });

    reader.on('status', (status) => {

        console.log('Status(', reader.name, '):', status);

        // check what has changed
        const changes = reader.state ^ status.state;

        if (!changes) {
            return;
        }

		// noinspection JSBitwiseOperatorUsage
        if ((changes & reader.SCARD_STATE_EMPTY) && (status.state & reader.SCARD_STATE_EMPTY)) {

            console.log("card removed");

            reader.disconnect(reader.SCARD_LEAVE_CARD, err => {

                if (err) {
                    console.log(err);
                    return;
                }

                console.log('Disconnected');

            });

            return;

        }

		// noinspection JSBitwiseOperatorUsage
        if ((changes & reader.SCARD_STATE_EMPTY) && (status.state & reader.SCARD_STATE_EMPTY)) {

            console.log("card inserted");

            reader.connect({ share_mode: reader.SCARD_SHARE_SHARED }, (err, protocol) => {

                if (err) {
                    console.log(err);
                    return;
                }

                console.log('Protocol(', reader.name, '):', protocol);

                reader.transmit(Buffer.from([0x00, 0xB0, 0x00, 0x00, 0x20]), 40, protocol, (err, data) => {

                    if (err) {
                        console.log(err);
                        return;
                    }

                    console.log('Data received', data);
                    reader.close();
                    pcsc.close();

                });

            });

            // noinspection UnnecessaryReturnStatementJS
            return;
        }

    });

    reader.on('end', () => {
        console.log('Reader', reader.name, 'removed');
    });

});

pcsc.on('error', err => {
    console.log('PCSC error', err.message);
});
```

## Behavior on different OS

TODO document


## API

### Class: PCSCLite

The PCSCLite object is an EventEmitter that notifies the existence of Card Readers.

#### Event:  'error'

* *err* `Error Object`. The error.

#### Event:  'reader'

* *reader* `CardReader`. A CardReader object associated to the card reader detected

Emitted whenever a new card reader is detected.

#### pcsclite.close()

It frees the resources associated with this PCSCLite instance. At a low level it calls [`SCardCancel`](https://pcsclite.alioth.debian.org/api/group__API.html#gaacbbc0c6d6c0cbbeb4f4debf6fbeeee6) so it stops watching for new readers.

#### pcsclite.readers

An object containing all detected readers by name. Updated as readers are attached and removed.

### Class: CardReader

The CardReader object is an EventEmitter that allows to manipulate a card reader.

#### Event:  'error'

* *err* `Error Object`. The error.

#### Event:  'end'

Emitted when the card reader has been removed.

#### Event:  'status'

* *status* `Object`.
    * *state* The current status of the card reader as returned by [`SCardGetStatusChange`](https://pcsclite.alioth.debian.org/api/group__API.html#ga33247d5d1257d59e55647c3bb717db24)
    * *atr* ATR of the card inserted (if any)

Emitted whenever the status of the reader changes.

#### reader.connect([options], callback)

* *options* `Object` Optional
    * *share_mode* `Number` Shared mode. Defaults to `SCARD_SHARE_EXCLUSIVE`
    * *protocol* `Number` Preferred protocol. Defaults to `SCARD_PROTOCOL_T0 | SCARD_PROTOCOL_T1`
* *callback* `Function` called when connection operation ends
    * *error* `Error`
    * *protocol* `Number` Established protocol to this connection.

Wrapper around [`SCardConnect`](https://pcsclite.alioth.debian.org/api/group__API.html#ga4e515829752e0a8dbc4d630696a8d6a5).
Establishes a connection to the reader.

#### reader.disconnect(disposition, callback)

* *disposition* `Number`. Reader function to execute. Defaults to `SCARD_UNPOWER_CARD`
* *callback* `Function` called when disconnection operation ends
    * *error* `Error`

Wrapper around [`SCardDisconnect`](https://pcsclite.alioth.debian.org/api/group__API.html#ga4be198045c73ec0deb79e66c0ca1738a).
Terminates a connection to the reader.

#### reader.transmit(input, res_len, protocol, callback)

* *input* `Buffer` input data to be transmitted
* *res_len* `Number`. Max. expected length of the response
* *protocol* `Number`. Protocol to be used in the transmission
* *callback* `Function` called when transmit operation ends
    * *error* `Error`
    * *output* `Buffer`

Wrapper around [`SCardTransmit`](https://pcsclite.alioth.debian.org/api/group__API.html#ga9a2d77242a271310269065e64633ab99).
Sends an APDU to the smart card contained in the reader connected to.

#### reader.control(input, control_code, res_len, callback)

* *input* `Buffer` input data to be transmitted
* *control_code* `Number`. Control code for the operation
* *res_len* `Number`. Max. expected length of the response
* *callback* `Function` called when control operation ends
    * *error* `Error`
    * *output* `Buffer`

Wrapper around [`SCardControl`](https://pcsclite.alioth.debian.org/api/group__API.html#gac3454d4657110fd7f753b2d3d8f4e32f).
Sends a command directly to the IFD Handler (reader driver) to be processed by the reader.

#### reader.close()

It frees the resources associated with this CardReader instance.
At a low level it calls [`SCardCancel`](https://pcsclite.alioth.debian.org/api/group__API.html#gaacbbc0c6d6c0cbbeb4f4debf6fbeeee6) so it stops watching for the reader status changes.


## FAQ

### Can I use this library in my [Electron](https://electron.atom.io/) app?

**Yes, you can!** It works well.

But please read carefully [Using Native Node Modules](https://electron.atom.io/docs/tutorial/using-native-node-modules/) guide in Electron documentation to fully understand the problematic.

**Note**, that because of Node Native Modules, you must build your app on target platform (you must run Windows build on Windows machine, etc.).  
You can use CI/CD server to build your app for certain platforms.  
For Windows, I recommend you to use [AppVeyor](https://appveyor.com/).  
For macOS and Linux build, there are plenty of services to choose from, for example [CircleCI](https://circleci.com/), [Travis CI](https://travis-ci.com/) [CodeShip](https://codeship.com/).

## License

[ISC](/LICENSE.md)
