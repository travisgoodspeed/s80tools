## AT commands and replies


Reply failure codes: fail, busy, nochannel, kill, cksum


| AT reply    | Separator | Data | Description |
| ------------|-----------|------|-------------|
| +DMOMES     | =         | unicode data | received chat message |
| +DMOSETGROUP| :         | status | set group response |
| +DMOMES     | :         | status | message send response |
| +DMOVERQ    | :         | action | get AT verq response |
| +DMOCONNECT | :         | status | connect response? (has no corresponding AT request) |
| +DMOAUTOPOWCONTR | :    | status | set auto power response |
| +DMOSETVOLUME | :       | status | set volume response |
| +DMOSETVOX  | :         | status | set vox response |
| DMOSETMIC   | :         | status | set mic response |
| +DMOREADMES |           |        | unknown, is only referenced once in the context of "if not this message" |


| AT Command  | Separator | Data | Description | Binary | Binary desc |
| ------------|-----------|------|-------------|--------|-------------|
| AT+DMOSETVOLUME | =     | volume | set volume | 68 02 01 01 00 00 00 01 + volume + 10 |
| AT+DMOAUTOPOWCONTR | =  | 0 or 1 | set autopower | 68 0C 01 01 00 00 00 03 + aSwitch + 1E 02 10 | 01 for auto power, FF for disable|
| AT+DMOVERQ  |           | get verq | 68 25 01 01 00 00 00 01 01 10 | |
| AT+DMOSETGROUP | =      | gwb, tfv, rfv, cxcss, sq, cxcss, flag | set group | 68 35 01 01 00 00 + dataLength + data + 10| |
| AT+DMOMES   | =         | (length < 10 ? "0" + length : length) + content | send message | | |


## Binary commands

|  B | Command                               | Data      | Description  |
|----|--------------------------------------|-----------|--------------|
| 02 | 68 02 01 01 00 00 00 01 + volume + 10 | volume | set volume      |
| 0C | 68 0C 01 01 00 00 00 03 + aSwitch + 1E 02 10 | 01 for auto power, FF for disable | set auto power |
| 25 | 68 25 01 01 00 00 00 01 01 10         |           | get verq     |
| 01 | 68 35 01 01 00 00 + dataLength + data + 10|       | set group    |
| 07 | 68 07 01 01 00 00 + dataLength + data + 10|       | send message |
| 06 | 68 06 01 + send + csum + 00 04 + callType + callNumber + "10" | send set to 0 to start, FF to stop | send sound |
| 88 | 68 88 01 01 00 00 00 01 00 10         |           | reset modem  |
| 11 | 68 11 01 01 00 00 00 01 01 10         |           | record message |
| 37 | 68 37 01 01 00 00 00 01 01 10         |           | check ac group |
| 38 | 68 38 01 01 00 00 00 01 01 10         |           | check dc group |


Byte two is the command
When binary, bytes 5 and 6 are the checksum. The checksum is generated over a packet whith an empty checksum.
Bytes two and three are the command
six and seven are the status register

### Packet structure
|  0 |  1 |  2 |  3 |  4 |  5 |  6 |  7  |  n |
|----|----|----|----|----|----|----|----|----|
| 68 | CMD | 01 | SR | SUM | SUM | 00 | ?? | 10 |


Generic SR values
```
+----+-------------+
| SR | Description |
+----+-------------+
| 00 | success     |
| 01 | busy        |
| 02 | nochannel   |
| 07 | kill        |
| 09 | cksum       |
| 70 | getmessage  |
| 71 | putmessage  |
| 7E | msgtimeout  |
| ?? | fail        |
+----+-------------+
```

SR for command 06
```
+----+-------------+
| SR | Description |
+----+-------------+
| 61 | sendstart   |
| 62 | sendstop    |
| 60 | recstart    |
| 6F | recstop     |
+----+-------------+
```

SR for command 07
```
+----+------------------------+
| SR |      Description       |
+----+------------------------+
| 70 | record message (audio) |
| 71 | chat message received  |
| ?? | no message             |
+----+------------------------+
```


SR for command 11

Unicode message received, recording will be stopped.
```
+----+-------------+
| SR | Description |
+----+-------------+
| 00 | success     |
| ?? | no message  |
+----+-------------+
```

SR for command 88

Reset success/fail
```
+----+-------------+
| SR | Description |
+----+-------------+
| 00 | success     |
| ?? | reset fail  |
+----+-------------+
```

### Checksumming algorithm
```
sum = (b[i+0] << 8 | b[i+1]) & 65535
if uneven length: sum += (b[i] & 255) << 8
while ((sum >> 16) != 0) {
    sum = (sum & 65535) + (sum >> 16);
}
return sum ^ 65535
```

### Update procedure

1. Send empty message
2. Transfer /system/etc/talkie/update.bin via ymodem protocol
3. Wait for RESTART as response, any other response means failure, try again up to 3x
