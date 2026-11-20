# Choosing the SMP buffer size

## Introduction

This page helps you choose the SMP buffer size for your product: it shows how the size affects
upload time, RAM and how long the serial port is busy.

During a firmware update, a PC (the SMP client) sends the new firmware file to the device (the SMP
server) in many requests. The device must receive each request whole before it can act on it, so it
keeps it in a receive buffer. The size of that buffer sets the largest request the device can
accept, and therefore how many requests the update takes.

The size is chosen when the firmware is built, in Kconfig. It does not change while the device runs.
The PC adapts to it.

- A **larger** buffer means fewer requests, so fewer waits for an answer and a slightly faster
  update. It costs RAM, and each message keeps the serial port busy for longer.
- A **smaller** buffer saves RAM, and each message keeps the serial port busy for a shorter time,
  but the update needs more requests and takes longer in total.

Facts marked *(source)* were checked in Zephyr v4.4.2, the version in `west.yml`. Everything else is
marked as an assumption.

## What you configure

| Kconfig | Default | Effect |
|---|---|---|
| `CONFIG_MCUMGR_TRANSPORT_NETBUF_SIZE` | 384 | `buf_size`, the size of one SMP buffer. The largest message the device can receive is `buf_size − 4` bytes. |
| `CONFIG_MCUMGR_TRANSPORT_NETBUF_COUNT` | 4 | The number of buffers. They are shared by requests and answers. |
| `CONFIG_MCUMGR_GRP_OS_MCUMGR_PARAMS` | off | Lets the PC ask the device for `buf_size` before an upload. |

*(source: `subsys/mgmt/mcumgr/transport/Kconfig`, `subsys/mgmt/mcumgr/transport/src/smp.c`,
`subsys/mgmt/mcumgr/grp/os_mgmt/src/os_mgmt.c`)*

The buffers take about `buf_count × buf_size` bytes of RAM, plus a few bytes of bookkeeping per
buffer.

## What the size affects

The examples transfer a firmware file of **205 000 bytes** over a serial port at 8N1. `p` is the
device's time per message before it answers (decoding, checking and writing to flash). It depends on
the chip and has to be measured; 5 ms is an assumption.

### Upload time for the file

| `buf_size` | 115 200 baud, `p` = 0 ms | 115 200 baud, `p` = 5 ms | 921 600 baud, `p` = 0 ms | 921 600 baud, `p` = 5 ms |
|---|---|---|---|---|
| 256 | 30.2 s | 34.7 s | 3.8 s | 8.3 s |
| 384 | 28.1 s | 31.0 s | 3.5 s | 6.4 s |
| 1 024 | 25.7 s | 26.8 s | 3.2 s | 4.3 s |
| 2 048 | 25.0 s | 25.5 s | 3.1 s | 3.6 s |

Without `CONFIG_IMG_ERASE_PROGRESSIVELY`, the device erases the whole slot when it receives the first
request, so the first answer comes only after the full erase time. *(source: `subsys/dfu/Kconfig`)*
That time is not included.

### How long one message keeps the serial port busy

For the largest message of each buffer size:

| `buf_size` | Bytes on the port | 115 200 baud | 921 600 baud |
|---|---|---|---|
| 256 | 353 | 31 ms | 3.8 ms |
| 384 | 527 | 46 ms | 5.7 ms |
| 1 024 | 1 404 | 122 ms | 15 ms |
| 2 048 | 2 801 | 243 ms | 30 ms |

### RAM

| `buf_size` | RAM with 4 buffers |
|---|---|
| 256 | 1 KB |
| 384 | 1.5 KB |
| 1 024 | 4 KB |
| 2 048 | 8 KB |

### What the examples show

- At 115 200 baud, the baud rate sets most of the time. Going from 256 to 2 048 bytes saves 5 to 9 s
  of 30 to 35 s.
- At 921 600 baud, the time per message `p` matters more. With `p` = 5 ms, 256 bytes takes more than
  twice as long as 2 048 bytes.
- The RAM cost grows with `buf_count × buf_size`: with 4 buffers, 4 × 256 = 1 KB versus
  4 × 2 048 = 8 KB.
- The time one message keeps the serial port busy grows with the buffer size: at 115 200 baud,
  31 ms for 256 bytes versus 243 ms for 2 048 bytes.

## The PC's SMP client

