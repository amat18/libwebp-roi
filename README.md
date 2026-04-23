# WebP Codec

```
      __   __  ____  ____  ____
     /  \\/  \/  _ \/  _ )/  _ \
     \       /   __/  _  \   __/
      \__\__/\____/\_____/__/ ____  ___
            / _/ /    \    \ /  _ \/ _/
           /  \_/   / /   \ \   __/  \__
           \____/____/\_____/_____/____/v1.6.0
```

WebP codec is a library to encode and decode images in WebP format. This package
contains the library that can be used in other programs to add WebP support, as
well as the command line tools 'cwebp' and 'dwebp' to compress and decompress
images respectively.

See https://developers.google.com/speed/webp for details on the image format.

The latest source tree is available at
https://chromium.googlesource.com/webm/libwebp

It is released under the same license as the WebM project. See
https://www.webmproject.org/license/software/ or the "COPYING" file for details.
An additional intellectual property rights grant can be found in the file
PATENTS.

## Building

See the [building documentation](doc/building.md).

## Encoding and Decoding Tools

The examples/ directory contains tools to encode and decode images and
animations, view information about WebP images, and more. See the
[tools documentation](doc/tools.md).

## APIs

See the [APIs documentation](doc/api.md), and API usage examples in the
`examples/` directory.

## Bugs

Please report all bugs to the [issue tracker](https://issues.webmproject.org).
For security reports, select 'Security report' from the Template dropdown.

Patches welcome! See [how to contribute](CONTRIBUTING.md).

## Discuss

Email: webp-discuss@webmproject.org

Web: https://groups.google.com/a/webmproject.org/group/webp-discuss

---

## ROI Encoding Feature (this fork) / ROI エンコード機能（このフォーク）

This fork extends `cwebp` with ROI (Region of Interest) encoding.
The ROI area is encoded at maximum quality; surrounding areas at minimum quality.

このフォークは `cwebp` にROI（関心領域）エンコード機能を追加します。
ROI領域を最高画質、それ以外を最低画質でエンコードします。

**Documentation / ドキュメント:**
- [日本語ドキュメント](libwebp-roi-ja.md)
- [English Documentation](libwebp-roi-en.md)

---

## License / ライセンス

This fork inherits the BSD 3-Clause License from the original libwebp.

このフォークはオリジナルのlibwebpのBSD 3-Clauseライセンスを継承します。

```
Copyright (c) 2010, Google Inc. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

  * Redistributions of source code must retain the above copyright notice,
    this list of conditions and the following disclaimer.
  * Redistributions in binary form must reproduce the above copyright notice,
    this list of conditions and the following disclaimer in the documentation
    and/or other materials provided with the distribution.
  * Neither the name of Google Inc. nor the names of its contributors may be
    used to endorse or promote products derived from this software without
    specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
```

See also: [COPYING](COPYING), [PATENTS](PATENTS)

---

## About this document / このドキュメントについて

This document was written with [Claude Code](https://claude.ai/code).

このドキュメントは [Claude Code](https://claude.ai/code) を使用して作成しました。
