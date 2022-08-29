# Morgan

**PyPI Mirror for Restricted/Offline Environments**

## TOC

<!-- vim-markdown-toc GFM -->

* [Overview](#overview)
* [Features and Concepts](#features-and-concepts)
* [Installation](#installation)
* [Usage](#usage)
    * [Sample Configuration File](#sample-configuration-file)
* [Limitations](#limitations)
* [Why Not Use X?](#why-not-use-x)
* [License](#license)

<!-- vim-markdown-toc -->

## Overview

Morgan is a PyPI mirror for restricted/offline networks/environments, where
access to the Internet is not available. It allows creating small mirrors that
can be used by multiple "client" Python environments (i.e. CPython versions,
operating systems, etc.). The Morgan server is a single-file script that only
uses modules from the standard library, making it easy to deploy in such
environments.

The basic idea is to run mirror utility where Internet access is available, which
generates a directory tree ("package index") with all of the required packages
and the server script, copy this tree to the restricted network (going through
whatever security policies are in place), run the server inside the restricted
network, and set `pip` in the client environments to use the mirror instead of
pypi.org, which they can't access anyway.

## Features and Concepts

- Runs under **Python 3.7 and up** (both server and mirrorer).
- Package index is a **simple directory tree** that can be easily archived, copied,
  rsynced, etc.
- Package index contains a **configuration file** that lists **markers** for
  different client environments as per [PEP 345](https://peps.python.org/pep-0345/#environment-markers), and a list of
  **package requirement strings**, as defined by [PEP 508](https://peps.python.org/pep-0508/).
- Mirrorer automatically and recursively mirrors **dependencies** of all direct
  requirements.
- Only mirrors **optional dependencies** if they were part of the requirement
  strings (a.k.a "extras"), or are relevant to the environment markers. This is
  true for direct requirements and for dependencies of dependencies.
- For each requirement, downloads the **latest version** that satisfies the
  requirement (e.g. "requests>=2.24.0,<2.27.0" will download 2.26.0, whereas
  "requests" will download the latest available version).
- Downloads both **source distributions** and **binary distributions**. Only binary
  distributions that are relevant to either of the environment markers defined in
  the configuration file are downloaded.
- **Server** is a one-file script with **no dependencies** outside the standard library
  available in Python 3.7 and above. The script is automatically extracted into
  the package index.
- Server implements [PEP 503](https://peps.python.org/pep-0503/) (Simple Repository API),
  [PEP 658](https://peps.python.org/pep-0658/) (Serve Distribution Metadata in the Simple Repository API),
  and [PEP 691](https://peps.python.org/pep-0691/) (JSON-based Simple API for Python Package Indexes).

## Installation

Morgan is meant to be used as a command line utility (although it can be used
as a library if necessary), therefore it is recommended to install via a utility
such as [pipx](https://github.com/pypa/pipx):

```sh
pipx install morgan
```

You can also install it directly through `pip`:

```sh
python3 -m pip install morgan
```

## Usage

1. Create a directory where the package index will reside.
2. Create a "morgan.ini" file in this directory, with at least one environment
   definition and list of requirements (see [Sample Configuration File](#sample-configuration-file) below).
   You can use `morgan generate_env >> morgan.ini` to generate a configuration
   block for the local interpreter.
3. Run the mirrorer from the package index via `morgan mirror` (alternatively,
   provide the path of the package index via the `--index-path` flag).
4. Copy the package index to the target environment, if necessary.
5. Run the server using `python3 server.py`. Use `--help` for a full list of
   flags and options. You can also use `morgan server` instead.

### Sample Configuration File

Environment configuration blocks can be automatically generated via
`morgan generate_env`. I recommend you read the "Environment Markers" section of
[PEP 345](https://peps.python.org/pep-0345/#environment-markers) to see exactly how they are calculated.

```ini
[env.legacy]
python_version = 3.7
python_full_version = 3.7.7
os_name = posix
sys_platform = linux
platform_machine = i686
platform_python_implementation = CPython
platform_system = Linux
implementation_name = cpython

[env.edge]
os_name = posix
sys_platform = linux
platform_machine = x86_64
platform_python_implementation = CPython
platform_system = Linux
python_version = 3.10
python_full_version = 3.10.6
implementation_name = cpython

[requirements]
requests = >=2.24.0
protobuf = ==3.20.1
redis = >4.1.0,<4.2.1
xonsh = [full]~=0.11.0
```

In this example we can see two different client environments: a "legacy" one with
a relatively old installation of CPython 3.7.7 on 32-bit Linux, and an "edge"
environment with a recent installation of CPython 3.10.6 on 64-bit Linux.

All these different markers are needed because they can be used in package
metadata, and Morgan needs them to determine which files and dependencies to
download.

This configuration file sets "requests", "protobuf" and "redis" as the packages
to mirror, with certain version specifications for each. When the mirrorer is
executed on a directory that contains this configuration file, it will find the
latest version that satisfies the specifications of each package, download
relevant files for each of them, and then recursively download _their_
dependencies. It will download source files suitable for all environments, and
binary distributions (wheels) that match the definitions of the "legacy" and
"edge" environments.

Configuration markers for a specific environment can easily be generated by
running the provided `generate_env` command in the target environment.

## Limitations

- Morgan currently only targets CPython. Packages/binary files that are specific
  to other Python implementations are not mirrored. This may change.

- Morgan only targets Python 3 packages. Python 2 packages are not mirrored and
  probably never will be.

- The only binary distributions supported are wheels. Eggs are not supported and
  probably never will be.

- The Morgan server is currently read-only. Packages cannot be published to the
  local index through it. This may change in the future.

- Morgan does not mirror packages with versions that do not comply with [PEP 440](https://peps.python.org/pep-0440/#version-scheme).
  These are generally older packages that are no longer accepted by PyPI anyway.
  This is unlikely to change.

- Morgan does not mirror yanked files (see [PEP 592](https://peps.python.org/pep-0592/)). These are files that should
  not be downloaded from PyPI unless a project is specifically pinned to those
  versions. This was a conscious decision that will probably not change, but
  may be made configurable.

## Why Not Use X?

Morgan was written because of insufficiencies of other mirroring solutions:

- [bandersnatch](https://github.com/pypa/bandersnatch/) is geared more towards mirroring of the entire PyPI repository,
  with support for different filters to reduce/limit the size of the mirror.
  Unfortunately, it doesn't automatically download dependencies of direct
  requirements, making the "package" filter virtually unusable, and the "platform"
  filters are not fine-grained enough. It also has many dependencies outside the
  standard library.
- [localshop](https://github.com/jazzband/localshop) is a proxy that basically caches PyPI responses to `pip` requests,
  so it's not useful for restricted networks. It also has many non-standard-library
  dependencies, and can't download binary distributions for multiple environments.
- [devpi](https://www.devpi.net/) also works as a caching proxy, and also attempts to automatically refresh
  from PyPI at regular intervals. It also has many non-standard-library
  dependencies.

## License

Morgan is licensed under the terms of the [Apache License 2.0](LICENSE).