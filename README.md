# node-pcsclite

[![npm](https://img.shields.io/npm/v/@pokusew/pcsclite.svg?maxAge=2592000)](https://www.npmjs.com/package/@pokusew/pcsclite)
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
const pcsclite = require('pcsclite');

const pcsc = pcsclite();

pcsc.on('reader', function(reader) {

    console.log('New reader detected', reader.name);

    reader.on('error', function(err) {
        console.log('Error(', this.name, '):', err.message);
    });

    reader.on('status', function(status) {
        console.log('Status(', this.name, '):', status);
        /* check what has changed */
        const changes = this.state ^ status.state;
        if (changes) {
            if ((changes & this.SCARD_STATE_EMPTY) && (status.state & this.SCARD_STATE_EMPTY)) {
                console.log("card removed");/* card removed */
                reader.disconnect(reader.SCARD_LEAVE_CARD, function(err) {
                    if (err) {
                        console.log(err);
                    } else {
                        console.log('Disconnected');
                    }
                });
            } else if ((changes & this.SCARD_STATE_PRESENT) && (status.state & this.SCARD_STATE_PRESENT)) {
                console.log("card inserted");/* card inserted */
                reader.connect({ share_mode : this.SCARD_SHARE_SHARED }, function(err, protocol) {
                    if (err) {
                        console.log(err);
                    } else {
                        console.log('Protocol(', reader.name, '):', protocol);
                        reader.transmit(new Buffer([0x00, 0xB0, 0x00, 0x00, 0x20]), 40, protocol, function(err, data) {
                            if (err) {
                                console.log(err);
                            } else {
                                console.log('Data received', data);
                                reader.close();
                                pcsc.close();
                            }
                        });
                    }
                });
            }
        }
    });

    reader.on('end', function() {
        console.log('Reader',  this.name, 'removed');
    });
});

pcsc.on('error', function(err) {
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

It frees the resources associated with this PCSCLite instance. At a low level it calls [`SCardCancel`](http://pcsclite.alioth.debian.org/pcsc-lite/node21.html) so it stops watching for new readers.


### Class: CardReader

The CardReader object is an EventEmitter that allows to manipulate a card reader.

#### Event:  'error'

* *err* `Error Object`. The error.

#### Event:  'end'

Emitted when the card reader has been removed.

#### Event:  'status'

* *status* `Object`.
    * *state* The current status of the card reader as returned by [`SCardGetStatusChange`](http://pcsclite.alioth.debian.org/pcsc-lite/node20.html)
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

### Can I use this library in my [Electron](http://electron.atom.io/) app?

**Yes, you can!** It works well.

But please read carefully [Using Native Node Modules](http://electron.atom.io/docs/tutorial/using-native-node-modules/) guide in Electron documentation to fully understand the problematic.

**Note**, that because of Node Native Modules, you must build your app on target platform (you must run Windows build on Windows machine, etc.).  
You can use CI/CD server to build your app for certain platforms.  
For Windows, I recommend you to use [AppVeyor](https://appveyor.com/).  
For macOS and Linux build, there are plenty of services to choose from, for example [CircleCI](https://circleci.com/), [Travis CI](https://travis-ci.com/) [CodeShip](https://codeship.com/).

## License

[ISC](/LICENSE.md)
