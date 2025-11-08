# Kroma Intercom PRTP Protocol Unofficial Specification

## Overview

This protocol provides low-latency voice communication over UDP with an in-band control channel. Two audio encoding modes are supported: Speex wideband and G.711 (custom variant). All communication occurs over UDP port 8087.

## Transport

- **Protocol**: UDP
- **Port**: 8087
- **Packet format**: Two modes (Speex or G.711 with PRTP)

## Audio Encoding

### Sample Rate and Format

- **Capture**: 16000 Hz mono, PCM 16-bit
- **Frame size**: 320 samples (20 ms)
- **Playback rate**:
  - Speex: 16000 Hz
  - G.711: 8333 Hz

### Codec Modes

#### 1. Speex Wideband

- **Encoding**: Speex WB, VBR enabled, quality 10
- **Decoding**: Enhancer enabled
- **Wire format**: Raw Speex encoded bytes (no framing header)
- **Frame interval**: 20 ms
- **Usage scenario**: Intended only for direct panel-to-panel links; PRTP control bytes are not injected while Speex is active, so button/LED control traffic will not reach the intercom matrix in this mode (even if ).

#### 2. G.711 Custom Variant

The protocol uses a custom G.711 variant  (referred to as K.711 in AEQ documents) with the following characteristics:

- **Range**: 12-bit linear PCM (0–4095)
- **Companding**: Custom logarithmic curve
  - Segment step sizes: 2 → 6 → 24 → 96 (repeats in both halves)
- **Sign encoding**: Bit 7 of the 8-bit code determines sign
  - Codes 0–127 (bit 7 = 0): positive values, decreasing from 4095 to 64
  - Codes 128–255 (bit 7 = 1): negative values (when negated), increasing from -1 to -4032
- **Decode table**: 256-entry lookup table storing absolute magnitude values
- **Encode**: Nearest-neighbor lookup from decode table

**Note**: This is not standard ITU-T A-law or μ-law. The companding structure follows G.711 principles but uses different quantization levels.

## Packet Format

### Speex Mode

- **Payload**: Raw Speex WB encoded bytes
- **No framing header**
- **Variable length** based on encoder output

### G.711 Mode (Framed with PRTP)

**Datagram size**: 0x10C (268) bytes

#### Header Layout

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 1 | Sync | 0xAA (sync byte) |
| 1 | 1 | Reserved | 0x00 |
| 2 | 1 | Version | 0x01 |
| 3 | 1 | Header Length | 0x0C (12 bytes) |
| 4 | 1 | Sequence/Reserved | 0x00 |
| 5 | 1 | Control Flag | 0x23 (no control) or 0x27 (control present) |
| 6 | 1 | Payload Type | 0x03 (G.711) |
| 7 | 1 | Control Length | N (0–4 control bytes) |
| 8–11 | 4 | Control Data | Up to 4 PRTP control bytes (if N > 0) |
| 12–267 | 256 | Audio Payload | 256 bytes G.711 encoded audio |

## PRTP Control Protocol

PRTP (Proprietary RTP) is an in-band control channel carried within G.711 packets.

### Control Location

- **Header byte 5**: 0x27 indicates control present, 0x23 indicates no control
- **Header byte 7**: N = control byte count (0–4)
- **Header bytes 8–11**: N control bytes
- **Note**: Speex packets do not carry PRTP control

### Framing

Control bytes use byte-stuffing for stream semantics:

- **0xFE**: Escape byte. Literal 0xFE or 0xFF in payload must be prefixed by 0xFE
- **0xFF**: End-of-frame delimiter
- **CRC-8**: Last byte before 0xFF is the CRC (polynomial 0x8D)

#### CRC-8 Algorithm

- Polynomial: 0x8D
- Initialization: First byte of frame
- Per-bit processing: For each bit, if MSB is clear, mix in next bit; then XOR with 0x8D

### Message Types

#### Server-to-Client Messages

| Type | Byte 0 | Description | Expected Response |
|------|--------|-------------|-------------------|
| Ping | 0x50 ('P') | Hello/keepalive | 0x41 (ACK); first ping also sends 0x53 (SYNC) |
| Command | 0x43 ('C') | Command message | 0x41 (ACK) |
| Info | 0x49 ('I') | Status/control information | See subtypes below |

#### Info Message Subtypes (Byte 2 & 0xF0)

