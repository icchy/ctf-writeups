# Strange (Forensic 150)
I lost my glasses! help me!!

## Analyse PNG
The given png has very large size so it can't be opened (actually, my viewer crashed).

```bash
~$ exiftool strange.png
ExifTool Version Number         : 9.96
File Name                       : strange.png
Directory                       : .
File Size                       : 14 MB
File Modification Date/Time     : 2015:09:12 01:33:42+09:00
File Access Date/Time           : 2015:10:12 09:35:28+09:00
File Inode Change Date/Time     : 2015:10:11 17:15:18+09:00
File Permissions                : rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 344987
Image Height                    : 344987
Bit Depth                       : 1
Color Type                      : Palette
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Palette                         : (Binary data 6 bytes, use -b option to extract)
Image Size                      : 344987x344987
Megapixels                      : 119016.0
```

so let's see the data.

```bash
~$ xxd -a strange.png
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
00000010: 0005 439b 0005 439b 0103 0000 00a5 a512  ..C...C.........
00000020: b100 0000 0650 4c54 4540 00e0 e000 40a3  .....PLTE@....@.
00000030: 7e63 ab00 dbfd f149 4441 5478 daec c101  ~c.....IDATx....
00000040: 0d00 0000 c2a0 f74f 6d0f 0704 0000 0000  .......Om.......
*
006dfbb0: 0000 0000 0000 00bc 1900 0000 ffff faf8  ................
006dfbc0: 47be c2be fe4f 8dfd 798b ffff ece5 feff  G....O..y.......
006dfbd0: 7dfc a0ae feff 1f7b f9fe f97f f8bf 3fff  }......{......?.
006dfbe0: fcf8 cfe7 fef6 c37f ce03 0000 00ff ffec  ................
006dfbf0: c101 0d00 0000 c2a0 f74f 6d10 8100 0000  .........Om.....
[snip]
```
There are long null bytes and extracted data would be huge data, maybe this is the reason why my viewer can't open this picture (I don't have enough memory to open this).

### zlib
The header of image data is IDAT, which is compressed with deflate and appearing immidiately after IDAT.  
Analyse zlib header("\x78\xda") refering [RFC 1950](https://tools.ietf.org/html/rfc1950) , I found the following fact.

 * CMF (0x78)
   * CM (Compression Method: 0x8)
     * using deflate
   * CINFO (Compression INFO: 0x7)
     * window size (maximum)
 * FLG (0xda: 0b11011010)
   * FCHECK (check bits for CMF and FLG: 0b11010)
     * it's none of my business.
   * FDICT (preset dictionary: 0)
     * this data has no preset dictionary for huffman code
   * FLEVEL (compression level: 3)
     * compressor used maximum compression, slowest algorithm

this data is compressed with deflate and no preset dictionary, so I tried to decompress.

```python
In [1]: data = open('./strange.png').read()[0x3b:]

In [2]: import zlib

In [3]: zlib.decompress(data)
Python(45782,0x7fff7e2cd310) malloc: *** mach_vm_map(size=140737488359424) failed (error code=3)
*** error: can't allocate region
*** set a breakpoint in malloc_error_break to debug
---------------------------------------------------------------------------
MemoryError                               Traceback (most recent call last)
<ipython-input-3-f38a88343e01> in <module>()
----> 1 zlib.decompress(data)

MemoryError:
```
The decompressed data is too big to deal with!  
But, are long null bytes between 0x40 and 0x6dfbb0 maningful?
```python
In [4]: zlib.compress("foo"+"\x00"*0x10000+"bar"+"\x00"*0x10000)
Out[4]: 'x\x9c\xed\xce\xb1\r\x00 \x08\x000^\x95\x81\x95\xc4\xff\x07\xf9B\x86\xf6\x82Vw\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xc0\xc8s\x7f\x17\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xd8\xe1\x01@\xdc\x02z'
```
WOW, huffman code seems to compress long null bytes to shorter null bytes. I tried to decompress data stripped long null bytes.

```bash
xxd -a strange.png | cut -d":" -f2- | xxd -r -p > stripped
```

```python
In [1]: import zlib

In [2]: d = zlib.decompress(open('./stripped').read()[0x3b:])
---------------------------------------------------------------------------
error                                     Traceback (most recent call last)
<ipython-input-2-df7c762dd4b4> in <module>()
----> 1 d = zlib.decompress(open('./stripped').read()[0x3b:])

error: Error -3 while decompressing data: incorrect data check
```
This data has incorrect Adler-32 checksum. to ignore this, skip header and give -8 (or -15) to wbits parameter.

```python
In [3]: d = zlib.decompress(open('./stripped').read()[0x3b+2:], -8)

In [4]: len(d)
Out[4]: 714711

In [5]: open('decompressed', 'w').write(d)
```
Now I'm ready to read real data!  
Fortunately, this PNG is 1-bit color so it is easy to convert image. 

```bash
~$ xxd -a decompressed                                                          ⏎
00000000: 0000 0000 0000 0000 0000 0000 0000 0000  ................
*
000070a0: 0000 0000 0000 0000 0000 0000 0000 00f1  ................
000070b0: fc1f 783f 7ffc 7c3f cf38 fffe 3f1e fffd  ..x?..|?.8..?...
000070c0: e3e0 7e7f fffc 3f1f 8f9f fc0f f7e7 f3e3  ..~...?.........
000070d0: fcf3 8f87 c3fc cf00 0000 0000 0000 0000  ................
000070e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
*
00011920: 0000 0000 f1f9 8f73 1eff f139 9fce 627f  .......s...9..b.
00011930: fc9e 4eff fd89 effe 7fff f99e 4e27 3ffd  ..N.........N'?.
00011940: fff7 e7f3 c9fc e727 3399 fce7 0000 0000  .......'3.......
00011950: 0000 0000 0000 0000 0000 0000 0000 0000  ................
*
0001c190: 0000 0000 0000 0000 00f5 f3ef 67de fff3  ............g...
0001c1a0: 9bcf 8e67 3ff9 cce6 fffd 9ccf fc7f fffb  ...g?...........
0001c1b0: cce6 733f f9ff f7c7 e39c f8e6 7379 bcf8  ..s?........sy..
0001c1c0: f700 0000 0000 0000 0000 0000 0000 0000  ................
0001c1d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
*
00026a00: 0000 0000 0000 0000 0000 0000 0000 e4f3  ................
00026a10: ef67 def8 f39b cf0c 2738 3bfd e68f 859c  .g......'8;.....
00026a20: cff8 7878 3bed e6fa 1839 fe17 87c3 bff0  ..xx;....9......
00026a30: c2f3 79be f0f7 0000 0000 0000 0000 0000  ..y.............
00026a40: 0000 0000 0000 0000 0000 0000 0000 0000  ................
[snip]
```
This shows that image width is 43125 (ex. 0x26a0e - 0x1c199), and I wrote following program converting bit to image.

```python
from PIL import Image

rawdata = open('./decompressed').read()
offset = 0x70a0
W = 43125

data = []
idx = offset
while idx < 0xa4f80:
    data.append(bin(int(rawdata[idx:idx+0x40].encode('hex'), 16))[2:])
    idx += W

img = Image.new("1", (0x40*8, len(data)))
pix = img.load()

for h in range(len(data)):
    for w in range(len(data[h])):
        if data[h][w] == "0":
            pix[w, h] = 0
        else:
            pix[w, h] = 255

img.save("./flag.png")
```
Finally I got flag!


```
ASIS{e834f8a60bd854ca902fa5d4464f0394}
```