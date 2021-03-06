# the irc crate [![Build Status][ci-badge]][ci] [![Crates.io][cr-badge]][cr] ![Downloads][dl-badge] [![Docs][doc-badge]][doc] [![Built with Spacemacs][bws]][sm]

[ci-badge]: https://travis-ci.org/aatxe/irc.svg?branch=stable
[ci]: https://travis-ci.org/aatxe/irc
[cr-badge]: https://img.shields.io/crates/v/irc.svg
[cr]: https://crates.io/crates/irc
[dl-badge]: https://img.shields.io/crates/d/irc.svg
[doc-badge]: https://docs.rs/irc/badge.svg
[doc]: https://docs.rs/irc
[bws]: https://cdn.rawgit.com/syl20bnr/spacemacs/442d025779da2f62fc86c2082703697714db6514/assets/spacemacs-badge.svg
[sm]: http://spacemacs.org

[rfc2812]: http://tools.ietf.org/html/rfc2812
[ircv3.1]: http://ircv3.net/irc/3.1.html 
[ircv3.2]: http://ircv3.net/irc/3.2.html

"the irc crate" is a thread-safe and async-friendly IRC client library written in Rust. It's
compliant with [RFC 2812][rfc2812], [IRCv3.1][ircv3.1], [IRCv3.2][ircv3.2], and includes some
additional, common features from popular IRCds. You can find up-to-date, ready-to-use documentation
online [on docs.rs][doc].

## Built with the irc crate

the irc crate is being used to build new IRC software in Rust. Here are some of our favorite
projects:

- [alectro][alectro] — a terminal IRC client
- [spilo][spilo] — a minimalistic IRC bouncer
- [irc-bot.rs][ircbot] — a library for writing IRC bots
- [playbot_ng][playbot_ng] — a Rust-evaluating IRC bot in Rust
- [bunnybutt-rs][bunnybutt] — an IRC bot for the [Feed The Beast Wiki][ftb-wiki]

[alectro]: https://github.com/aatxe/alectro
[spilo]: https://github.com/aatxe/spilo
[ircbot]: https://github.com/8573/irc-bot.rs
[bunnybutt]: https://github.com/FTB-Gamepedia/bunnybutt-rs
[playbot_ng]: https://github.com/panicbit/playbot_ng
[ftb-wiki]: https://ftb.gamepedia.com/FTB_Wiki

Making your own project? [Submit a pull request](https://github.com/aatxe/irc/pulls) to add it!

## Getting Started

To start using the irc crate with cargo, you can simply add `irc = "0.13"` to your dependencies in
your Cargo.toml file. The high-level API can be found in [`irc::client::prelude`][irc-prelude].
You'll find a number of examples to help you get started in `examples/`, throughout the
documentation, and below.

[irc-prelude]: https://docs.rs/irc/*/irc/client/prelude/index.html

## A Tale of Two APIs

### Reactors (The "New" API)

The release of v0.13 brought with it a new API called `IrcReactor` that enables easier multiserver
support and more graceful error handling. The general model is that you use the reactor to create
new `IrcClients`, register message handler functions, and finally block the thread to run the
clients with their respective handlers. Here's an example:

```rust,no_run
extern crate irc;

use irc::client::prelude::*;

fn main() {
    // We can also load the Config at runtime via Config::load("path/to/config.toml")
    let config = Config {
        nickname: Some("the-irc-crate".to_owned()),
        server: Some("irc.pdgn.co".to_owned()),
        channels: Some(vec!["#test".to_owned()]),
        ..Config::default()
    };

    let mut reactor = IrcReactor::new().unwrap();
    let client = reactor.prepare_client_and_connect(&config).unwrap();
    client.identify().unwrap();
    
    reactor.register_client_with_handler(client, |client, message| {
        print!("{}", message);
        // And here we can do whatever we want with the messages.
        Ok(())
    });

    reactor.run().unwrap();
}
```


### Direct Style (The "Old" API)

The old API for connecting to an IRC server is still supported through the `IrcClient` type. It's
simpler for the most basic use case, but will panic upon encountering any sort of connection issues.
In general, it's recommended that users switch to the new API if possible. Nevertheless, here is an
example:

```rust,no_run
extern crate irc;

use std::default::Default;
use irc::client::prelude::*;

fn main() {
    // We can also load the Config at runtime via Config::load("path/to/config.toml")
    let cfg = Config {
        nickname: Some(format!("the-irc-crate")),
        server: Some(format!("irc.example.com")),
        channels: Some(vec![format!("#test")]),
        .. Default::default()
    };

    let client = IrcClient::from_config(cfg).unwrap();
    client.identify().unwrap();

    client.for_each_incoming(|message| {
        print!("{}", message);
        // And here we can do whatever we want with the messages.
    }).unwrap()
}
```

## Configuring IRC Clients

As seen above, there are two techniques for configuring the irc crate: runtime loading and
programmatic configuration. Runtime loading is done via the function `Config::load`, and is likely
sufficient for most IRC bots. Programmatic configuration is convenient for writing tests, but can
also be useful when defining your own custom configuration format that can be converted to `Config`.
The primary configuration format is TOML, but if you are so inclined, you can use JSON and/or YAML
via the optional `json` and `yaml` features respectively. At the minimum, a configuration requires
`nickname` and `server` to be defined, and all other fields are optional. You can find detailed
explanations of the various fields on [docs.rs][config-fields].

[config-fields]: https://docs.rs/irc/*/irc/client/data/config/struct.Config.html#fields

Alternatively, you can look at the example below of a TOML configuration with all the fields:

```toml
owners = []
nickname = "user"
nick_password = "password"
alt_nicks = ["user_", "user__"]
username = "user"
realname = "Test User"
server = "chat.freenode.net"
port = 6697
password = ""
use_ssl = true
cert_path = "cert.der"
client_cert_path = "client.der"
client_cert_pass = "password"
encoding = "UTF-8"
channels = ["#rust", "#haskell", "#fake"]
umodes = "+RB-x"
user_info = "I'm a test user for the irc crate."
version = "irc:git:Rust"
source = "https://github.com/aatxe/irc"
ping_time = 180
ping_timeout = 10
burst_window_length = 8
max_messages_in_burst = 15
should_ghost = false
ghost_sequence = []

[channel_keys]
"#fake" = "password"

[options]
note = "anything you want can be in here!"
and = "you can use it to build your own additional configuration options."
key = "value"
```

You can convert between different configuration formats with `convertconf` like so:

```shell
cargo run --example convertconf -- -i client_config.json -o client_config.toml
```

Note that the formats are automatically determined based on the selected file extensions. This 
tool should make it easy for users to migrate their old configurations to TOML.

## Contributing
the irc crate is a free, open source library that relies on contributions from its maintainers,
Aaron Weiss ([@aatxe][awe]) and Peter Atashian ([@retep998][bun]), as well as the broader Rust
community. It's licensed under the Mozilla Public License 2.0 whose text can be found in
`LICENSE.md`. To foster an inclusive community around the irc crate, we have adopted a Code of
Conduct whose text can be found in `CODE_OF_CONDUCT.md`. You can find details about how to
contribute in `CONTRIBUTING.md`.

[awe]: https://github.com/aatxe/
[bun]: https://github.com/retep998/
