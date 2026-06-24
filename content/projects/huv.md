+++
title = "huv"
date = "2026-06-24T00:00:00+01:00"
author = "Igor Mihajlov"
cover = ""
tags = ["c", "networking", "http", "libuv"]
keywords = ["c http server", "libuv", "llhttp", "mbedtls", "http library"]
description = "A small C HTTP/1.1 server library built on libuv and llhttp, with optional TLS via mbedTLS and multi-worker scaling."
showFullContent = false
readingTime = false
hideComments = false
+++

A small C HTTP/1.1 server library built on [libuv](https://libuv.org/) and [llhttp](https://github.com/nodejs/llhttp), with optional TLS via [mbedTLS](https://www.trustedfirmware.org/projects/mbed-tls/) and multi-worker scaling via `SO_REUSEPORT` + `fork()`.

## Features

- **Non-blocking** — single-threaded event loop per worker
- **Middleware + routing** — Express-style `use()` + `get/post/...` with `:param` captures and `405 + Allow` for unregistered methods on known paths
- **HTTP/HTTPS on one process** — plain HTTP, TLS, or both in the same worker sharing one router
- **Multi-worker** — set `workers=N` to fork N processes that share the listen port, with the kernel load-balancing incoming connections

## Tech Stack

- **C** — core implementation
- **libuv** — non-blocking event loop
- **llhttp** — HTTP parsing (the parser behind Node.js)
- **mbedTLS** — optional TLS support
- **CMake** — build system

## Links

- [GitHub Repository](https://github.com/irog-c/huv)