The PC's SMP client must keep each message to `buf_size − 4` bytes. A larger message is dropped: the
device sends no answer, and the upload fails after the client's timeout. *(source:
`subsys/mgmt/mcumgr/transport/src/serial_util.c`)*

- Clients that ask the device for `buf_size` adapt automatically. This needs
  `CONFIG_MCUMGR_GRP_OS_MCUMGR_PARAMS`. Example: smpclient 7.4.0 (Python) asks by default and keeps
  messages to `buf_size − 4`.
- Clients that do not ask must be configured to match. Example: mcumgr-client (Rust) uses its
  `--mtu` option instead, and its default of 512 fits a `buf_size` of at least 382. For a smaller
  buffer, `--mtu` must be lowered.

Different clients encode requests slightly differently. A client that sends extra fields has a few
bytes less room per message, which changes the upload time by a few percent at most.

## Not verified

- The time per message `p`. It has to be measured on your product under normal load. It depends on
  the chip's flash, on Kconfig such as `CONFIG_IMG_ERASE_PROGRESSIVELY`, and on how often other
  threads delay mcumgr's work queue thread (priority `CONFIG_MCUMGR_TRANSPORT_WORKQUEUE_THREAD_PRIO`,
  default 3). See [issue #4](https://github.com/sigmundrulle/zephyr-fw-update/issues/4) for a tool
  to measure it.
- How the module receives SMP on a serial port shared with text commands. The `buf_size − 4` limit
  assumes the module uses Zephyr's serial receive code, and the time one message keeps the port busy
  matters only if something else can use the port during an update. See
  [issue #5](https://github.com/sigmundrulle/zephyr-fw-update/issues/5).

## Appendix: how the numbers are calculated

### Terms

| Term | Meaning |
|---|---|
| File | The signed firmware file that is transferred: MCUboot header, code and TLV. |
| Message | One SMP request or answer: an 8-byte header followed by a CBOR payload. |
| Frame | One piece of a message on the serial port: 2 start bytes, base64 text and a newline, at most 127 bytes. |
| `buf_size` | The size of one SMP buffer on the device, in bytes. |

A file is sent as one or more messages. A message is sent as one or more frames. A frame always
belongs to one message.

### The buffer on the device

- On the serial port, the receive buffer also holds the 2-byte length field and the 2-byte checksum
  of the message. **A message can therefore be at most `buf_size − 4` bytes.** *(source:
  `subsys/mgmt/mcumgr/transport/src/serial_util.c`)*
- A message that does not fit is dropped: the device sends no answer, and the PC waits until it
  times out. *(source: as above, the base64 decoding fails when the buffer is full)*

### Step 1: the PC learns the size

If the device has `CONFIG_MCUMGR_GRP_OS_MCUMGR_PARAMS` enabled, the PC can ask for the buffer size
before it starts:

```
PC → device:  MCUmgr parameters   { }
device → PC:                      { "buf_size": 384, "buf_count": 4 }
```

The device answers with `CONFIG_MCUMGR_TRANSPORT_NETBUF_SIZE` and
`CONFIG_MCUMGR_TRANSPORT_NETBUF_COUNT`. *(source: `subsys/mgmt/mcumgr/grp/os_mgmt/src/os_mgmt.c`)*
If the device does not support the command, or the PC does not ask, the size has to be set by hand
in the PC's SMP client.

Zephyr's documentation describes `buf_size` as the SMP header plus the CBOR payload. On the serial
port the largest message is 4 bytes smaller (see above).

### Step 2: the PC sends the file

The PC sends image upload requests (group 1, command 1). Each request carries the next part of the
file. The device writes it to the second slot and answers with the offset it expects next.
*(source: `subsys/mgmt/mcumgr/grp/img_mgmt/src/img_mgmt.c`)*

```
request:  { "off": 0, "len": <file size>, "sha": <32 bytes>, "data": <first part> }
answer:   { "off": <bytes received so far> }
request:  { "off": <the same number>, "data": <next part> }
answer:   { "off": … }
…
```

- `len` and `sha` are only in the first request. `sha`, the SHA-256 of the whole file, is optional.
- The PC waits for each answer before it sends the next request.
- The TLV at the end of the file is not handled separately. To SMP it is just the last bytes of the
  file, and it may end up split across two requests.

### Step 3: each message becomes frames

Before a message is sent on the serial port:

1. A 2-byte length field is put in front of it and a 2-byte checksum (CRC16) after it.
2. The result is base64-encoded: every 3 bytes become 4 characters.
3. It is cut into frames of at most 127 bytes. Each frame has 2 start bytes and a newline, which
   leaves 124 characters, or 93 bytes before encoding.

```
frame 1:      0x06 0x09  base64(length field + start of message)  \n
next frames:  0x04 0x14  base64(continuation)                      \n
last frame:   0x04 0x14  base64(end of message + checksum)         \n
```

*(source: `include/zephyr/mgmt/mcumgr/transport/serial.h` and `serial_util.c`)*

Zephyr's own sender fills every frame except the last one. The receiver accepts frames of any size
and collects them until it has the number of bytes given in the length field. *(source:
`serial_util.c`)* The calculations assume that the PC's SMP client cuts frames the same way.

### How much of the file fits in one message

A message has room for the file data that remains after the header and the CBOR field names and
values:

```
data = (buf_size − 4) − 8 − CBOR overhead
```

The CBOR overhead depends on how many bytes each number needs, so it changes a little during the
transfer. With the fields shown in Step 2:

| Request | CBOR overhead (bytes) |
|---|---|
| First (`off` 0, `len` ≥ 65 536, 32-byte `sha`) | 60 for up to 255 bytes of data, 61 for more |
| `off` below 65 536 | 14 to 16 |
| `off` 65 536 or above | 17 to 18 |

Assumption: the PC encodes CBOR in the shortest form and sends exactly these fields. A client that
also sends `image` or `upgrade`, or encodes differently, has a few bytes less room per message.

### Frames and bytes for one message

For a message of *n* bytes:

```
bytes before encoding:  r = n + 4                       (length field + checksum)
frames:                 f = ⌈ r / 93 ⌉
bytes on the port:      b = 3·f + 124·(f − 1) + 4·⌈ (r − 93·(f − 1)) / 3 ⌉
```

The first term counts the start bytes and newlines, the second the full frames, and the third the
last frame's base64 text.

| Message (bytes) | Frames | Bytes on the port |
|---|---|---|
| 18 (an answer `{ "off": … }`) | 1 | 35 |
| 252 | 3 | 353 |
| 380 | 5 | 527 |
| 1 020 | 12 | 1 404 |
| 2 044 | 23 | 2 801 |

### Transfer time

A serial port at 8N1 sends 10 bits per byte, so it carries `baud / 10` bytes per second.

| | 115 200 baud | 921 600 baud |
|---|---|---|
| Bytes per second | 11 520 | 92 160 |
| One full frame (127 bytes) | 11.0 ms | 1.4 ms |

The time for a whole file is:

```
T = (sum of b for all requests + sum of b for all answers) / (baud / 10)  +  messages × p
```

### The example file

The examples use a file of 205 000 bytes: 204 800 bytes of image followed by a 200-byte TLV at bytes
204 800 – 204 999.

| `buf_size` | Largest message | Data per message | Messages | Frames (PC → device) |
|---|---|---|---|---|
| 256 | 252 | 184 – 230 | 901 | 2 703 |
| 384 (default) | 380 | 311 – 356 | 579 | 2 891 |
| 1 024 | 1 020 | 951 – 996 | 207 | 2 474 |
| 2 048 | 2 044 | 1 975 – 2 020 | 102 | 2 336 |

How the file is split with `buf_size` 256:

| Messages | Data per message | File bytes |
|---|---|---|
| 1 (the first, with `len` and `sha`) | 184 | 0 – 183 |
| 1 | 230 | 184 – 413 |
| 285 | 229 | 414 – 65 678 |
| 613 | 227 | 65 679 – 204 829 |
| 1 (the last) | 170 | 204 830 – 204 999 |

The TLV is split: its first 30 bytes are in message 900 and the remaining 170 bytes in message 901.

Bytes on the serial port:

| `buf_size` | Requests (bytes) | Answers (bytes) | Requests ÷ file size |
|---|---|---|---|
| 256 | 317 977 | 30 391 | 1.55 |
| 384 | 304 733 | 19 529 | 1.49 |
| 1 024 | 289 470 | 6 985 | 1.41 |
| 2 048 | 284 528 | 3 442 | 1.39 |
