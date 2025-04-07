+++
title = 'Base64'
date = 2025-04-07T16:30:25+08:00
draft = false
tags = ['golang', 'base64']
+++

## 什么是 Base64

Base64 是一种基于 64 个可打印字符来表示二进制数据的表示方法。

HTTP 的全称是 HyperText Transfer Protocol（超文本传输协议），最初设计的目的确实是：

- 传输 超文本（HTML）

但实际上作为应用层协议，HTTP 可以传输任意类型的数据。

- HTTP Header 不能直接放图片字节进去，只能用 ASCII 字符（Base64 就是 ASCII-safe）
- Email 内容（MIME 协议）SMTP 协议历史原因只支持 7-bit ASCII，不能发图片，怎么办？Base64！
- JSON / XML 这些格式都是文本格式，不能放二进制文件进去，必须转 Base64 编码后作为字符串存放

Base64要求把每三个8Bit的字节转换为四个6Bit的字节(`3*8 = 4*6 = 24`)，然后把6Bit再添两位高位0，组成四个8Bit的字节，也就是说，转换后的字符串理论上将要比原来的长1/3。

例如：

```
11111111,11111111,11111111 -> 00111111,00111111,00111111,00111111
```

知道了 byte 的表示，但是还需要一张表，作为 base64 的编码表。

base64 中的 64 表示 64 个字符：

```go
// StdEncoding is the standard base64 encoding, as defined in RFC 4648.
var StdEncoding = NewEncoding("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/")

// URLEncoding is the alternate base64 encoding defined in RFC 4648.
// It is typically used in URLs and file names.
var URLEncoding = NewEncoding("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_")

```


## Golang 中的 Base64 编码

在 Go 语言中，可以使用 `encoding/base64` 包来实现 Base64 编码和解码。


```go
// An Encoding is a radix 64 encoding/decoding scheme, defined by a
// 64-character alphabet. The most common encoding is the "base64"
// encoding defined in RFC 4648 and used in MIME (RFC 2045) and PEM
// (RFC 1421).  RFC 4648 also defines an alternate encoding, which is
// the standard encoding with - and _ substituted for + and /.
type Encoding struct {
	encode    [64]byte   // mapping of symbol index to symbol byte value
	decodeMap [256]uint8 // mapping of symbol byte value to symbol index
	padChar   rune
	strict    bool
}

/*
 * Encoder
 */

// Encode encodes src using the encoding enc,
// writing [Encoding.EncodedLen](len(src)) bytes to dst.
//
// The encoding pads the output to a multiple of 4 bytes,
// so Encode is not appropriate for use on individual blocks
// of a large data stream. Use [NewEncoder] instead.
func (enc *Encoding) Encode(dst, src []byte) {
	if len(src) == 0 {
		return
	}
	// enc is a pointer receiver, so the use of enc.encode within the hot
	// loop below means a nil check at every operation. Lift that nil check
	// outside of the loop to speed up the encoder.
	_ = enc.encode

	di, si := 0, 0
	n := (len(src) / 3) * 3
	for si < n {
		// Convert 3x 8bit source bytes into 4 bytes
		val := uint(src[si+0])<<16 | uint(src[si+1])<<8 | uint(src[si+2])

		dst[di+0] = enc.encode[val>>18&0x3F]
		dst[di+1] = enc.encode[val>>12&0x3F]
		dst[di+2] = enc.encode[val>>6&0x3F]
		dst[di+3] = enc.encode[val&0x3F]

		si += 3
		di += 4
	}

	remain := len(src) - si
	if remain == 0 {
		return
	}
	// Add the remaining small block
	val := uint(src[si+0]) << 16
	if remain == 2 {
		val |= uint(src[si+1]) << 8
	}

	dst[di+0] = enc.encode[val>>18&0x3F]
	dst[di+1] = enc.encode[val>>12&0x3F]

	switch remain {
	case 2:
		dst[di+2] = enc.encode[val>>6&0x3F]
		if enc.padChar != NoPadding {
			dst[di+3] = byte(enc.padChar)
		}
	case 1:
		if enc.padChar != NoPadding {
			dst[di+2] = byte(enc.padChar)
			dst[di+3] = byte(enc.padChar)
		}
	}
}

```

encode 代码实现比较简单，但有几个注意点:
- `_ = enc.encode` 在循环外，避免每次循环都进行 nil 检查（hot loop）
- `val := uint(src[si+0])<<16 | uint(src[si+1])<<8 | uint(src[si+2])` 将 3 个字节转换为 24 位
- `dst[di+0] = enc.encode[val>>18&0x3F]` 将 24 位 0-5 位转换为 base64 编码
- `dst[di+1] = enc.encode[val>>12&0x3F]` 将 24 位 6-11 位转换为 base64 编码
- `dst[di+2] = enc.encode[val>>6&0x3F]` 将 24 位 12-17 位转换为 base64 编码
- `dst[di+3] = enc.encode[val&0x3F]` 将 24 位 18-23 位转换为 base64 编码

