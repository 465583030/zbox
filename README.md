![Zbox](https://www.zbox.io/img/logo_96x96.png) Zbox
======
[![Travis](https://img.shields.io/travis/zboxfs/zbox.svg)](https://travis-ci.org/zboxfs/zbox)
[![Crates.io](https://img.shields.io/crates/d/zbox.svg)](https://crates.io/crates/zbox)
[![Crates.io](https://img.shields.io/crates/v/zbox.svg)](https://crates.io/crates/zbox)
[![GitHub last commit](https://img.shields.io/github/last-commit/zboxfs/zbox.svg)](https://github.com/zboxfs/zbox)
[![license](https://img.shields.io/github/license/zboxfs/zbox.svg)](https://github.com/zboxfs/zbox)
[![GitHub stars](https://img.shields.io/github/stars/zboxfs/zbox.svg?style=social&label=Stars)](https://github.com/zboxfs/zbox)

Zbox is a zero-knowledge, privacy focused embeddable file system. Its goal is
to help application store files securely, privately and reliably. By
encapsulating files and directories into an encrypted repository, it provides
a virtual file system and exclusive access to authorised application.

Unlike other system-level file systems, such as ext4, XFS and btrfs, which
provide shared access to multiple processes, Zbox is file system that runs
inside the same memory space as the application. It only provide access to one
process at a time.

By abstracting IO access, Zbox supports a variety of underneath storage layers.
Memory and OS file system are supported, RDBMS and key-value object store
supports are incoming soon.

Disclaimer
----------
Zbox is under active development, we are not responsible for any data loss
or leak caused by using it. Always back up your files and use at your own risk!

Features
========
- Everything is encrypted :lock:, including metadata and directory structure,
  no knowledge is leaked to underneath storage
- State-of-the-art cryptography: AES-256-GCM (hardware), ChaCha20-Poly1305,
  Argon2 password hashing and etc., empowered by
  [libsodium](https://libsodium.org/)
- Content-based data chunk deduplication and file-based deduplication
- Data compression using [LZ4](http://www.lz4.org) in fast mode
- Data integrity is guranteed by authenticated encryption primitives
- File content Revision history
- Copy-on-write (COW :cow:) semantics
- ACID transactional operations
- Snapshot :camera:
- Support multiple storages, including memory, OS file system, RDBMS (incoming),
  Key-value object store (incoming) and more
- Build in Rust with :hearts:

Comparison
----------
Many OS-level file systems support encryption, such as
[EncFS](https://vgough.github.io/encfs/),
[APFS](https://en.wikipedia.org/wiki/Apple_File_System) and
[ZFS](https://en.wikipedia.org/wiki/ZFS). Some disk encryption tools also
provide virtual file system, such as TrueCrypt and VeraCrypt. Below is the
comparison between Zbox and them.

|                             | Zbox                     | OS-level File Systems    | Disk Encryption Tools    |
| --------------------------- | ------------------------ | ------------------------ | ------------------------ |
| Encrypts file contents      | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |
| Encrypts file metadata      | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |
| Encrypts directory          | :heavy_check_mark:       | partial                  | :heavy_check_mark:       |
| Data integrity              | :heavy_check_mark:       | partial                  | :heavy_multiplication_x: |
| Shared access for processes | :heavy_multiplication_x: | :heavy_check_mark:       | :heavy_check_mark:       |
| Deduplication               | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| Compression                 | :heavy_check_mark:       | partial                  | :heavy_multiplication_x: |
| COW semantics               | :heavy_check_mark:       | partial                  | :heavy_multiplication_x: |
| ACID Transaction            | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| Multiple storage layers     | :heavy_check_mark:       | :heavy_multiplication_x: | :heavy_multiplication_x: |
| Direct API access           | :heavy_check_mark:       | through VFS              | through VFS              |
| Symbolic links              | :heavy_multiplication_x: | :heavy_check_mark:       | depends on inner FS      |
| FUSE support                | :heavy_multiplication_x: | :heavy_check_mark:       | :heavy_check_mark:       |
| Linux and macOS support     | :heavy_check_mark:       | :heavy_check_mark:       | :heavy_check_mark:       |
| Windows support             | :heavy_multiplication_x: | partial                  | :heavy_check_mark:       |

How to use
==========
For reference documentation, please visit [documentation](https://docs.rs/zbox).

Requirements
------------
- [Rust](https://www.rust-lang.org/) stable >= 1.21
- [libsodium](https://libsodium.org/) >= 1.0.15

Supported Platforms
-------------------
- 64-bit Debian-based Linux, such as Ubuntu
- 64-bit macOS

32-bit OS and Windows are not supported yet.

Usage
------
Add the following dependency to your `Cargo.toml`:

```toml
[dependencies]
zbox = "~0.1"
```

Example
-------
```rust
extern crate zbox;

use std::io::{Read, Write};
use zbox::{zbox_init, RepoOpener, OpenOptions};

fn main() {
    // Initialise zbox environment, need to be called first and only.
    zbox_init();

    // Speicify repository location using URI-like string. Currently, two
    // types of prefixes are supported:
    //   - "file://": use OS file as storage
    //   - "mem://": use memory as storage
    // After the prefix is the actual location of repository. Here we're
    // going to create an OS file repository called 'my_repo' under current
    // directory.
    let repo_uri = "file://./my_repo";

    // Speicify password of the repository.
    let pwd = "your secret";

    // Create and open the repository.
    let mut repo = RepoOpener::new().create(true).open(&repo_uri, &pwd).unwrap();

    // Speicify file path we need to create in the repository and its data.
    let file_path = "/my_file";
    let data = String::from("Hello, world").into_bytes();

    // Create and open a regular file for writing, this file is inside the
    // repository so it will be encrypted and kept privately.
    let mut f = OpenOptions::new()
        .create(true)
        .open(&mut repo, &file_path)
        .unwrap();

    // Like normal file operations, we can use std::io::Write trait to write
    // data into it.
    f.write_all(&data).unwrap();

    // But need to finish the writting before a permanent content version
    // can be made.
    f.finish().unwrap();

    // Now we can read content from the file using std::io::Read trait.
    let mut buf = Vec::new();
    f.read_to_end(&mut buf).unwrap();

    // Convert content from bytes to string and print it to stdout. It should
    // display 'Hello, world' to your terminal.
    let output = String::from_utf8(buf).unwrap();
    println!("{}", output);
}
```

Build with Docker
-----------------
Zbox comes with Docker support, it is based on rust:latest and libsodium is
included. Check the [Dockerfile](Dockerfile) for the details.

First, we build the Docker image which can be used to compile Zbox, run below
commands from Zbox project folder.
```bash
docker build --force-rm -t zbox ./
```

After the Docker image is built, we can use it to build Zbox.
```bash
docker run --rm -v $PWD:/zbox zbox cargo build
```

Contributing
============
Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of
conduct, and the process for submitting pull requests to us.

License
=======
`Zbox` is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE)
file for details.

