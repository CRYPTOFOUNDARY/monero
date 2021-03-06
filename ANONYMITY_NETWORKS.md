# Anonymity Networks with FlakeChain

Currently only Tor has been integrated into FlakeChain. Providing support for
Kovri/I2P should be minimal, but has not yet been attempted. The usage of
these networks is still considered experimental - there are a few pessimistic
cases where privacy is leaked. The design is intended to maximize privacy of
the source of a transaction by broadcasting it over an anonymity network, while
relying on IPv4 for the remainder of messages to make surrounding node attacks
(via sybil) more difficult.


## Behavior

If _any_ anonymity network is enabled, transactions being broadcast that lack
a valid "context" (i.e. the transaction did not come from a p2p connection),
will only be sent to peers on anonymity networks. If an anonymity network is
enabled but no peers over an anonymity network are available, an error is
logged and the transaction is kept for future broadcasting over an anonymity
network. The transaction will not be broadcast unless an anonymity connection
is made or until `flakechaind` is shutdown and restarted with only public
connections enabled.


## P2P Commands

Only handshakes, peer timed syncs, and transaction broadcast messages are
supported over anonymity networks. If one `--add-exclusive-node` onion address
is specified, then no syncing will take place and only transaction broadcasting
can occur. It is therefore recommended that `--add-exclusive-node` be combined
with additional exclusive IPv4 address(es).


## Usage

Anonymity networks have no seed nodes (the feature is still considered
experimental), so a user must specify an address. If configured properly,
additional peers can be found through typical p2p peerlist sharing.

### Outbound Connections

Connecting to an anonymous address requires the command line option
`--proxy` which tells `flakechaind` the ip/port of a socks proxy provided by a
separate process. On most systems the configuration will look like:

> `--proxy tor,127.0.0.1:9050,10`
> `--proxy i2p,127.0.0.1:9000`

which tells `flakechaind` that ".onion" p2p addresses can be forwarded to a socks
proxy at IP 127.0.0.1 port 9050 with a max of 10 outgoing connections and
".i2p" p2p addresses can be forwarded to a socks proxy at IP 127.0.0.1 port 9000
with the default max outgoing connections. Since there are no seed nodes for
anonymity connections, peers must be manually specified:

> `--add-exclusive-node rveahdfho7wo4b2m.onion:28083`
> `--add-peer rveahdfho7wo4b2m.onion:28083`

Either option can be listed multiple times, and can specify any mix of Tor,
I2P, and IPv4 addresses. Using `--add-exclusive-node` will prevent the usage of
seed nodes on ALL networks, which will typically be undesireable.

### Inbound Connections

Receiving anonymity connections is done through the option
`--anonymous-inbound`. This option tells `flakechaind` the inbound address, network
type, and max connections:

> `--anonymous-inbound rveahdfho7wo4b2m.onion:28083,127.0.0.28083,25`
> `--anonymous-inbound foobar.i2p:5000,127.0.0.1:30000`

which tells `flakechaind` that a max of 25 inbound Tor connections are being
received at address "rveahdfho7wo4b2m.onion:28083" and forwarded to `flakechaind`
localhost port 28083, and a default max I2P connections are being received at
address "foobar.i2p:5000" and forwarded to `flakechaind` localhost port 30000.
These addresses will be shared with outgoing peers, over the same network type,
otherwise the peer will not be notified of the peer address by the proxy.

### Network Types

#### Tor

Options `--add-exclusive-node` and `--add-peer` recognize ".onion" addresses,
and will properly forward those addresses to the proxy provided with
`--proxy tor,...`.

Option `--anonymous-inbound` also recognizes ".onion" addresses, and will
automatically be sent out to outgoing Tor connections so the peer can
distribute the address to its other peers.

##### Configuration

Tor must be configured for hidden services. An example configuration ("torrc")
might look like:

> HiddenServiceDir /var/lib/tor/data/flakechain
> HiddenServicePort 28083 127.0.0.1:28083

This will store key information in `/var/lib/tor/data/flakechain` and will forward
"Tor port" 28083 to port 28083 of ip 127.0.0.1. The file
`/usr/lib/tor/data/flakechain/hostname` will contain the ".onion" address for use
with `--anonymous-inbound`.

#### Kovri/I2P

Support for this network has not been implemented. Using ".i2p" addresses or
specifying "i2p" will currently generate an error.


## Privacy Limitations

There are currently some techniques that could be used to _possibly_ identify
the machine that broadcast a transaction over an anonymity network.

### Timestamps

The peer timed sync command sends the current time in the message. This value
can be used to link an onion address to an IPv4/IPv6 address. If a peer first
sees a transaction over Tor, it could _assume_ (possibly incorrectly) that the
transaction originated from the peer. If both the Tor connection and an
IPv4/IPv6 connection have timestamps that are approximately close in value they
could be used to link the two connections. This is less likely to happen if the
system clock is fairly accurate - many peers on the FlakeChain network should have
similar timestamps.

#### Mitigation

Keep the system clock accurate so that fingerprinting is more difficult. In
the future a random offset might be applied to anonymity networks so that if
the system clock is noticeably off (and therefore more fingerprintable),
linking the public IPv4/IPv6 connections with the anonymity networks will be
more difficult.

### Bandwidth Usage

An ISP can passively monitor `flakechaind` connections from a node and observe when
a transaction is sent over a Tor/Kovri connection via timing analysis + size of
data sent during that timeframe. Kovri should provide better protection against
this attack - its connections are not circuit based. However, if a node is
only using Kovri for broadcasting FlakeChain transactions, the total aggregate of
Kovri/I2P data would also leak information.

#### Mitigation

There is no current mitigation for the user right now. This attack is fairly
sophisticated, and likely requires support from the internet host of a FlakeChain
user.

In the near future, "whitening" the amount of data sent over anonymity network
connections will be performed. An attempt will be made to make a transaction
broadcast indistinguishable from a peer timed sync command.

### Intermittent FlakeChain Syncing

If a user only runs `flakechaind` to send a transaction then quit, this can also
be used by an ISP to link a user to a transaction.

#### Mitigation

Run `flakechaind` as often as possible to conceal when transactions are being sent.
Future versions will also have peers that first receive a transaction over an
anonymity network delay the broadcast to public peers by a randomized amount.
This will not completetely mitigate a user who syncs up sends then quits, in
part because this rule is not enforceable, so this mitigation strategy is
simply a best effort attempt.

### Active Bandwidth Shaping

An attacker could attempt to bandwidth shape traffic in an attempt to determine
the source of a Tor/Kovri/I2P connection. There isn't great mitigation against
this, but Kovri/I2P should provide better protection against this attack since
the connections are not circuit based.

#### Mitigation

The best mitigiation is to use Kovri/I2P instead of Tor. However, Kovri/I2P
has a smaller set of users (less cover traffic) and academic reviews, so there
is a tradeoff in potential isses. Also, anyone attempting this strategy really
wants to uncover a user, it seems unlikely that this would be performed against
every Tor/Kovri/I2P user.

