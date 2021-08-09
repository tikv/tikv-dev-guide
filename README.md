# TiKV Development Guide

This repository contains the source of TiKV Development Guide.

## Requirements

Building the book requires [mdBook](https://github.com/rust-lang-nursery/mdBook). To get it:

```bash
$ cargo install mdbook
```

## Preview

To preview the book, type:

```bash
$ mdbook serve
```

By default, it will create a server listening on [http://localhost:3000](http://localhost:3000), you can open it with a brower to preview the book.

## Building

To build the book, type:

```bash
$ mdbook build
```

The output will be in the `book` subdirectory. To check it out, open it in
your web browser.

_Firefox:_
```bash
$ firefox book/index.html                       # Linux
$ open -a "Firefox" book/index.html             # OS X
$ Start-Process "firefox.exe" .\book\index.html # Windows (PowerShell)
$ start firefox.exe .\book\index.html           # Windows (Cmd)
```

_Chrome:_
```bash
$ google-chrome book/index.html                 # Linux
$ open -a "Google Chrome" book/index.html       # OS X
$ Start-Process "chrome.exe" .\book\index.html  # Windows (PowerShell)
$ start chrome.exe .\book\index.html            # Windows (Cmd)
```