| Subtype | Description | Format |
|---------|-------------|--------|
| 0x40 | Key bitmaps | Count in byte 2 & 0x0F; group bytes follow |
| 0x50 | LED green fixed | Count in byte 2 & 0x0F; group bytes follow |
| 0x60 | LED red fixed | Count in byte 2 & 0x0F; group bytes follow |
| 0x70 | LED green blinking | Count in byte 2 & 0x0F; group bytes follow |
| 0x80 | LED red blinking | Count in byte 2 & 0x0F; group bytes follow |
| 0x90 | Label update | Byte 3 = index; length = (byte 2 & 0x0F) - 1; text at byte 4+ |
| 0xD0 | Identity/handshake | ASCII payload at byte 3+ |

**Key/LED bitmaps**: Each group byte represents 8 keys/LEDs (bit 0 = index 0, bit 7 = index 7)

#### Client-to-Server Messages

| Type | Byte 0 | Description | Format |
|------|--------|-------------|--------|
| ACK | 0x41 | Acknowledgment | Single byte + CRC |
| NACK | 0x4E | Negative acknowledgment | Single byte + CRC |
| SYNC | 0x53 | Synchronization | Single byte + CRC |
| Key press/release | 0x49 | Key event | Bytes: 0x49, 0x00, (0x80 if pressed) \| (keyIndex & 0x7F), CRC |

### Example Frame Encoding

To send ACK (0x41):

1. Payload: `[0x41]`
2. Compute CRC-8(0x41) → result byte (e.g., 0x8A)
3. Frame: `[0x41, 0x8A]`
4. Byte-stuff (escape 0xFE/0xFF if present)
5. Append terminator: `[0x41, 0x8A, 0xFF]`
6. Inject into G.711 packet header bytes 8–11

## Jitter Management

- **Buffer**: 20000 sample FIFO
- **Playback rate adjustment**: Adaptive around 16 kHz for Speex to manage jitter
- **Frame cadence**:
  - Speex: 20 ms
  - G.711: ~30.7 ms (256 samples @ 8333 Hz)

## Protocol Notes

- PRTP is proprietary and unrelated to IETF RTP (no sequence numbers, timestamps, or SSRC)
- The 12-byte header is not compatible with standard RTP parsers
- Control messages are only present in G.711 mode; Speex packets carry audio only
- See Appendix A for the complete G.711 decode table

---

## Appendix A: G.711 Decode Table

The custom G.711 decode table maps 8-bit codes (0–255) to 12-bit PCM samples. Sign encoding uses bit 7 of the code.

