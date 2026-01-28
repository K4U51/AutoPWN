# Theoretical Overview: Open-Source Subaru ECU Flashing via Serial Transport
# Including Procedural Workflow and Conceptual Use of Hak5 Pager for Baud Control

-----------------------------------------------------------------------

PURPOSE & SCOPE

This document describes a theoretical, open-source method for reading and
writing Subaru ECUs using serial diagnostics and documented MCU behavior,
without proprietary flashing software or OEM-locked tooling.

It demonstrates:
- The ECU exposes standardized diagnostic and programming services
- Flash control is obtained through sequence and state, not exploits
- Commodity serial hardware (and programmable UART platforms) is sufficient

-----------------------------------------------------------------------

MANDATORY DISCLAIMER

DO NOT ATTEMPT THIS ON A VEHICLE OR ECU YOU CANNOT AFFORD TO LOSE

- Partial or improper flashing will brick the ECU
- Erasing flash without immediate reprogramming will brick the ECU
- Recovery requires boot-mode access and soldering and is not guaranteed
- Provided scripts and methods are partially tested
- Use is entirely at your own risk

-----------------------------------------------------------------------

TARGET ECU (REFERENCE PLATFORM)

- 2006 EDM Subaru Impreza STI
- Drive-by-wire ECU
- Renesas SH-7058 MCU
- SH-2e instruction set
- Subaru SSM2 protocol (diagnostic layer)

-----------------------------------------------------------------------

PROTOCOL & FIRMWARE CONTEXT

- ROM memory map starts at 0x00000000
- ROM images can be loaded directly into Ghidra or IDA Pro
- CPU manuals define serial interfaces, I/O registers, and flash control

SSM2 is closely related to:
- KWP-2000
- Early UDS

Negative responses follow standard 0x7F behavior.

=======================================================================
HOW-TO: ECU CONTROL & FLASHING WORKFLOW
(Procedural Reference)
=======================================================================

-----------------------------------------------------------------------
1. ECU TEST / PROGRAMMING MODE
-----------------------------------------------------------------------

- Connect the test-mode connector in the vehicle
- Not present on all cars
- Some ECUs route this to the diagnostic port
- Can be permanently grounded or placed on a switch

This places the ECU into a diagnostic-permissive state.

-----------------------------------------------------------------------
2. DIAGNOSTIC CONNECTION
-----------------------------------------------------------------------

- Use any K-Line serial cable
- Often sold as "VAG-KKL"
- FTDI-based cables (FT232R) are known to work

Initial serial configuration:
- 4800 baud
- 8 data bits
- No parity
- 1 stop bit

-----------------------------------------------------------------------
3. PACKET FRAMING
-----------------------------------------------------------------------

Tester -> ECU:
0x80 0xF0 0x10 [length] [data bytes...] [checksum]

ECU -> Tester:
0x80 0x10 0xF0 ...

Checksum = sum of all prior bytes, masked to 0xFF

-----------------------------------------------------------------------
4. DIAGNOSTIC INITIALIZATION SEQUENCE
-----------------------------------------------------------------------

Send init command:
0xBF

Example packet:
0x80 0xF0 0x10 0x01 0xBF 0x40

ECU replies with identification and state data.

Then send:
0x81

Then send:
0x83 0x00

-----------------------------------------------------------------------
5. SECURITY AUTHENTICATION
-----------------------------------------------------------------------

Request seed:
0x27 0x01

ECU replies:
0x67 0x02 [4-byte seed]

Compute key from seed (algorithm not included).

Send key:
0x27 0x02 [4-byte key]

ECU replies:
0x67 (success)

-----------------------------------------------------------------------
6. ENTER PROGRAMMING SESSION
-----------------------------------------------------------------------

Send:
0x10 0x85 0x02

ECU replies:
0x50

-----------------------------------------------------------------------
7. BAUD RATE CHANGE
-----------------------------------------------------------------------

Immediately switch serial speed to:
15625 baud

Wait approximately 300 ms before sending additional packets.

This pause is critical for resynchronization.

Programmable serial devices (theoretically including a Hak5 Pager)
can handle this transition cleanly.

-----------------------------------------------------------------------
8. RAM CODE UPLOAD (KERNEL INJECTION)
-----------------------------------------------------------------------

Request download using service 0x34.

Assuming:
- RAM address: 0xFFFF3000
- Kernel size: 0x0C28 bytes

Send:
0x34 FF 30 00 04 00 0C 28

ECU replies:
0x74

-----------------------------------------------------------------------
9. DATA TRANSFER (SERVICE 0x36)
-----------------------------------------------------------------------

- Send up to 128 bytes per packet
- Address is 3 bytes (leading 0xFF omitted)
- Data must be encrypted

Example:
0x36 FF 30 00 [128 bytes]

ECU replies:
0x76

Repeat until complete.

-----------------------------------------------------------------------
10. FINAL INTEGRITY TRANSFER (SPECIAL CASE)
-----------------------------------------------------------------------

Send raw data without standard packet framing:

0x53
FF 30 00
0C 28
[entire kernel payload]
[checksum]

ECU may not reply.

-----------------------------------------------------------------------
11. EXECUTE UPLOADED CODE
-----------------------------------------------------------------------

Send:
0x31 0x01 0x01

ECU replies:
0x71

Kernel executes immediately.

-----------------------------------------------------------------------
12. KERNEL COMMUNICATION MODE
-----------------------------------------------------------------------

Immediately switch baud rate to:
62500

Kernel packet format (both directions):

0xBE 0xEF [size_hi] [size_lo] [data bytes...] [checksum]

-----------------------------------------------------------------------
13. KERNEL COMMANDS
-----------------------------------------------------------------------

0x01 - Get kernel version
0x02 - CRC32 over memory range
0x03 - Read memory
0x04 - Battery voltage

0x20 - Enable flash
0x21 - Disable flash
0x22 - Write flash buffer
0x23 - Validate flash buffer
0x24 - Commit flash buffer
0x25 - Erase flash block

Error codes:
0xF0 - Bad data length
0xF1 - Bad data value
0xF2 - Programming failure
0xF3 - Voltage too low
0xF4 - Voltage too high
0xF5 - CRC incorrect
0xFF - Bad command

-----------------------------------------------------------------------
14. FLASH BLOCK LAYOUT (RENESAS SH-7058)
-----------------------------------------------------------------------

Flash erase/write must occur per block:

0x00000  length 0x01000
0x01000  length 0x01000
0x02000  length 0x01000
0x03000  length 0x01000
0x04000  length 0x01000
0x05000  length 0x01000
0x06000  length 0x01000
0x07000  length 0x01000
0x08000  length 0x18000
0x20000  length 0x20000
0x40000  length 0x20000
0x60000  length 0x20000
0x80000  length 0x20000
0xA0000  length 0x20000
0xC0000  length 0x20000
0xE0000  length 0x20000

Partial block writes are not permitted.

-----------------------------------------------------------------------
15. CRC VALIDATION WORKFLOW
-----------------------------------------------------------------------

Request CRC:
0x02 [4-byte address] [4-byte length]

Example:
0x02 00 0E 00 00 00 02 00 00

ECU replies:
0x82 [4-byte CRC32]

-----------------------------------------------------------------------
KEY TAKEAWAY
-----------------------------------------------------------------------

This workflow:
- Uses documented MCU behavior
- Uses industry-standard diagnostics
- Requires no proprietary infrastructure
- Is limited only by procedural accuracy

The ECU is not "hacked" — it is spoken to correctly.
