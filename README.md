<div align="center" markdown="1">

<img src="https://user-images.githubusercontent.com/29507195/207267672-d422db6c-d89a-4bbe-9822-468a55c15053.png" alt="fair Books logo" width="384"/>

---

[![GitHub release (latest by date)](https://img.shields.io/github/v/release/fair/books)](https://github.com/fair/books/releases)
![Platforms](https://img.shields.io/badge/platform-mac%2C%20windows%2C%20linux-yellowgreen)
[![Publish](https://github.com/fair/books/actions/workflows/publish.yml/badge.svg)](https://github.com/fair/books/actions/workflows/publish.yml)

Free Desktop book-keeping software for small businesses and freelancers.

[fairbooks.com](https://fairbooks.com/)

<img src="https://user-images.githubusercontent.com/29507195/207267857-4ae48890-3fb2-4046-80cf-3256b46c72a0.png" alt="fair Books Preview"/>

</div>

## Index

<details>
<summary><code>[show/hide]</code></summary>

1. [Features](#features)
2. [Installation](#installation)
3. [Development](#development)
4. [Contributions and Community](#contributions-and-community)
5. [Links](#links)
6. [Translation Contributors](#translation-contributors)
7. [License](#license)

</details>

## Features

1. Double-entry accounting
1. Invoicing
1. Billing
1. Payments
1. Journal Entries
1. Dashboard
1. Works Offline
1. Financial Reports
   - General Ledger
   - Profit and Loss Statement
   - Balance Sheet
   - Trial Balance

## Installation

Download and install the latest release for your platform from the [releases
page](https://github.com/fair/books/releases) or the [download
page](https://fairbooks.com/download).

## Development

fair Books is built on Vue.js and Electron. It is offline by default and uses
a local SQLite file as the database.

### Pre-requisites

To get the dev environment up and running you need to first set up Node.js version
16.13.1 and npm. For this, we suggest using
[nvm](https://github.com/nvm-sh/nvm#installing-and-updating).

Next, you will need to install [yarn](https://classic.yarnpkg.com/lang/en/docs/install/#mac-stable).

### Clone and Run

Once you are through the Pre-requisites, you can run the following commands to
setup fair Books for development and building:

```bash
# clone the repository
git clone https://github.com/fair/books.git

# change directory
cd books

# install dependencies
yarn
```

#### Development

To run fair Books in development mode (with hot reload, etc):

```bash
# start the electron app
yarn electron:serve
```

#### Build

To build fair Books and create an installer:

```bash
# start the electron app
yarn electron:build
```

**Note**
By default the above command will build for your computer's operating system and
architecture. To build for other environments (example: for linux from a windows
computer) check the _Building_ section at
[electron.build/cli](https://www.electron.build/cli).

So to build for linux you could use the `--linux` flag like so: `yarn electron:build --linux`.


## License

[GNU Affero General Public License v3.0](LICENSE)