## Golang 中的 Base64 解码

解码函数是 `Decode` 函数，它将 Base64 编码的字符串解码为原始数据。

```go
// An Encoding is a radix 64 encoding/decoding scheme, defined by a
// 64-character alphabet. The most common encoding is the "base64"
// encoding defined in RFC 4648 and used in MIME (RFC 2045) and PEM
// (RFC 1421).  RFC 4648 also defines an alternate encoding, which is
// the standard encoding with - and _ substituted for + and /.
type Encoding struct {
	encode    [64]byte   // mapping of symbol index to symbol byte value
	decodeMap [256]uint8 // mapping of symbol byte value to symbol index
	padChar   rune
	strict    bool
}

// Decode decodes src using the encoding enc. It writes at most
// [Encoding.DecodedLen](len(src)) bytes to dst and returns the number of bytes
// written. The caller must ensure that dst is large enough to hold all
// the decoded data. If src contains invalid base64 data, it will return the
// number of bytes successfully written and [CorruptInputError].
// New line characters (\r and \n) are ignored.
func (enc *Encoding) Decode(dst, src []byte) (n int, err error) {
	if len(src) == 0 {
		return 0, nil
	}

	// Lift the nil check outside of the loop. enc.decodeMap is directly
	// used later in this function, to let the compiler know that the
	// receiver can't be nil.
	_ = enc.decodeMap

	si := 0
	for strconv.IntSize >= 64 && len(src)-si >= 8 && len(dst)-n >= 8 {
		src2 := src[si : si+8]
		if dn, ok := assemble64(
			enc.decodeMap[src2[0]],
			enc.decodeMap[src2[1]],
			enc.decodeMap[src2[2]],
			enc.decodeMap[src2[3]],
			enc.decodeMap[src2[4]],
			enc.decodeMap[src2[5]],
			enc.decodeMap[src2[6]],
			enc.decodeMap[src2[7]],
		); ok {
			binary.BigEndian.PutUint64(dst[n:], dn)
			n += 6
			si += 8
		} else {
			var ninc int
			si, ninc, err = enc.decodeQuantum(dst[n:], src, si)
			n += ninc
			if err != nil {
				return n, err
			}
		}
	}

	for len(src)-si >= 4 && len(dst)-n >= 4 {
		src2 := src[si : si+4]
		if dn, ok := assemble32(
			enc.decodeMap[src2[0]],
			enc.decodeMap[src2[1]],
			enc.decodeMap[src2[2]],
			enc.decodeMap[src2[3]],
		); ok {
			binary.BigEndian.PutUint32(dst[n:], dn)
			n += 3
			si += 4
		} else {
			var ninc int
			si, ninc, err = enc.decodeQuantum(dst[n:], src, si)
			n += ninc
			if err != nil {
				return n, err
			}
		}
	}

	for si < len(src) {
		var ninc int
		si, ninc, err = enc.decodeQuantum(dst[n:], src, si)
		n += ninc
		if err != nil {
			return n, err
		}
	}
	return n, err
}

// assemble32 assembles 4 base64 digits into 3 bytes.
// Each digit comes from the decode map, and will be 0xff
// if it came from an invalid character.
func assemble32(n1, n2, n3, n4 byte) (dn uint32, ok bool) {
	// Check that all the digits are valid. If any of them was 0xff, their
	// bitwise OR will be 0xff.
	if n1|n2|n3|n4 == 0xff {
		return 0, false
	}
	return uint32(n1)<<26 |
			uint32(n2)<<20 |
			uint32(n3)<<14 |
			uint32(n4)<<8,
		true
}

// assemble64 assembles 8 base64 digits into 6 bytes.
// Each digit comes from the decode map, and will be 0xff
// if it came from an invalid character.
func assemble64(n1, n2, n3, n4, n5, n6, n7, n8 byte) (dn uint64, ok bool) {
	// Check that all the digits are valid. If any of them was 0xff, their
	// bitwise OR will be 0xff.
	if n1|n2|n3|n4|n5|n6|n7|n8 == 0xff {
		return 0, false
	}
	return uint64(n1)<<58 |
			uint64(n2)<<52 |
			uint64(n3)<<46 |
			uint64(n4)<<40 |
			uint64(n5)<<34 |
			uint64(n6)<<28 |
			uint64(n7)<<22 |
			uint64(n8)<<16,
		true
}
```

`Decode` 函数实现比较简单:
- 读取 8 个字节，通过 assemble64 函数将 8 个字节转换为 6 个字节，然后写入到 dst 中。
- 如果读取的字节数不足 8 个字节，则通过 assemble32 函数将 4 个字节转换为 3 个字节，然后写入到 dst 中。
- 如果读取的字节数不足 4 个字节，则通过 assemble32 函数将 4 个字节转换为 3 个字节，然后写入到 dst 中。