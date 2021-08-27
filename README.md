# Create a C/C++ app with the ACR122U (Linux)



The ACR122U NFC Reader is a PC-linked contactless smart card reader/writer. In this tutorial, we are going to see how to create a simple C/C++ application running on Linux that communicates with the ACR122U. This application will be used to send and retrieve data from the MIFARE Ultralight and MIFARE Classic 1K tags.

## 1. Installing the driver

The first step is to install the driver of the ACR122U. This step is needed for both **developing** and **running** the application we are going to create.

Let's get to the [official website](https://www.acs.com.hk/en/products/3/acr122u-usb-nfc-reader/), on the *Downloads* section.

Two downloads are available for Linux:

* [PC/SC Driver Package](https://www.acs.com.hk/download-driver-unified/11929/ACS-Unified-PKG-Lnx-118-P.zip), which contains packages for different distributions (Debian, Fedora, Ubuntu, Raspbian...). This is the easiest way to install the driver if you are using one of these distributions.
* [PC/SC Drivers](https://www.acs.com.hk/download-driver-unified/12030/ACS-Unified-Driver-Lnx-Mac-118-P.zip), which contains the sources of the driver. Here, it's a bit more complex because we have to build the driver before installing it, but it can work on any Linux distribution.

To make this tutorial as compatible as possible, I will show how to build and install the [PC/SC Drivers](https://www.acs.com.hk/download-driver-unified/12030/ACS-Unified-Driver-Lnx-Mac-118-P.zip). First, download and uncompress the sources:

```shell
wget https://www.acs.com.hk/download-driver-unified/12030/ACS-Unified-Driver-Lnx-Mac-118-P.zip
unzip ACS-Unified-Driver-Lnx-Mac-118-P.zip
cd ACS-Unified-Driver-Lnx-Mac-118-P/
```

A `README` file indicates what is required to build the sources:

* pcsclite 1.8.3 or above
* libusb 1.0.9 or above
* flex
* perl
* pkg-config

In addition, you will need to install these two other packages:

* pcsclite-devel
* libusb-devel

So you should install all of these packages first. On Fedora, the command would be:

```shell
sudo dnf install pcsc-lite libusb flex perl pkg-config pcsc-lite-devel libusb-devel
```

Then we can uncompress, configure, build, and install the sources:

```shell
tar -xf acsccid-1.1.8.tar.bz2
cd acsccid-1.1.8/
./configure
make
sudo make install
```

Two more steps are needed. First, you need to make sure that the `pcscd` service is running:

```shell
sudo systemctl start pcscd
sudo systemctl enable pcscd
```

Then, you have to run the following command:

```shell
echo "blacklist pn533_usb" | sudo tee -a /etc/modprobe.d/blacklist-libnfc.conf
```

This is used to avoid conflicts between the driver and one of the default kernel modules.

You can now reboot your computer.

## 2. Installing the SDK

The next step is to install the SDK, which is required for **developing** the application but not for running it.

We will not use the SDK proposed on the official website. It is not open-source and only available for the i386 and x86_64 CPU architecture, which means that it is not possible to use it on Raspberry Pi for example (with their ARM CPU architecture).

Instead, we are going to use [PCSC lite](https://pcsclite.apdu.fr/). You will need the `systemd-devel` package:

```shell
sudo dnf install systemd-devel
```

Then you can download, uncompress, configure, build, and install the sources:

```shell
wget https://pcsclite.apdu.fr/files/pcsc-lite-1.9.3.tar.bz2
tar -xf pcsc-lite-1.9.3.tar.bz2
cd pcsc-lite-1.9.3/
./configure
make
sudo make install
```

## 3. Check that the ACR122U is detected by your computer (optional)

Before starting to create the application, it's better to make sure your computer recognizes the reader: plug in the reader, install the `pcsc-tools` package, and run the `pscs_scan` command.

If your reader is not detected, the following message will appear:

```
Waiting for the first reader... 
```

A solution is to restart the `pcscd` service:

```
sudo systemctl restart pcscd
```

If it still doesn't work, you can run the following command:

```shell
systemctl status pcscd
```

which may give you the reason of why the reader is not detected.

## 4. Creating the application

Let's now create the application. I will use the C language with the *gcc* compiler, but it's also compatible in C++ with *g++*. To communicate with the reader, we are going to use the [PC/SC Lite API (WinSCard)](https://pcsclite.apdu.fr/api/group__API.html). All the code is written is a `main.c` file which is available on [GitHub](https://github.com/Rylern/ACR122U-tutorial).

In this tutorial, we will be use the MIFARE Ultralight and MIFARE Classic 1K tags.

First, let's check that we can include the WinSCard library:

```c
#include <winscard.h>

int main() {
	return 0;
}
```

The header file of the WinSCard library is in `/usr/local/include/PCSC`. Therefore, the program can be compiled and run with:

```shell
gcc main.c -lpcsclite -I/usr/local/include/PCSC
./a.out
```

Now, we will use the documentation of the [WinSCard API](https://pcsclite.apdu.fr/api/group__API.html) to communicate with the reader. The first step is to establish a context (which has to be released at the end of the application):

```c
#include <stdio.h>
#include <stdlib.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;

void establishContext() {
	SCardEstablishContext(SCARD_SCOPE_SYSTEM, NULL, NULL, &applicationContext);
}
void releaseContext() {
	SCardReleaseContext(applicationContext);
}

int main() {
	establishContext();
    
	releaseContext();
	return 0;
}
```

We use a global variable `applicationContext` because we will need it for future functions. This code does not check whether the operations succeeded or not. Therefore, we add:

```c
#include <stdio.h>
#include <stdlib.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;

void establishContext() {
	LONG status = SCardEstablishContext(SCARD_SCOPE_SYSTEM, NULL, NULL, &applicationContext);
	if (status == SCARD_S_SUCCESS) {
		printf("Context established\n");
	} else {
		printf("Establish context error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}
void releaseContext() {
	LONG status = SCardReleaseContext(applicationContext);
	if (status == SCARD_S_SUCCESS) {
		printf("Context released\n");
	} else {
		printf("Release context error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}

int main() {
	establishContext();
    
	releaseContext();
	return 0;
}
```

This will give us some information on the success of each operation, and exit the program as soon as a failure occures.

Then, we have to list the available readers:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;
LPSTR reader = NULL;

void establishContext() {}
void releaseContext() {}

void listReaders() {
	DWORD readers = SCARD_AUTOALLOCATE;
	LONG status = SCardListReaders(applicationContext, NULL, (LPSTR)&reader, &readers);
	
	if (status == SCARD_S_SUCCESS) {
		char *p = reader;
		while (*p) {
			printf("Reader found: %s\n", p);
			p += strlen(p) +1;
		}
	} else {
		printf("List reader error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}
void freeListReader() {
	LONG status = SCardFreeMemory(applicationContext, reader);
	if (status == SCARD_S_SUCCESS) {
		printf("Reader list free\n");
	} else {
		printf("Free reader list error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}

int main() {
	establishContext();
	listReaders();
	
	freeListReader();
	releaseContext();
	return 0;
}
```

We add a global variable `reader` which will be used to identify the reader for further use. The function `SCardListReaders` allocates some memory for `reader`, which has to be freed with the `SCardFreeMemory` function before the end of the program. If you get an error here even so the reader is plugged in, be sure to check that the ACR122U is detected by your computer.

The next step is to establish a connection with a tag. Put a tag on the reader (for example a MIFARE Ultralight), and run the following code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;
LPSTR reader = NULL;
SCARDHANDLE connectionHandler;
DWORD activeProtocol;

void establishContext() {}
void releaseContext() {}
void listReaders() {}
void freeListReader() {}

void connectToCard() {
	activeProtocol = -1;

	LONG status = SCardConnect(applicationContext, reader, SCARD_SHARE_SHARED, SCARD_PROTOCOL_T0 | SCARD_PROTOCOL_T1, &connectionHandler, &activeProtocol);
	if (status == SCARD_S_SUCCESS) {
		printf("Connected to card\n");
	} else {
		printf("Card connection error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}
void disconnectFromCard() {
	LONG status = SCardDisconnect(connectionHandler, SCARD_LEAVE_CARD);
	if (status == SCARD_S_SUCCESS) {
		printf("Disconnected from card\n");
	} else {
		printf("Card deconnection error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}


int main() {
	establishContext();
	listReaders();
	connectToCard();

	disconnectFromCard();
	freeListReader();
	releaseContext();
	return 0;
}
```

We have here two more global variables:

* `connectionHandler`: used to communicate with the tag.
* `activeProtocol`: is the established protocol to this connection. We need to keep track of the protocol because we let WinSCard choose which protocol to use, as we gave `SCARD_PROTOCOL_T0 | SCARD_PROTOCOL_T1` as the fourth argument of the `SCardConnect` function. You can specify which protocol to use by giving only `SCARD_PROTOCOL_T0` for example. The protocol will be required when we want to send a custom command the the tag.

Like before, the connection must be closed before the end of the program.

Now that we have a connection to the tag, we can get some information from it, like its [ATR](https://en.wikipedia.org/wiki/Answer_to_reset). The ATR can be used to identify the type of card. A complete list is available [here](https://www.eftlab.com/knowledge-base/171-atr-list-full/).

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;
LPSTR reader = NULL;
SCARDHANDLE connectionHandler;
DWORD activeProtocol;

void establishContext() {}
void releaseContext() {}
void listReaders() {}
void freeListReader() {}
void connectToCard() {}
void disconnectFromCard() {}

void getCardInformation() {
	BYTE ATR[MAX_ATR_SIZE] = "";
	DWORD ATRLength = sizeof(ATR);
	char readerName[MAX_READERNAME] = "";
	DWORD readerLength = sizeof(readerName);
	DWORD readerState;
	DWORD readerProtocol;
	
	LONG status = SCardStatus(connectionHandler, readerName, &readerLength, &readerState, &readerProtocol, ATR, &ATRLength);
	if (status == SCARD_S_SUCCESS) {
		printf("\n");
		printf("Name of the reader: %s\n", readerName);
		printf("ATR: ");
		for (int i=0; i<ATRLength; i++) {
			printf("%02X ", ATR[i]);
		}
		printf("\n\n");
	} else {
		printf("Get card information error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}

int main() {
	establishContext();
	listReaders();
	connectToCard();
	
	getCardInformation();
	
	disconnectFromCard();
	freeListReader();
	releaseContext();
	return 0;
}
```

Finally, we can send a custom command to the reader. Going back to the [ACR122U webpage](https://www.acs.com.hk/en/products/3/acr122u-usb-nfc-reader/), you can download the [API Driver Manual of ACR122U NFC Contactless Smart Card Reader](https://www.acs.com.hk/download-manual/419/API-ACR122U-2.04.pdf). This document presents a list of commands we can send to the reader. For example, let's retrieve the firmware version of the reader (on page 24). The associated command is `FFh 00h 48h 00h 00h`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;
LPSTR reader = NULL;
SCARDHANDLE connectionHandler;
DWORD activeProtocol;

void establishContext() {}
void releaseContext() {}
void listReaders() {}
void freeListReader() {}
void connectToCard() {}
void getCardInformation() {}

void sendCommand(uint8_t command[], unsigned short commandLength) {
	const SCARD_IO_REQUEST *pioSendPci;
	SCARD_IO_REQUEST pioRecvPci;
	uint8_t response[300];
	unsigned long responseLength = sizeof(response);
	
	switch(activeProtocol) {
		case SCARD_PROTOCOL_T0:
			pioSendPci = SCARD_PCI_T0;
			break;
		case SCARD_PROTOCOL_T1:
			pioSendPci = SCARD_PCI_T1;
			break;
		default:
			printf("Protocol not found\n");
			exit(1);
	}
	
	LONG status = SCardTransmit(connectionHandler, pioSendPci, command, commandLength, &pioRecvPci, response, &responseLength);
	if (status == SCARD_S_SUCCESS) {
		printf("Command sent: \n");
		for (int i=0; i<commandLength; i++) {
			printf("%02X ", command[i]);
		}
		printf("\nResponse: \n");
		for (int i=0; i<responseLength; i++) {
			printf("%02X ", response[i]);
		}
		printf("\n\n");
	} else {
		printf("Send command error: %s\n", pcsc_stringify_error(status));
		exit(1);
	}
}

int main() {
	establishContext();
	listReaders();
	connectToCard();
	
	getCardInformation();
	
	printf("Firmware command:\n");
	uint8_t firmwareCommand[] = { 0xFF, 0x00, 0x48, 0x00, 0x00 };
	unsigned short firmwareCommandLength = sizeof(firmwareCommand);
	sendCommand(firmwareCommand, firmwareCommandLength);
	
	disconnectFromCard();
	freeListReader();
	releaseContext();
	
	return 0;
}
```

The `switch` statement is needed here because the established protocol to the connection will be determined on runtime. If we had only used `SCARD_PROTOCOL_T0` in the `SCardConnect` function, we could have directly written `pioSendPci = SCARD_PCI_T0;` instead of the switch statement.

We have now a running application that can be used to send custom commands to the reader and the tag. We will apply to send and retrieve data from the MIFARE Ultralight and the MIFARE Classic 1k. Let's start with the MIFARE Ultralight.

The MIFARE Ultralight is composed of 16 pages with 4 bytes each. The user memory starts from the fifth page and the data is unprotected (we can access without authentication). You can find more information on the [data sheet](https://www.nxp.com/docs/en/data-sheet/MF0ICU1.pdf) of the tag.

On page 16 and 17 of the [API Driver Manual of ACR122U](https://www.acs.com.hk/download-manual/419/API-ACR122U-2.04.pdf), we can see the read and write commands. Let's write a function that read 16 bytes from the fifth page (page `04h`) and write 4 bytes to the fifth page:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;
LPSTR reader = NULL;
SCARDHANDLE connectionHandler;
DWORD activeProtocol;

void establishContext() {}
void releaseContext() {}
void listReaders() {}
void freeListReader() {}
void connectToCard() {}
void disconnectFromCard() {}
void getCardInformation() {}
void sendCommand(uint8_t command[], unsigned short commandLength) {}

void mifareUltralight() {
	printf("### MIFARE Ultralight ###\n");
	uint8_t pageNumber = 0x04;
	
	// Read 4 blocks (16 bytes) starting from pageNumber
	uint8_t readCommand[] = { 0xFF, 0xB0, 0x00, pageNumber, 0x10 };
	unsigned short readCommandLength = sizeof(readCommand);
	sendCommand(readCommand, readCommandLength);
	
	// Write 1 block (4 bytes) to pageNumber
	uint8_t data[] = { 0x00, 0x01, 0x02, 0x03 };
	uint8_t writeCommand[9] = { 0xFF, 0xD6, 0x00, pageNumber, 0x04 };
	unsigned short writeCommandLength = sizeof(writeCommand);
	for (int i=0; i<4; i++) {
		writeCommand[i+5] = data[i];
	}
	sendCommand(writeCommand, writeCommandLength);
}

int main() {
	establishContext();
	listReaders();
	connectToCard();
	
	getCardInformation();
	
	printf("Firmware command:\n");
	uint8_t firmwareCommand[] = { 0xFF, 0x00, 0x48, 0x00, 0x00 };
	unsigned short firmwareCommandLength = sizeof(firmwareCommand);
	sendCommand(firmwareCommand, firmwareCommandLength);
	
	mifareUltralight();
	
	disconnectFromCard();
	freeListReader();
	releaseContext();
	return 0;
}
```

Finally, let's do the same but for the MIFARE Classic 1k. This tag consists of 16 sectors of 4 blocks, each block containing 16 bytes. The data is protected by a 6-bytes key, the default key being `FFh FFh FFh FFh FFh FFh`. You can find more information on the [data sheet](https://www.nxp.com/docs/en/data-sheet/MF1S50YYX_V1.pdf) of the tag.

With the MIFARE Classic 1k, we need do authenticate before accessing a sector, which is indicated on page 12 and 13 of the [API Driver Manual of ACR122U](https://www.acs.com.hk/download-manual/419/API-ACR122U-2.04.pdf).

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <winscard.h>

SCARDCONTEXT applicationContext;
LPSTR reader = NULL;
SCARDHANDLE connectionHandler;
DWORD activeProtocol;

void establishContext() {}
void releaseContext() {}
void listReaders() {}
void freeListReader() {}
void connectToCard() {}
void disconnectFromCard() {}
void getCardInformation() {}
void sendCommand(uint8_t command[], unsigned short commandLength) {}
void mifareUltralight() {}

void mifareClassic() {
	printf("### MIFARE Classic ###\n");
	uint8_t blockNumber = 0x04;
	
	// Load Authentication Keys
	uint8_t key[6] = { 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF };
	uint8_t authenticationKeysCommand[11] = { 0xFF, 0x82, 0x00, 0x00, 0x06 };
	unsigned short authenticationKeysCommandLength = sizeof(authenticationKeysCommand);
	for (int i=0; i<6; i++) {
		authenticationKeysCommand[i+5] = key[i];
	}
	sendCommand(authenticationKeysCommand, authenticationKeysCommandLength);
	
	// Authenticate
	uint8_t authenticateCommand[] = { 0xFF, 0x86, 0x00, 0x00, 0x05, 0x01, 0x00, blockNumber, 0x60, 0x00 };
	unsigned short authenticateCommandLength = sizeof(authenticateCommand);
	sendCommand(authenticateCommand, authenticateCommandLength);
	
	// Read 1 block (16 bytes) at blockNumber
	uint8_t readCommand[] = { 0xFF, 0xB0, 0x00, blockNumber, 0x10 };
	unsigned short readCommandLength = sizeof(readCommand);
	sendCommand(readCommand, readCommandLength);
	
	// Write 1 block (16 bytes) at blockNumber
	uint8_t data[] = { 0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F };
	uint8_t writeCommand[21] = { 0xFF, 0xD6, 0x00, blockNumber, 0x10 };
	unsigned short writeCommandLength = sizeof(writeCommand);
	for (int i=0; i<16; i++) {
		writeCommand[i+5] = data[i];
	}
	sendCommand(writeCommand, writeCommandLength);
}

int main() {
	establishContext();
	listReaders();
	connectToCard();
	
	getCardInformation();
	
	printf("Firmware command:\n");
	uint8_t firmwareCommand[] = { 0xFF, 0x00, 0x48, 0x00, 0x00 };
	unsigned short firmwareCommandLength = sizeof(firmwareCommand);
	sendCommand(firmwareCommand, firmwareCommandLength);
	
	//mifareUltralight();
	mifareClassic();
	
	disconnectFromCard();
	freeListReader();
	releaseContext();
	return 0;
}
```

## 5. Conclusion

You know now how to send and retrieve data from the MIFARE Ultralight and Classic 1k with the ACR122U. This should be enough for most of the applications. Otherwise, you can take a look at:

* The [PC/SC Lite API (WinSCard)](https://pcsclite.apdu.fr/api/group__API.html) documentation.
* The [API Driver Manual of ACR122U NFC Contactless Smart Card Reader](https://www.acs.com.hk/download-manual/419/API-ACR122U-2.04.pdf).
* The data sheets of your own tags.
