# Backport: TLS 1.1/1.2 support for Qt 4.8.4 with OpenSSL 1.0.2u

## Overview

- Adds TLS 1.1 and TLS 1.2 to Qt 4.8.4's SSL stack when used with OpenSSL 1.0.2u.
- Keeps API/ABI stable; apps opt-in via `QSsl::SslProtocol` or use `AnyProtocol`.
- Uses runtime symbol resolution; works with dynamically loaded 1.0.2u DLLs.
- Default protocol remains `SecureProtocols` (Qt 4.8.4 behavior).

## What Changed

- New protocols: `QSsl::TlsV1_1` and `QSsl::TlsV1_2` in `src/network/ssl/qssl.h`.
- Resolver: declares and resolves `TLSv1_1_*_method` and `TLSv1_2_*_method` in
  `src/network/ssl/qsslsocket_openssl_symbols_p.h/.cpp`.
- Enforcement: for TLSv1.1/1.2 selections, creates an SSLv23 context and sets
  `SSL_OP_NO_*` flags to disable lower versions in
  `src/network/ssl/qsslsocket_openssl.cpp`.
- Cipher parsing: maps "TLSv1.1" and "TLSv1.2" description strings to the new
  protocol enums in `src/network/ssl/qsslsocket_openssl.cpp`.
- Docs: updates the `QSsl::SslProtocol` documentation in `src/network/ssl/qssl.cpp`.

## Files Modified

- `src/network/ssl/qssl.h`: added enum values `TlsV1_1`, `TlsV1_2`.
- `src/network/ssl/qsslsocket_openssl_symbols_p.h`:
  - added TLSv1.1/1.2 method prototypes for client/server on both const and non-const branches.
  - defined `SSL_OP_NO_TLSv1_1` and `SSL_OP_NO_TLSv1_2` if missing.
- `src/network/ssl/qsslsocket_openssl_symbols.cpp`:
  - added `DEFINEFUNC` entries for `TLSv1_1_*_method` and `TLSv1_2_*_method`.
  - added `RESOLVEFUNC(...)` calls to resolve the new methods by name.
- `src/network/ssl/qsslsocket_openssl.cpp`:
  - `QSslCipher_from_SSL_CIPHER(...)` now maps "TLSv1.1" and "TLSv1.2" to the new enums.
  - `initSslContext()` handles `TlsV1_1` and `TlsV1_2` by using `SSLv23_*_method()` and
    disabling lower versions via `SSL_OP_NO_*` flags.
  - SNI setup treats TLSv1.1/1.2 like TLSv1/AnyProtocol.
- `src/network/ssl/qssl.cpp`: documents the new TLS protocol enum values.

## Build Notes (MinGW 4.4, OpenSSL 1.0.2u)

### Prerequisite

(Optional) Set the `OPENSSL_INCDIR` and `OPENSSL_LIBDIR`

```ps
$env:OPENSSL_INCDIR = "C:\Users\liya\Repos\openssl-1.0.2u-symbian\outinc"
$env:OPENSSL_LIBDIR = "C:\Users\liya\Repos\openssl-1.0.2u-symbian\out"
```

From repo root:

### Option A — dynamic OpenSSL (recommended with your DLLs)

- This uses Qt's runtime resolver (no link to import libs needed).
- Build Qt normally after the patch:

```ps
./configure.exe -platform win32-g++ -opensource -confirm-license -nomake demos -nomake examples -no-webkit -openssl -I C:\Users\liya\Repos\openssl-1.0.2u-symbian\outinc
```

Or use environment variables:

```ps
./configure.exe -platform win32-g++ -opensource -confirm-license -nomake demos -nomake examples -no-webkit -openssl -I %OPENSSL_INCDIR%
```


### Option B — link against OpenSSL

- Requires import libs in your OpenSSL folder (e.g., `libssl.dll.a`, `libcrypto.dll.a`).
- If your import libs are `libssleay32.a` and `libeay32.a`, set `OPENSSL_LIBS=-lssleay32 -leay32`.
  If they are `libssl.dll.a` and `libcrypto.dll.a`, use `OPENSSL_LIBS=-lssl -lcrypto`.

```ps
./configure.exe -platform win32-g++ -opensource -confirm-license -nomake demos -nomake examples -no-webkit -openssl-linked -I C:\Users\liya\Repos\openssl-1.0.2u-symbian\outinc -L C:\Users\liya\Repos\openssl-1.0.2u-symbian\out
```

Or use environment variables:

```ps
./configure.exe -platform win32-g++ -opensource -confirm-license -nomake demos -nomake examples -no-webkit -openssl-linked -I %OPENSSL_INCDIR% -L %OPENSSL_LIBDIR%
```

Then build the essential libs:

```ps
mingw32-make -C src/tools/bootstrap -j8
```

```ps
mingw32-make -C src/tools/moc -j8
```

If you are building only QtNetwork, build corelib first:

```ps
mingw32-make -C src/corelib -j8
```

Now build QtNetwork:

```ps
mingw32-make -C src/network -j8
```

- **(Option A)** At runtime, ensure `ssleay32.dll` and `libeay32.dll` from your 1.0.2u build are
- **(Option B)** Deploy the matching DLLs with your application.

## Usage

- Use `QSsl::TlsV1_2` to force TLS 1.2:

```cpp
socket.setProtocol(QSsl::TlsV1_2);
```

- `AnyProtocol` will negotiate the highest version supported by the SSL backend.
- OpenSSL 1.0.2u supports up to TLS 1.2 (not TLS 1.3).

## Verification

- Build QtNetwork and link a small test to `connectToHostEncrypted()` against a TLS 1.2-only endpoint.
- Connect to a TLS 1.2-only server and check
  `QSslSocket::sessionCipher().protocol()` returns `QSsl::TlsV1_2` (or `TlsV1_1`).
- Ensure your `ssleay32.dll`/`libeay32.dll` are from 1.0.2u and load first at runtime.

## Notes

- `SSL_OP_NO_TLSv1_1` and `SSL_OP_NO_TLSv1_2` are defined if absent to compile
  on older headers; they are no-ops if the runtime lacks those options.
- This patch does not change the default protocol from `SecureProtocols`.
  If you want a default of `AnyProtocol`, apply a separate change to
  `src/network/ssl/qsslconfiguration_p.h` and `src/network/ssl/qsslconfiguration.cpp`.