```
Code    Value  |  Code    Value  |  Code    Value  |  Code    Value
------- ------ | ------- ------ | ------- ------ | ------- ------
  0   4095   |  64   3832   | 128      1   | 192    264
  1   4093   |  65   3816   | 129      3   | 193    280
  2   4091   |  66   3800   | 130      5   | 194    296
  3   4089   |  67   3784   | 131      7   | 195    312
  4   4087   |  68   3768   | 132      9   | 196    328
  5   4085   |  69   3752   | 133     11   | 197    344
  6   4083   |  70   3736   | 134     13   | 198    360
  7   4081   |  71   3720   | 135     15   | 199    376
  8   4079   |  72   3704   | 136     17   | 200    392
  9   4077   |  73   3688   | 137     19   | 201    408
 10   4075   |  74   3672   | 138     21   | 202    424
 11   4073   |  75   3656   | 139     23   | 203    440
 12   4071   |  76   3640   | 140     25   | 204    456
 13   4069   |  77   3624   | 141     27   | 205    472
 14   4067   |  78   3608   | 142     29   | 206    488
 15   4065   |  79   3592   | 143     31   | 207    504
 16   4063   |  80   3568   | 144     33   | 208    528
 17   4061   |  81   3536   | 145     35   | 209    560
 18   4059   |  82   3504   | 146     37   | 210    592
 19   4057   |  83   3472   | 147     39   | 211    624
 20   4055   |  84   3440   | 148     41   | 212    656
 21   4053   |  85   3408   | 149     43   | 213    688
 22   4051   |  86   3376   | 150     45   | 214    720
 23   4049   |  87   3344   | 151     47   | 215    752
 24   4047   |  88   3312   | 152     49   | 216    784
 25   4045   |  89   3280   | 153     51   | 217    816
 26   4043   |  90   3248   | 154     53   | 218    848
 27   4041   |  91   3216   | 155     55   | 219    880
 28   4039   |  92   3184   | 156     57   | 220    912
 29   4037   |  93   3152   | 157     59   | 221    944
 30   4035   |  94   3120   | 158     61   | 222    976
 31   4033   |  95   3088   | 159     63   | 223   1008
 32   4030   |  96   3040   | 160     66   | 224   1056
 33   4026   |  97   2976   | 161     70   | 225   1120
 34   4022   |  98   2912   | 162     74   | 226   1184
 35   4018   |  99   2848   | 163     78   | 227   1248
 36   4014   | 100   2784   | 164     82   | 228   1312
 37   4010   | 101   2720   | 165     86   | 229   1376
 38   4006   | 102   2656   | 166     90   | 230   1440
 39   4002   | 103   2592   | 167     94   | 231   1504
 40   3998   | 104   2528   | 168     98   | 232   1568
 41   3994   | 105   2464   | 169    102   | 233   1632
 42   3990   | 106   2400   | 170    106   | 234   1696
 43   3986   | 107   2336   | 171    110   | 235   1760
 44   3982   | 108   2272   | 172    114   | 236   1824
 45   3978   | 109   2208   | 173    118   | 237   1888
 46   3974   | 110   2144   | 174    122   | 238   1952
 47   3970   | 111   2080   | 175    126   | 239   2016
 48   3964   | 112   1984   | 176    132   | 240   2112
 49   3956   | 113   1856   | 177    140   | 241   2240
 50   3948   | 114   1728   | 178    148   | 242   2368
 51   3940   | 115   1600   | 179    156   | 243   2496
 52   3932   | 116   1472   | 180    164   | 244   2624
 53   3924   | 117   1344   | 181    172   | 245   2752
 54   3916   | 118   1216   | 182    180   | 246   2880
 55   3908   | 119   1088   | 183    188   | 247   3008
 56   3900   | 120    960   | 184    196   | 248   3136
 57   3892   | 121    832   | 185    204   | 249   3264
 58   3884   | 122    704   | 186    212   | 250   3392
 59   3876   | 123    576   | 187    220   | 251   3520
 60   3868   | 124    448   | 188    228   | 252   3648
 61   3860   | 125    320   | 189    236   | 253   3776
 62   3852   | 126    192   | 190    244   | 254   3904
 63   3844   | 127     64   | 191    252   | 255   4032
```

**Usage**:
- **Codes 0–127 (bit 7 = 0)**: Represent positive values
- **Codes 128–255 (bit 7 = 1)**: Represent negative values (negate the table value)
- **Example**: 
  - Code 0 → +4095
  - Code 128 → +1 (stored positive, bit 7=1 indicates sign, so actual value could be -1 depending on implementation)
  - Code 255 → value 4032 (as positive magnitude)

**Encoding**: Use nearest-neighbor search through this table to find the code that produces the closest decoded value to the input PCM sample.

### Mathematical Approximation

The decode table follows a segmented logarithmic pattern that can be approximated with the following formula:

```python
def decode_g711_custom_approx(code):
    """
    Approximation formula for the custom G.711 decode table.
    This provides an approximation without requiring the full 256-entry table.
    """
    # Extract sign bit (bit 7) and absolute code (bits 0-6)
    sign = 1 if (code & 0x80) == 0 else -1
    abs_code = code & 0x7F
    
    # Extract segment (bits 4-6) and quantization (bits 0-3)
    segment = (abs_code >> 4) & 0x07
    quant = abs_code & 0x0F
    
    # Base values for each of the 8 segments
    bases = [4095, 4063, 4030, 3964, 3832, 3568, 3040, 1984]
    
    # Step size per segment:
    # Segments 0-1: step size = 2
    # Segments 2-7: step size = 2^(segment-1)
    if segment < 2:
        step = 2
    else:
        step = 1 << (segment - 1)  # Equivalent to 2^(segment-1)
    
    # Calculate value: base - (quantization * step)
    value = bases[segment] - (quant * step)
    
    # Apply sign
    return sign * value
```

**Segment Structure**:
- Segment 0 (codes 0-15): step=2, range 4095 to 4065
- Segment 1 (codes 16-31): step=2, range 4063 to 4033  
- Segment 2 (codes 32-47): step=4, range 4030 to 3970
- Segment 3 (codes 48-63): step=8, range 3964 to 3844
- Segment 4 (codes 64-79): step=16, range 3832 to 3592
- Segment 5 (codes 80-95): step=32, range 3568 to 3088
- Segment 6 (codes 96-111): step=64, range 3040 to 2080
- Segment 7 (codes 112-127): step=128, range 1984 to 64

This formula produces values very close to the table (typically within a few LSBs) and can be used when the full table lookup is not feasible.
