# codebase Change Log




## Fix buffer overflow in bundled upnp

Bundled miniupnpc was updated to 1.9.20151008. This fixes a buffer overflow in the XML parser during initial network discovery.

Details can be found here: http://talosintel.com/reports/TALOS-2015-0035/

This applies to the distributed executables only, not when building from source or using distribution provided packages.

Additionally, upnp has been disabled by default. This may result in a lower number of reachable nodes on IPv4, however this prevents future libupnpc vulnerabilities from being a structural risk to the network.




## Test for LowS signatures before relaying

Make the node require the canonical ‘low-s’ encoding for ECDSA signatures when relaying or mining. This removes a nuisance malleability vector.

Consensus behavior is unchanged.

If widely deployed this change would eliminate the last remaining known vector for nuisance malleability on SIGHASH_ALL P2PKH transactions. On the down-side it will block most transactions made by sufficiently out of date software.

Unlike the other avenues to change txids on transactions this one was randomly violated by all deployed bitcoin software prior to its discovery. So, while other malleability vectors where made non-standard as soon as they were discovered, this one has remained permitted. Even BIP62 did not propose applying this rule to old version transactions, but conforming implementations have become much more common since BIP62 was initially written.

Bitcoin Core has produced compatible signatures since a28fb70e in September 2013, but this didn’t make it into a release until 0.9 in March 2014; Bitcoinj has done so for a similar span of time. Bitcoinjs and electrum have been more recently updated.

This does not replace the need for BIP62 or similar, as miners can still cooperate to break transactions. Nor does it replace the need for wallet software to handle malleability sanely[1]. This only eliminates the cheap and irritating DOS attack.

[1] On the Malleability of Bitcoin Transactions Marcin Andrychowicz, Stefan Dziembowski, Daniel Malinowski, Łukasz Mazurek http://fc15.ifca.ai/preproceedings/bitcoin/paper_9.pdf

## Minimum relay fee default increase

The default for the ```-minrelaytxfee``` setting has been increased from ```0.00001``` to ```0.00005```.

This is necessitated by the current transaction flooding, causing outrageous memory usage on nodes due to the mempool ballooning. This is a temporary measure, bridging the time until a dynamic method for determining this fee is merged (which will be in 0.12).



## BIP113 mempool-only locktime enforcement using GetMedianTimePast()

verge transactions currently may specify a locktime indicating when they may be added to a valid block. Current consensus rules require that blocks have a block header time greater than the locktime specified in any transaction in that block.

Miners get to choose what time they use for their header time, with the consensus rule being that no node will accept a block whose time is more than two hours in the future. This creates a incentive for miners to set their header times to future values in order to include locktimed transactions which weren’t supposed to be included for up to two more hours.

The consensus rules also specify that valid blocks may have a header time greater than that of the median of the 11 previous blocks. This GetMedianTimePast() time has a key feature we generally associate with time: it can’t go backwards.

BIP113 specifies a soft fork (not enforced in this release) that weakens this perverse incentive for individual miners to use a future time by requiring that valid blocks have a computed GetMedianTimePast() greater than the locktime specified in any transaction in that block.

Mempool inclusion rules currently require transactions to be valid for immediate inclusion in a block in order to be accepted into the mempool. This release begins applying the BIP113 rule to received transactions, so transaction whose time is greater than the GetMedianTimePast() will no longer be accepted into the mempool.

Implication for miners: you will begin rejecting transactions that would not be valid under BIP113, which will prevent you from producing invalid blocks if/when BIP113 is enforced on the network. Any transactions which are valid under the current rules but not yet valid under the BIP113 rules will either be mined by other miners or delayed until they are valid under BIP113. Note, however, that time-based locktime transactions are more or less unseen on the network currently.

Implication for users: GetMedianTimePast() always trails behind the current time, so a transaction locktime set to the present time will be rejected by nodes running this release until the median time moves forward. To compensate, subtract one hour (3,600 seconds) from your locktimes to allow those transactions to be included in mempools at approximately the expected time.


## Windows bug fix for corrupted UTXO database on unclean shutdowns

Several Windows users reported that they often need to reindex the entire blockchain after an unclean shutdown of VERGE on Windows (or an unclean shutdown of Windows itself). Although unclean shutdowns remain unsafe, this release no longer relies on memory-mapped files for the UTXO database, which significantly reduced the frequency of unclean shutdowns leading to required reindexes during testing.

Other fixes for database corruption on Windows are expected in the next major release.


## Signature validation using libsecp256k1

ECDSA signatures inside verge transactions now use validation using libsecp256k1 instead of OpenSSL.

Depending on the platform, this means a significant speedup for raw signature validation speed. The advantage is largest on x86_64, where validation is over five times faster. In practice, this translates to a raw reindexing and new block validation times that are less than half of what it was before.

Libsecp256k1 has undergone very extensive testing and validation.

A side effect of this change is that libconsensus no longer depends on OpenSSL.


## Reduce upload traffic

A major part of the outbound traffic is caused by serving historic blocks to other nodes in initial block download state.

It is now possible to reduce the total upload traffic via the ```-maxuploadtarget``` parameter. This is not a hard limit but a threshold to minimize the outbound traffic. When the limit is about to be reached, the uploaded data is cut by not serving historic blocks (blocks older than one week). Moreover, any SPV peer is disconnected when they request a filtered block.

This option can be specified in MiB per day and is turned off by default (```-maxuploadtarget=0```). The recommended minimum is 144 * MAX_BLOCK_SIZE (currently 144MB) per day.

Whitelisted peers will never be disconnected, although their traffic counts for calculating the target.

A more detailed documentation about keeping traffic low can be found in [reduce-traffic](https://github.com/bitcoin/bitcoin/blob/v0.12.0/doc/reduce-traffic.md)


## Direct headers announcement (BIP 130)

Between compatible peers, [BIP 130](https://github.com/bitcoin/bips/blob/master/bip-0130.mediawiki) direct headers announcement is used. This means that blocks are advertised by announcing their headers directly, instead of just announcing the hash. In a reorganization, all new headers are sent, instead of just the new tip. This can often prevent an extra roundtrip before the actual block is downloaded.

With this change, pruning nodes are now able to relay new blocks to compatible peers.


## Memory pool limiting

Previous versions of Bitcoin Core had their mempool limited by checking a transaction’s fees against the node’s minimum relay fee. There was no upper bound on the size of the mempool and attackers could send a large number of transactions paying just slighly more than the default minimum relay fee to crash nodes with relatively low RAM. A temporary workaround for previous versions of Bitcoin Core was to raise the default minimum relay fee.

Bitcoin Core 0.12 will have a strict maximum size on the mempool. The default value is 300 MB and can be configured with the ```-maxmempool``` parameter. Whenever a transaction would cause the mempool to exceed its maximum size, the transaction that (along with in-mempool descendants) has the lowest total feerate (as a package) will be evicted and the node’s effective minimum relay feerate will be increased to match this feerate plus the initial minimum relay feerate. The initial minimum relay feerate is set to 1000 satoshis per kB.

Bitcoin Core 0.12 also introduces new default policy limits on the length and size of unconfirmed transaction chains that are allowed in the mempool (generally limiting the length of unconfirmed chains to 25 transactions, with a total size of 101 KB). These limits can be overriden using command line arguments; see the extended help (```--help -help-debug```) for more information.


## Opt-in Replace-by-fee transactions

It is now possible to replace transactions in the transaction memory pool of Bitcoin Core 0.12 nodes. Bitcoin Core will only allow replacement of transactions which have any of their inputs’ ```nSequence``` number set to less than ```0xffffffff - 1```. Moreover, a replacement transaction may only be accepted when it pays sufficient fee, as described in [BIP 125](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki).

Transaction replacement can be disabled with a new command line option, ```-mempoolreplacement=0```. Transactions signaling replacement under BIP125 will still be allowed into the mempool in this configuration, but replacements will be rejected. This option is intended for miners who want to continue the transaction selection behavior of previous releases.

The ```-mempoolreplacement``` option is not recommended for wallet users seeking to avoid receipt of unconfirmed opt-in transactions, because this option does not prevent transactions which are replaceable under BIP 125 from being accepted (only subsequent replacements, which other nodes on the network that implement BIP 125 are likely to relay and mine). Wallet users wishing to detect whether a transaction is subject to replacement under BIP 125 should instead use the updated RPC calls ```gettransaction``` and ```listtransactions```, which now have an additional field in the output indicating if a transaction is replaceable under BIP125 (“bip125-replaceable”).

Note that the wallet in Bitcoin Core 0.12 does not yet have support for creating transactions that would be replaceable under BIP 125.


## RPC: Random-cookie RPC authentication

When no ```-rpcpassword``` is specified, the daemon now uses a special ‘cookie’ file for authentication. This file is generated with random content when the daemon starts, and deleted when it exits. Its contents are used as authentication token. Read access to this file controls who can access through RPC. By default it is stored in the data directory but its location can be overridden with the option ```-rpccookiefile```.

This is similar to Tor’s CookieAuthentication: see https://www.torproject.org/docs/tor-manual.html.en

This allows running verged without having to do any manual configuration.

 
## Relay: Any sequence of pushdatas in OP_RETURN outputs now allowed

Previously OP_RETURN outputs with a payload were only relayed and mined if they had a single pushdata. This restriction has been lifted to allow any combination of data pushes and numeric constant opcodes (OP_1 to OP_16) after the OP_RETURN. The limit on OP_RETURN output size is now applied to the entire serialized scriptPubKey, 83 bytes by default. (the previous 80 byte default plus three bytes overhead)


## Relay and Mining: Priority transactions

Bitcoin Core has a heuristic ‘priority’ based on coin value and age. This calculation is used for relaying of transactions which do not pay the minimum relay fee, and can be used as an alternative way of sorting transactions for mined blocks. VERGE will relay transactions with insufficient fees depending on the setting of ```-limitfreerelay=<r>``` (default: ```r=15``` kB per minute) and ```-blockprioritysize=<s>```.

In Bitcoin Core 0.12, when mempool limit has been reached a higher minimum relay fee takes effect to limit memory usage. Transactions which do not meet this higher effective minimum relay fee will not be relayed or mined even if they rank highly according to the priority heuristic.

The mining of transactions based on their priority is also now disabled by default. To re-enable it, simply set ```-blockprioritysize=<n>``` where is the size in bytes of your blocks to reserve for these transactions. The old default was 50k, so to retain approximately the same policy, you would set ```-blockprioritysize=50000```.

Additionally, as a result of computational simplifications, the priority value used for transactions received with unconfirmed inputs is lower than in prior versions due to avoiding recomputing the amounts as input transactions confirm.

External miner policy set via the ```prioritisetransaction``` RPC to rank transactions already in the mempool continues to work as it has previously. Note, however, that if mining priority transactions is left disabled, the priority delta will be ignored and only the fee metric will be effective.

This internal automatic prioritization handling is being considered for removal entirely in Bitcoin Core 0.13, and it is at this time undecided whether the more accurate priority calculation for chained unconfirmed transactions will be restored. Community direction on this topic is particularly requested to help set project priorities.


## Automatically use Tor hidden services

Starting with Tor version 0.2.7.1 it is possible, through Tor’s control socket API, to create and destroy ‘ephemeral’ hidden services programmatically. VERGE has been updated to make use of this.

This means that if Tor is running (and proper authorization is available), VERGE automatically creates a hidden service to listen on, without manual configuration. VERGE will also use Tor automatically to connect to other .onion nodes if the control socket can be successfully opened. This will positively affect the number of available .onion nodes and their usage.

This new feature is enabled by default if VERGE is listening, and a connection to Tor can be made. It can be configured with the ```-listenonion```, ```-torcontrol``` and ```-torpassword``` settings. To show verbose debugging information, pass ```-debug=tor```.


## Notifications through ZMQ

verged can now (optionally) asynchronously notify clients through a ZMQ-based PUB socket of the arrival of new transactions and blocks. This feature requires installation of the ZMQ C API library 4.x and configuring its use through the command line or configuration file. Please see [ZeroMQ](https://github.com/bitcoin/bitcoin/blob/v0.12.0/doc/zmq.md) for details of operation.



## Wallet: Transaction fees

Various improvements have been made to how the wallet calculates transaction fees.

Users can decide to pay a predefined fee rate by setting ```-paytxfee=<n>``` (or ```settxfee <n>``` rpc during runtime). A value of ```n=0``` signals VERGE to use floating fees. By default, VERGE will use floating fees.

Based on past transaction data, floating fees approximate the fees required to get into the mth block from now. This is configurable with ```-txconfirmtarget=<m>``` (default: 2).

Sometimes, it is not possible to give good estimates, or an estimate at all. Therefore, a fallback value can be set with ```-fallbackfee=<f>``` (default: ```0.0002``` BTC/kB).

At all times, VERGE will cap fees at ```-maxtxfee=<x>``` (default: 0.10) BTC. Furthermore, VERGE will never create transactions paying less than the current minimum relay fee. Finally, a user can set the minimum fee rate for all transactions with ```-mintxfee=<i>```, which defaults to 1000 satoshis per kB.



## Wallet: Negative confirmations and conflict detection

The wallet will now report a negative number for confirmations that indicates how deep in the block chain the conflict is found. For example, if a transaction A has 5 confirmations and spends the same input as a wallet transaction B, B will be reported as having -5 confirmations. If another wallet transaction C spends an output from B, it will also be reported as having -5 confirmations. To detect conflicts with historical transactions in the chain a one-time ```-rescan``` may be needed.

Unlike earlier versions, unconfirmed but non-conflicting transactions will never get a negative confirmation count. They are not treated as spendable unless they’re coming from ourself (change) and accepted into our local mempool, however. The new “trusted” field in the ```listtransactions``` RPC output indicates whether outputs of an unconfirmed transaction are considered spendable.



## Wallet: Merkle branches removed

Previously, every wallet transaction stored a Merkle branch to prove its presence in blocks. This wasn’t being used for more than an expensive sanity check. Since 0.12, these are no longer stored. When loading a 0.12 wallet into an older version, it will automatically rescan to avoid failed checks.



## Wallet: Pruning

With 0.12 it is possible to use wallet functionality in pruned mode. This can reduce the disk usage from currently around 60 GB to around 2 GB.

However, rescans as well as the RPCs ```importwallet```, ```importaddress```, ```importprivkey``` are disabled.

To enable block pruning set ```prune=<N>``` on the command line or in ```verge.conf```, where N is the number of MiB to allot for raw block & undo data.

A value of 0 disables pruning. The minimal value above 0 is 550. Your wallet is as secure with high values as it is with low ones. Higher values merely ensure that your node will not shut down upon blockchain reorganizations of more than 2 days - which are unlikely to happen in practice. In future releases, a higher value may also help the network as a whole: stored blocks could be served to other nodes.

For further information about pruning, you may also consult the release notes of [v0.11.0](https://github.com/bitcoin/bitcoin/blob/v0.11.0/doc/release-notes.md#block-file-pruning).



## ```NODE_BLOOM``` service bit

Support for the ```NODE_BLOOM``` service bit, as described in [BIP 111](https://github.com/verge/bips/blob/master/bip-0111.mediawiki), has been added to the P2P protocol code.

BIP 111 defines a service bit to allow peers to advertise that they support bloom filters (such as used by SPV clients) explicitly. It also bumps the protocol version to allow peers to identify old nodes which allow bloom filtering of the connection despite lacking the new service bit.

In this version, it is only enforced for peers that send protocol versions ```>=70011```. For the next major version it is planned that this restriction will be removed. It is recommended to update SPV clients to check for the ```NODE_BLOOM``` service bit for nodes that report versions newer than 70011.



## Option parsing behavior

Command line options are now parsed strictly in the order in which they are specified. It used to be the case that ```-X -noX``` ends up, unintuitively, with X set, as ```-X``` had precedence over ```-noX```. This is no longer the case. Like for other software, the last specified value for an option will hold.



## RPC: Low-level API changes

* Monetary amounts can be provided as strings. This means that for example the argument to sendtoaddress can be “0.0001” instead of 0.0001. This can be an advantage if a JSON library insists on using a lossy floating point type for numbers, which would be dangerous for monetary amounts.

* The ```asm``` property of each scriptSig now contains the decoded signature hash type for each signature that provides a valid defined hash type.

* OP_NOP2 has been renamed to OP_CHECKLOCKTIMEVERIFY by BIP 65

The following items contain assembly representations of scriptSig signatures and are affected by this change:

* RPC ```getrawtransaction```
* RPC ```decoderawtransaction```
* RPC ```decodescript```
* REST ```/rest/tx/``` (JSON format)
* REST ```/rest/block/``` (JSON format when including extended tx details)
* ```verge-tx -json```

For example, the scriptSig.asm property of a transaction input that previously showed an assembly representation of:
```
304502207fa7a6d1e0ee81132a269ad84e68d695483745cde8b541e3bf630749894e342a022100c1f7ab20e13e22fb95281a870f3dcf38d782e53023ee313d741ad0cfbc0c509001 400000 OP_NOP2
```
now shows as:
```
304502207fa7a6d1e0ee81132a269ad84e68d695483745cde8b541e3bf630749894e342a022100c1f7ab20e13e22fb95281a870f3dcf38d782e53023ee313d741ad0cfbc0c5090[ALL] 400000 OP_CHECKLOCKTIMEVERIFY
```
Note that the output of the RPC ```decodescript``` did not change because it is configured specifically to process scriptPubKey and not scriptSig scripts.


## RPC: SSL support dropped

SSL support for RPC, previously enabled by the option ```rpcssl``` has been dropped from both the client and the server. This was done in preparation for removing the dependency on OpenSSL for the daemon completely.

Trying to use ```rpcssl``` will result in an error:

```
Error: SSL mode for RPC (-rpcssl) is no longer supported.
```

If you are one of the few people that relies on this feature, a flexible migration path is to use stunnel. This is an utility that can tunnel arbitrary TCP connections inside SSL. On e.g. Ubuntu it can be installed with:

```
sudo apt-get install stunnel4
```

Then, to tunnel a SSL connection on 28332 to a RPC server bound on localhost on port 18332 do:

```
stunnel -d 28332 -r 127.0.0.1:18332 -p stunnel.pem -P ''
```

It can also be set up system-wide in inetd style.

Another way to re-attain SSL would be to setup a httpd reverse proxy. This solution would allow the use of different authentication, loadbalancing, on-the-fly compression and caching. A sample config for apache2 could look like:

```
Listen 443

NameVirtualHost *:443
<VirtualHost *:443>

SSLEngine On
SSLCertificateFile /etc/apache2/ssl/server.crt
SSLCertificateKeyFile /etc/apache2/ssl/server.key

<Location /vergerpc>
    ProxyPass http://127.0.0.1:8332/
    ProxyPassReverse http://127.0.0.1:8332/
    # optional enable digest auth
    # AuthType Digest
    # ...

    # optional bypass verged rpc basic auth
    # RequestHeader set Authorization "Basic <hash>"
    # get the <hash> from the shell with: base64 <<< vergerpc:<password>
</Location>

# Or, balance the load:
# ProxyPass / balancer://balancer_cluster_name

</VirtualHost>
```


## Mining Code Changes

The mining code in 0.12 has been optimized to be significantly faster and use less memory. As part of these changes, consensus critical calculations are cached on a transaction’s acceptance into the mempool and the mining code now relies on the consistency of the mempool to assemble blocks. However all blocks are still tested for validity after assembly.


## Other P2P Changes

The list of banned peers is now stored on disk rather than in memory. Restarting verged will no longer clear out the list of banned peers; instead a new RPC call (```clearbanned```) can be used to manually clear the list. The new ```setban``` RPC call can also be used to manually ban or unban a peer.


## First version bits BIP9 softfork deployment

This release includes a soft fork deployment to enforce BIP68, BIP112 and BIP113 using the BIP9 deployment mechanism.

The deployment sets the block version number to 0x20000001 between midnight 1st May 2016 and midnight 1st May 2017 to signal readiness for deployment. The version number consists of 0x20000000 to indicate version bits together with setting bit 0 to indicate support for this combined deployment, shown as “csv” in the ```getblockchaininfo``` RPC call.

For more information about the soft forking change, please see https://github.com/bitcoin/bitcoin/pull/7648

This specific backport pull-request can be viewed at https://github.com/bitcoin/bitcoin/pull/7543


## BIP68 soft fork to enforce sequence locks for relative locktime

BIP68 introduces relative lock-time consensus-enforced semantics of the sequence number field to enable a signed transaction input to remain invalid for a defined period of time after confirmation of its corresponding outpoint.

For more information about the implementation, see https://github.com/bitcoin/bitcoin/pull/7184


## BIP112 soft fork to enforce OP_CHECKSEQUENCEVERIFY

BIP112 redefines the existing OP_NOP3 as OP_CHECKSEQUENCEVERIFY (CSV) for a new opcode in the verge scripting system that in combination with BIP68 allows execution pathways of a script to be restricted based on the age of the output being spent.

For more information about the implementation, see https://github.com/bitcoin/bitcoin/pull/7524


## BIP113 locktime enforcement soft fork

VERGE 0.11.2 previously introduced mempool-only locktime enforcement using GetMedianTimePast(). This release seeks to consensus enforce the rule.

verge transactions currently may specify a locktime indicating when they may be added to a valid block. Current consensus rules require that blocks have a block header time greater than the locktime specified in any transaction in that block.

Miners get to choose what time they use for their header time, with the consensus rule being that no node will accept a block whose time is more than two hours in the future. This creates a incentive for miners to set their header times to future values in order to include locktimed transactions which weren’t supposed to be included for up to two more hours.

The consensus rules also specify that valid blocks may have a header time greater than that of the median of the 11 previous blocks. This GetMedianTimePast() time has a key feature we generally associate with time: it can’t go backwards.

BIP113 specifies a soft fork enforced in this release that weakens this perverse incentive for individual miners to use a future time by requiring that valid blocks have a computed GetMedianTimePast() greater than the locktime specified in any transaction in that block.

Mempool inclusion rules currently require transactions to be valid for immediate inclusion in a block in order to be accepted into the mempool. This release begins applying the BIP113 rule to received transactions, so transaction whose time is greater than the GetMedianTimePast() will no longer be accepted into the mempool.


#### Implication for miners: 
you will begin rejecting transactions that would not be valid under BIP113, which will prevent you from producing invalid blocks when BIP113 is enforced on the network. Any transactions which are valid under the current rules but not yet valid under the BIP113 rules will either be mined by other miners or delayed until they are valid under BIP113. Note, however, that time-based locktime transactions are more or less unseen on the network currently.


#### Implication for users: 
GetMedianTimePast() always trails behind the current time, so a transaction locktime set to the present time will be rejected by nodes running this release until the median time moves forward. To compensate, subtract one hour (3,600 seconds) from your locktimes to allow those transactions to be included in mempools at approximately the expected time.

For more information about the implementation, see https://github.com/bitcoin/bitcoin/pull/6566


## Miscellaneous

The p2p alert system is off by default. To turn on, use ```-alert``` with startup configuration.


## Database cache memory increased

As a result of growth of the UTXO set, performance with the prior default database cache of 100 MiB has suffered. For this reason the default was changed to 300 MiB in this release.

For nodes on low-memory systems, the database cache can be changed back to 100 MiB (or to another value) by either:

* Adding ```dbcache=100``` in verge.conf

* Changing it in the GUI under ```Options → Size of database cache```

###### Note that the database cache setting has the most performance impact during initial sync of a node, and when catching up after downtime.


## verge-cli: arguments privacy

The RPC command line client gained a new argument, ```-stdin``` to read extra arguments from standard input, one per line until EOF/Ctrl-D. For example:
```
$ src/verge-cli -stdin walletpassphrase
mysecretcode
120
..... press Ctrl-D here to end input
$
```

It is recommended to use this for sensitive information such as wallet passphrases, as command-line arguments can usually be read from the process table by any user on the system.


## C++11 and Python 3

Various code modernizations have been done. The VERGE code base has started using C++11. This means that a C++11-capable compiler is now needed for building. Effectively this means GCC 4.7 or higher, or Clang 3.3 or higher.

When cross-compiling for a target that doesn’t have C++11 libraries, configure with ```./configure --enable-glibc-back-compat ... LDFLAGS=-static-libstdc++.```

For running the functional tests in qa/rpc-tests, Python3.4 or higher is now required.
Linux ARM builds

Due to popular request, Linux ARM builds have been added to the uploaded executables.

The following extra files can be found in the download directory or torrent:

* ```verge-${VERSION}-arm-linux-gnueabihf.tar.gz```: Linux binaries for the most common 32-bit ARM architecture.


* ```verge-${VERSION}-aarch64-linux-gnu.tar.gz```: Linux binaries for the most common 64-bit ARM architecture.


ARM builds are still experimental. If you have problems on a certain device or Linux distribution combination please report them on the bug tracker, it may be possible to resolve them.

Note that Android is not considered ARM Linux in this context. The executables are not expected to work out of the box on Android.


## Compact Block support (BIP 152)

Support for block relay using the Compact Blocks protocol has been implemented in PR 8068.

The primary goal is reducing the bandwidth spikes at relay time, though in many cases it also reduces propagation delay. It is automatically enabled between compatible peers. BIP 152

As a side-effect, ordinary non-mining nodes will download and upload blocks faster if those blocks were produced by miners using similar transaction filtering policies. This means that a miner who produces a block with many transactions discouraged by your node will be relayed slower than one with only transactions already in your memory pool. The overall effect of such relay differences on the network may result in blocks which include widely- discouraged transactions losing a stale block race, and therefore miners may wish to configure their node to take common relay policies into consideration.


## Hierarchical Deterministic Key Generation

Newly created wallets will use hierarchical deterministic key generation according to BIP32 (keypath m/0’/0’/k’). Existing wallets will still use traditional key generation.

Backups of HD wallets, regardless of when they have been created, can therefore be used to re-generate all possible private keys, even the ones which haven’t already been generated during the time of the backup. Attention: Encrypting the wallet will create a new seed which requires a new backup!

Wallet dumps (created using the dumpwallet RPC) will contain the deterministic seed. This is expected to allow future versions to import the seed and all associated funds, but this is not yet implemented.

HD key generation for new wallets can be disabled by -usehd=0. Keep in mind that this flag only has affect on newly created wallets. You can’t disable HD key generation once you have created a HD wallet.

There is no distinction between internal (change) and external keys.

HD wallets are incompatible with older versions of VERGE.


## Segregated Witness

The code preparations for Segregated Witness (“segwit”), as described in BIP 141, BIP 143, BIP 144, and BIP 145 are finished and included in this release. However, BIP 141 does not yet specify activation parameters on mainnet, and so this release does not support segwit use on mainnet. Testnet use is supported, and after BIP 141 is updated with proposed parameters, a future release of VERGE is expected that implements those parameters for mainnet.

Furthermore, because segwit activation is not yet specified for mainnet, version 0.13.0 will behave similarly as other pre-segwit releases even after a future activation of BIP 141 on the network. Upgrading from 0.13.0 will be required in order to utilize segwit-related features on mainnet (such as signal BIP 141 activation, mine segwit blocks, fully validate segwit blocks, relay segwit blocks to other segwit nodes, and use segwit transactions in the wallet, etc).


## Mining transaction selection (“Child Pays For Parent”)

The mining transaction selection algorithm has been replaced with an algorithm that selects transactions based on their feerate inclusive of unconfirmed ancestor transactions. This means that a low-fee transaction can become more likely to be selected if a high-fee transaction that spends its outputs is relayed.

With this change, the ```-blockminsize``` command line option has been removed.

The command line option ```-blockmaxsize``` remains an option to specify the maximum number of serialized bytes in a generated block. In addition, the new command line option ```-blockmaxweight``` has been added, which specifies the maximum “block weight” of a generated block, as defined by [BIP 141 (Segregated Witness)](https://github.com/verge/bips/blob/master/bip-0141.mediawiki).

In preparation for Segregated Witness, the mining algorithm has been modified to optimize transaction selection for a given block weight, rather than a given number of serialized bytes in a block. In this release, transaction selection is unaffected by this distinction (as BIP 141 activation is not supported on mainnet in this release, see above), but in future releases and after BIP 141 activation, these calculations would be expected to differ.

For optimal runtime performance, miners using this release should specify ```-blockmaxweight``` on the command line, and not specify ```-blockmaxsize```. Additionally (or only) specifying ```-blockmaxsize```, or relying on default settings for both, may result in performance degradation, as the logic to support ```-blockmaxsize``` performs additional computation to ensure that constraint is met. (Note that for mainnet, in this release, the equivalent parameter for ```-blockmaxweight``` would be four times the desired ```-blockmaxsize```. See [BIP 141](https://github.com/verge/bips/blob/master/bip-0141.mediawiki) for additional details.)

In the future, the ```-blockmaxsize``` option may be removed, as block creation is no longer optimized for this metric. Feedback is requested on whether to deprecate or keep this command line option in future releases.


## Reindexing changes

In earlier versions, reindexing did validation while reading through the block files on disk. These two have now been split up, so that all blocks are known before validation starts. This was necessary to make certain optimizations that are available during normal synchronizations also available during reindexing.

The two phases are distinct in the verge-Qt GUI. During the first one, “Reindexing blocks on disk” is shown. During the second (slower) one, “Processing blocks on disk” is shown.

It is possible to only redo validation now, without rebuilding the block index, using the command line option ```-reindex-chainstate``` (in addition to ```-reindex```which does both). This new option is useful when the blocks on disk are assumed to be fine, but the chainstate is still corrupted. It is also useful for benchmarks.
Removal of internal miner

As CPU mining has been useless for a long time, the internal miner has been removed in this release, and replaced with a simpler implementation for the test framework.

The overall result of this is that ```setgenerate``` RPC call has been removed, as well as the ```-gen``` and ```-genproclimit``` command-line options.

For testing, the ```generate``` call can still be used to mine a block, and a new RPC call ```generatetoaddress``` has been added to mine to a specific address. This works with wallet disabled.


## New bytespersigop implementation

The former implementation of the bytespersigop filter accidentally broke bare multisig (which is meant to be controlled by the ```permitbaremultisig``` option), since the consensus protocol always counts these older transaction forms as 20 sigops for backwards compatibility. Simply fixing this bug by counting more accurately would have reintroduced a vulnerability. It has therefore been replaced with a new implementation that rather than filter such transactions, instead treats them (for fee purposes only) as if they were in fact the size of a transaction actually using all 20 sigops.


## Low-level P2P changes

* The optional new p2p message “feefilter” is implemented and the protocol version is bumped to 70013. Upon receiving a feefilter message from a peer, a node will not send invs for any transactions which do not meet the filter feerate. BIP 133

* The P2P alert system has been removed in PR #7692 and the alert P2P message is no longer supported.

* The transaction relay mechanism used to relay one quarter of all transactions instantly, while queueing up the rest and sending them out in batch. As this resulted in chains of dependent transactions being reordered, it systematically hurt transaction relay. The relay code was redesigned in PRs <a href=”https://github.com/bitcoin/bitcoin/pull/7840”>#7840</a> and #8082, and now always batches transactions announcements while also sorting them according to dependency order. This significantly reduces orphan transactions. To compensate for the removal of instant relay, the frequency of batch sending was doubled for outgoing peers.

* Since PR #7840 the BIP35 mempool command is also subject to batch processing. Also the mempool message is no longer handled for non-whitelisted peers when NODE_BLOOM is disabled through -peerbloomfilters=0.

* The maximum size of orphan transactions that are kept in memory until their ancestors arrive has been raised in PR #8179 from 5000 to 99999 bytes. They are now also removed from memory when they are included in a block, conflict with a block, and time out after 20 minutes.

* We respond at most once to a getaddr request during the lifetime of a connection since PR #7856.

* Connections to peers who have recently been the first one to give us a valid new block or transaction are protected from disconnections since PR #8084.


## Low-level RPC changes

* RPC calls have been added to output detailed statistics for individual mempool entries, as well as to calculate the in-mempool ancestors or descendants of a transaction: see getmempoolentry, getmempoolancestors, getmempooldescendants.

* gettxoutsetinfo UTXO hash (hash_serialized) has changed. There was a divergence between 32-bit and 64-bit platforms, and the txids were missing in the hashed data. This has been fixed, but this means that the output will be different than from previous versions.

* Full UTF-8 support in the RPC API. Non-ASCII characters in, for example, wallet labels have always been malformed because they weren’t taken into account properly in JSON RPC processing. This is no longer the case. This also affects the GUI debug console.

* Asm script outputs replacements for OP_NOP2 and OP_NOP3

    * OP_NOP2 has been renamed to OP_CHECKLOCKTIMEVERIFY by BIP 65

    * OP_NOP3 has been renamed to OP_CHECKSEQUENCEVERIFY by BIP 112

    * The following outputs are affected by this change:
        * RPC ```getrawtransaction``` (in verbose mode)
        * RPC ```decoderawtransaction```
        * RPC ```decodescript```
        * REST ```/rest/tx/``` (JSON format)
        * REST ```/rest/block/``` (JSON format when including extended tx details)
        * ```verge-tx -json```

* The sorting of the output of the ```getrawmempool``` output has changed.

* New RPC commands: ```generatetoaddress```, ```importprunedfunds```, ```removeprunedfunds```, ```signmessagewithprivkey```, ```getmempoolancestors```, ```getmempooldescendants```, ```getmempoolentry```, ```createwitnessaddress```, ```addwitnessaddress```.

* Removed RPC commands: ```setgenerate```, ```getgenerate```.

* New options were added to fundrawtransaction: ```includeWatching```, ```changeAddress```, ```changePosition``` and ```feeRate```.


## Low-level ZMQ changes

* Each ZMQ notification now contains an up-counting sequence number that allows listeners to detect lost notifications. The sequence number is always the last element in a multi-part ZMQ notification and therefore backward compatible. Each message type has its own counter. PR #7762.

## Change to wallet handling of mempool rejection

When a newly created transaction failed to enter the mempool due to the limits on chains of unconfirmed transactions the sending RPC calls would return an error. The transaction would still be queued in the wallet and, once some of the parent transactions were confirmed, broadcast after the software was restarted.

This behavior has been changed to return success and to reattempt mempool insertion at the same time transaction rebroadcast is attempted, avoiding a need for a restart.

Transactions in the wallet which cannot be accepted into the mempool can be abandoned with the previously existing abandontransaction RPC (or in the GUI via a context menu on the transaction).


## Performance Improvements

A number of significant performance improvements, which make Initial Block Download, startup, transaction and block validation much faster. Additionally, Validation speed and network propagation performance have been greatly
improved, leading to much shorter sync and initial block download times.

- The script signature cache has been reimplemented as a "cuckoo cache",
  allowing for more signatures to be cached and faster lookups.

- Assumed-valid blocks have been introduced which allows script validation to
  be skipped for ancestors of known-good blocks, without changing the security
  model. See below for more details.

- In some cases, compact blocks are now relayed before being fully validated as
  per BIP152.

- P2P networking has been refactored with a focus on concurrency and
  throughput. Network operations are no longer bottlenecked by validation. As a
  result, block fetching is several times faster than previous releases in many
  cases.

- The UTXO cache now claims unused mempool memory. This speeds up initial block
  download as UTXO lookups are a major bottleneck there, and there is no use for
  the mempool at that stage.

- The chainstate database (which is used for tracking UTXOs) has been changed
  from a per-transaction model to a per-output model (See [PR 10195](https://github.com/bitcoin/bitcoin/pull/10195)). Advantages of this model
  are that it:
    - avoids the CPU overhead of deserializing and serializing the unused outputs;
    - has more predictable memory usage;
    - uses simpler code;
    - is adaptable to various future cache flushing strategies.

  As a result, validating the blockchain during Initial Block Download (IBD) and reindex
  is ~30-40% faster, uses 10-20% less memory, and flushes to disk far less frequently.
  The only downside is that the on-disk database is 15% larger. During the conversion from the previous format a few extra gigabytes may be used.

- Earlier versions experienced a spike in memory usage while flushing UTXO updates to disk.
  As a result, only half of the available memory was actually used as cache, and the other half was
  reserved to accommodate flushing. This is no longer the case (See [PR 10148](https://github.com/bitcoin/bitcoin/pull/10148)), and the entirety of
  the available cache (see `-dbcache`) is now actually used as cache. This reduces the flushing
  frequency by a factor 2 or more.

- In previous versions, signature validation for transactions has been cached when the
  transaction is accepted to the mempool. Version 0.15 extends this to cache the entire script
  validity (See [PR 10192](https://github.com/bitcoin/bitcoin/pull/10192)). This means that if a transaction in a block has already been accepted to the
  mempool, the scriptSig does not need to be re-evaluated. Empirical tests show that
  this results in new block validation being 40-50% faster.

- LevelDB has been upgraded to version 1.20 (See [PR 10544](https://github.com/bitcoin/bitcoin/pull/10544)). This version contains hardware acceleration for CRC
  on architectures supporting SSE 4.2. As a result, synchronization and block validation are now faster.

- Refill of the keypool no longer flushes the wallet between each key which resulted in a ~20x speedup in creating a new wallet. Part of this speedup was used to increase the default keypool to 1000 keys to make recovery more robust. (See [PR 10831](https://github.com/bitcoin/bitcoin/pull/10831)).


## Manual Pruning

Manual block pruning can now be enabled by setting `-prune=1`. Once that is set, the RPC command `pruneblockchain` can be used to prune the blockchain up to the specified height or timestamp.


## `getinfo` Deprecated


The `getinfo` RPC command has been deprecated. Each field in the RPC call has been moved to another command's output with that command also giving additional information that `getinfo` did not provide. The following table shows where each field has been moved to:

|`getinfo` field   | Moved to                                  |
|------------------|-------------------------------------------|
`"version"`	   | `getnetworkinfo()["version"]`
`"protocolversion"`| `getnetworkinfo()["protocolversion"]`
`"walletversion"`  | `getwalletinfo()["walletversion"]`
`"balance"`	   | `getwalletinfo()["balance"]`
`"blocks"`	   | `getblockchaininfo()["blocks"]`
`"timeoffset"`	   | `getnetworkinfo()["timeoffset"]`
`"connections"`	   | `getnetworkinfo()["connections"]`
`"proxy"`	   | `getnetworkinfo()["networks"][0]["proxy"]`
`"difficulty"`	   | `getblockchaininfo()["difficulty"]`
`"testnet"`	   | `getblockchaininfo()["chain"] == "test"`
`"keypoololdest"`  | `getwalletinfo()["keypoololdest"]`
`"keypoolsize"`	   | `getwalletinfo()["keypoolsize"]`
`"unlocked_until"` | `getwalletinfo()["unlocked_until"]`
`"paytxfee"`	   | `getwalletinfo()["paytxfee"]`
`"relayfee"`	   | `getnetworkinfo()["relayfee"]`
`"errors"`	   | `getnetworkinfo()["warnings"]`

## ZMQ On Windows

Previously the ZeroMQ notification system was unavailable on Windows due to various issues with ZMQ. These have been fixed upstream and now ZMQ can be used on Windows. Please see [this document](https://github.com/bitcoin/bitcoin/blob/master/doc/zmq.md) for help with using ZMQ in general.

## Nested RPC Commands in Debug Console

The ability to nest RPC commands has been added to the debug console. This allows users to have the output of a command become the input to another command without running the commands separately.

The nested RPC commands use bracket syntax (i.e. `getwalletinfo()`) and can be nested (i.e. `getblock(getblockhash(1))`). Simple queries can be done with square brackets where object values are accessed with either an array index or a non-quoted string (i.e. `listunspent()[0][txid]`). Both commas and spaces can be used to separate parameters in both the bracket syntax and normal RPC command syntax.

## Network Activity Toggle

A RPC command and GUI toggle have been added to enable or disable all p2p network activity. The network status icon in the bottom right hand corner is now the GUI toggle. Clicking the icon will either enable or disable all
p2p network activity. If network activity is disabled, the icon will be grayed out with an X on top of it.

Additionally the `setnetworkactive` RPC command has been added which does the same thing as the GUI icon. The command takes one boolean parameter, `true` enables networking and `false` disables it.


## Support for JSON-RPC Named Arguments

Commands sent over the JSON-RPC interface and through the `verge-cli` binary can now use named arguments. This follows the [JSON-RPC specification](http://www.jsonrpc.org/specification) for passing parameters by-name with an object.

`verge-cli` has been updated to support this by parsing `name=value` arguments when the `-named` option is given.

Some examples:

    src/verge-cli -named help command="help"
    src/verge-cli -named getblockhash height=0
    src/verge-cli -named getblock blockhash=000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
    src/verge-cli -named sendtoaddress address="(snip)" amount="1.0" subtractfeefromamount=true

The order of arguments doesn't matter in this case. Named arguments are also useful to leave out arguments that should stay at their default value. The rarely-used arguments `comment` and `comment_to` to `sendtoaddress`, for example, can be left out. However, this is not yet implemented for many RPC calls, this is expected to land in a later release.

The RPC server remains fully backwards compatible with positional arguments.

## Opt into RBF When Sending

A new startup option, `-walletrbf`, has been added to allow users to have all transactions sent opt into RBF support. The default value for this option is
currently `false`, so transactions will not opt into RBF by default. The new `bumpfee` RPC can be used to replace transactions that opt into RBF.

## Sensitive Data Is No Longer Stored In Debug Console History

The debug console maintains a history of previously entered commands that can be accessed by pressing the Up-arrow key so that users can easily reuse previously entered commands. Commands which have sensitive information such as passphrases and private keys will now have a `(...)` in place of the parameters when accessed through the history.

## Retaining the Mempool Across Restarts

The mempool will be saved to the data directory prior to shutdown to a `mempool.dat` file. This file preserves the mempool so that when the noderestarts the mempool can be filled with transactions without waiting for new transactions
to be created. This will also preserve any changes made to a transaction through commands such as `prioritisetransaction` so that those changes will not be lost.

## Low-level RPC changes


 - `importprunedfunds` only accepts two required arguments. Some versions accept
   an optional third arg, which was always ignored. Make sure to never pass more
   than two arguments.

 - The first boolean argument to `getaddednodeinfo` has been removed. This is 
   an incompatible change.

 - RPC command `getmininginfo` loses the "testnet" field in favor of the more
   generic "chain" (which has been present for years).

 - A new RPC command `preciousblock` has been added which marks a block as
   precious. A precious block will be treated as if it were received earlier
   than a competing block.

 - A new RPC command `importmulti` has been added which receives an array of 
   JSON objects representing the intention of importing a public key, a 
   private key, an address and script/p2sh

 - Use of `getrawtransaction` for retrieving confirmed transactions with unspent
   outputs has been deprecated. For now this will still work, but in the future
   it may change to only be able to retrieve information about transactions in
   the mempool or if `txindex` is enabled.

 - A new RPC command `getmemoryinfo` has been added which will return information
   about the memory usage of VERGE. This was added in conjunction with
   optimizations to memory management. See [Pull #8753](https://github.com/bitcoin/bitcoin/pull/8753)
   for more information.

 - A new RPC command `bumpfee` has been added which allows replacing an
   unconfirmed wallet transaction that signaled RBF (see the `-walletrbf`
   startup option above) with a new transaction that pays a higher fee, and
   should be more likely to get confirmed quickly.

## HTTP REST Changes

 - UTXO set query (`GET /rest/getutxos/<checkmempool>/<txid>-<n>/<txid>-<n>
   /.../<txid>-<n>.<bin|hex|json>`) responses were changed to return status 
   code `HTTP_BAD_REQUEST` (400) instead of `HTTP_INTERNAL_SERVER_ERROR` (500)
   when requests contain invalid parameters.

## Minimum Fee Rate Policies

Since the changes in 0.12 to automatically limit the size of the mempool and improve the performance of block creation in mining code it has not been important for relay nodes or miners to set `-minrelaytxfee`. With this release the following concepts that were tied to this option have been separated out:
- incremental relay fee used for calculating BIP 125 replacement and mempool limiting. (1000 satoshis/kB)
- calculation of threshold for a dust output. (effectively 3 * 1000 satoshis/kB)
- minimum fee rate of a package of transactions to be included in a block created by the mining code. If miners wish to set this minimum they can use the new `-blockmintxfee` option.  (defaults to 1000 satoshis/kB)

The `-minrelaytxfee` option continues to exist but is recommended to be left unset.

## Fee Estimation Changes


- Since 0.13.2 fee estimation for a confirmation target of 1 block has been
  disabled. The fee slider will no longer be able to choose a target of 1 block.
  This is only a minor behavior change as there was often insufficient
  data for this target anyway. `estimatefee 1` will now always return -1 and
  `estimatesmartfee 1` will start searching at a target of 2.

- The default target for fee estimation is changed to 6 blocks in both the GUI
  (previously 25) and for RPC calls (previously 2).

## Removal of Priority Estimation

- Estimation of "priority" needed for a transaction to be included within a target
  number of blocks has been removed.  The RPC calls are deprecated and will either
  return -1 or 1e24 appropriately. The format for `fee_estimates.dat` has also
  changed to no longer save these priority estimates. It will automatically be
  converted to the new format which is not readable by prior versions of the
  software.

- Support for "priority" (coin age) transaction sorting for mining is
  considered deprecated in Core and will be removed in the next major version.
  This is not to be confused with the `prioritisetransaction` RPC which will remain
  supported by Core for adding fee deltas to transactions.

## P2P connection management


- Peers manually added through the `-addnode` option or `addnode` RPC now have their own
  limit of eight connections which does not compete with other inbound or outbound
  connection usage and is not subject to the limitation imposed by the `-maxconnections`
  option.

- New connections to manually added peers are performed more quickly.

## Introduction of assumed-valid blocks

- A significant portion of the initial block download time is spent verifying
  scripts/signatures.  Although the verification must pass to ensure the security
  of the system, no other result from this verification is needed: If the node
  knew the history of a given block were valid it could skip checking scripts
  for its ancestors.

- A new configuration option 'assumevalid' is provided to express this knowledge
  to the software.  Unlike the 'checkpoints' in the past this setting does not
  force the use of a particular chain: chains that are consistent with it are
  processed quicker, but other chains are still accepted if they'd otherwise
  be chosen as best. Also unlike 'checkpoints' the user can configure which
  block history is assumed true, this means that even outdated software can
  sync more quickly if the setting is updated by the user.

- Because the validity of a chain history is a simple objective fact it is much
  easier to review this setting.  As a result the software ships with a default
  value adjusted to match the current chain shortly before release.  The use
  of this default value can be disabled by setting -assumevalid=0

## Fundrawtransaction change address reuse

- Before 0.14, `fundrawtransaction` was by default wallet stateless. In
  almost all cases `fundrawtransaction` does add a change-output to the
  outputs of the funded transaction. Before 0.14, the used keypool key was
  never marked as change-address key and directly returned to the keypool
  (leading to address reuse).  Before 0.14, calling `getnewaddress`
  directly after `fundrawtransaction` did generate the same address as
  the change-output address.

- Since 0.14, fundrawtransaction does reserve the change-output-key from
  the keypool by default (optional by setting  `reserveChangeKey`, default =
  `true`)

- Users should also consider using `getrawchangeaddress()` in conjunction
  with `fundrawtransaction`'s `changeAddress` option.

## Unused mempool memory used by coincache

- Before 0.14, memory reserved for mempool (using the `-maxmempool` option)
  went unused during initial block download, or IBD. In 0.14, the UTXO DB cache
  (controlled with the `-dbcache` option) borrows memory from the mempool
  when there is extra memory available. This may result in an increase in
  memory usage during IBD for those previously relying on only the `-dbcache`
  option to limit memory during that time.

  RPC changes
-----------

- The first positional argument of `createrawtransaction` was renamed from
  `transactions` to `inputs`.

- The argument of `disconnectnode` was renamed from `node` to `address`.

These interface changes break compatibility with 0.14.0, when the named
arguments functionality, introduced in 0.14.0, is used. Client software
using these calls with named arguments needs to be updated.

## UTXO memory accounting

Memory usage for the UTXO cache is being calculated more accurately, so that
the configured limit (`-dbcache`) will be respected when memory usage peaks
during cache flushes.  The memory accounting in prior releases is estimated to
only account for half the actual peak utilization.

The default `-dbcache` has also been changed in this release to 450MiB.  Users
who currently set `-dbcache` to a high value (e.g. to keep the UTXO more fully
cached in memory) should consider increasing this setting in order to achieve
the same cache performance as prior releases.  Users on low-memory systems
(such as systems with 1GB or less) should consider specifying a lower value for
this parameter.

Additional information relating to running on low-memory systems can be found
here:
[reducing-verged-memory-usage.md](https://gist.github.com/laanwj/efe29c7661ce9b6620a7).

## miniupnp CVE-2017-8798

Bundled miniupnpc was updated to 2.0.20170509. This fixes an integer signedness error
(present in MiniUPnPc v1.4.20101221 through v2.0) that allows remote attackers
(within the LAN) to cause a denial of service or possibly have unspecified
other impact.

This only affects users that have explicitly enabled UPnP through the GUI
setting or through the `-upnp` option, as since the last UPnP vulnerability
(in VERGE 0.10.3) it has been disabled by default.

If you use this option, it is recommended to upgrade to this version as soon as
possible.

## Fee Estimation Improvements

Fee estimation has been significantly improved in version 0.15, with more accurate fee estimates used by the wallet and a wider range of options for advanced users of the `estimatesmartfee` and `estimaterawfee` RPCs (See [PR 10199](https://github.com/bitcoin/bitcoin/pull/10199)).

### Changes to internal logic and wallet behavior

- Internally, estimates are now tracked on 3 different time horizons. This allows for longer targets and means estimates adjust more quickly to changes in conditions.
- Estimates can now be *conservative* or *economical*. *Conservative* estimates use longer time horizons to produce an estimate which is less susceptible to rapid changes in fee conditions. *Economical* estimates use shorter time horizons and will be more affected by short-term changes in fee conditions. Economical estimates may be considerably lower during periods of low transaction activity (for example over weekends), but may result in transactions being unconfirmed if prevailing fees increase rapidly.
- By default, the wallet will use conservative fee estimates to increase the reliability of transactions being confirmed within the desired target. For transactions that are marked as replaceable, the wallet will use an economical estimate by default, since the fee can be 'bumped' if the fee conditions change rapidly (See [PR 10589](https://github.com/bitcoin/bitcoin/pull/10589)).
- Estimates can now be made for confirmation targets up to 1008 blocks (one week).
- More data on historical fee rates is stored, leading to more precise fee estimates.
- Transactions which leave the mempool due to eviction or other non-confirmed reasons are now taken into account by the fee estimation logic, leading to more accurate fee estimates.
- The fee estimation logic will make sure enough data has been gathered to return a meaningful estimate. If there is insufficient data, a fallback default fee is used.

### Changes to fee estimate RPCs

- The `estimatefee` RPC is now deprecated in favor of using only `estimatesmartfee` (which is the implementation used by the GUI)
- The `estimatesmartfee` RPC interface has been changed (See [PR 10707](https://github.com/bitcoin/bitcoin/pull/10707)):
    - The `nblocks` argument has been renamed to `conf_target` (to be consistent with other RPC methods).
    - An `estimate_mode` argument has been added. This argument takes one of the following strings: `CONSERVATIVE`, `ECONOMICAL` or `UNSET` (which defaults to `CONSERVATIVE`).
    - The RPC return object now contains an `errors` member, which returns errors encountered during processing.
    - If VERGE has not been running for long enough and has not seen enough blocks or transactions to produce an accurate fee estimation, an error will be returned (previously a value of -1 was used to indicate an error, which could be confused for a feerate).
- A new `estimaterawfee` RPC is added to provide raw fee data. External clients can query and use this data in their own fee estimation logic.

## Multi-wallet support

VERGE now supports loading multiple, separate wallets (See [PR 8694](https://github.com/bitcoin/bitcoin/pull/8694), [PR 10849](https://github.com/bitcoin/bitcoin/pull/10849)). The wallets are completely separated, with individual balances, keys and received transactions.

Multi-wallet is enabled by using more than one `-wallet` argument when starting verge, either on the command line or in the verge config file.

**In verge-Qt, only the first wallet will be displayed and accessible for creating and signing transactions.** GUI selectable multiple wallets will be supported in a future version. However, even in 0.15 other loaded wallets will remain synchronized to the node's current tip in the background. This can be useful if running a pruned node, since loading a wallet where the most recent sync is beyond the pruned height results in having to download and revalidate the whole blockchain. Continuing to synchronize all wallets in the background avoids this problem.

Note the following changes to the RPC interface and `verge-cli` for multi-wallet:

* When running VERGE with a single wallet, there are **no** changes to the RPC interface or `verge-cli`. All RPC calls and `verge-cli` commands continue to work as before.
* When running VERGE with multi-wallet, all *node-level* RPC methods continue to work as before. HTTP RPC requests should be send to the normal `<RPC IP address>:<RPC port>` endpoint, and `verge-cli` commands should be run as before. A *node-level* RPC method is any method which does not require access to the wallet.
* When running VERGE with multi-wallet, *wallet-level* RPC methods must specify the wallet for which they're intended in every request. HTTP RPC requests should be send to the `<RPC IP address>:<RPC port>/wallet/<wallet name>` endpoint, for example `127.0.0.1:8332/wallet/wallet1.dat`. `verge-cli` commands should be run with a `-rpcwallet` option, for example `verge-cli -rpcwallet=wallet1.dat getbalance`.
* A new *node-level* `listwallets` RPC method is added to display which wallets are currently loaded. The names returned by this method are the same as those used in the HTTP endpoint and for the `rpcwallet` argument.

Note that while multi-wallet is now fully supported, the RPC multi-wallet interface should be considered unstable for version 0.15.0, and there may backwards-incompatible changes in future versions.

## Replace-by-fee control in the GUI

VERGE has supported creating opt-in replace-by-fee (RBF) transactions
since version 0.12.0, and since version 0.14.0 has included a `bumpfee` RPC method to
replace unconfirmed opt-in RBF transactions with a new transaction that pays
a higher fee.

In version 0.15, creating an opt-in RBF transaction and replacing the unconfirmed
transaction with a higher-fee transaction are both supported in the GUI (See [PR 9592](https://github.com/bitcoin/bitcoin/pull/9592)).

## Removal of Coin Age Priority

In previous versions of VERGE, a portion of each block could be reserved for transactions based on the age and value of UTXOs they spent. This concept (Coin Age Priority) is a policy choice by miners, and there are no consensus rules around the inclusion of Coin Age Priority transactions in blocks. In practice, only a few miners continue to use Coin Age Priority for transaction selection in blocks. VERGE 0.15 removes all remaining support for Coin Age Priority (See [PR 9602](https://github.com/bitcoin/bitcoin/pull/9602)). This has the following implications:

- The concept of *free transactions* has been removed. High Coin Age Priority transactions would previously be allowed to be relayed even if they didn't attach a miner fee. This is no longer possible since there is no concept of Coin Age Priority. The `-limitfreerelay` and `-relaypriority` options which controlled relay of free transactions have therefore been removed.
- The `-sendfreetransactions` option has been removed, since almost all miners do not include transactions which do not attach a transaction fee.
- The `-blockprioritysize` option has been removed.
- The `estimatepriority` and `estimatesmartpriority` RPCs have been removed.
- The `getmempoolancestors`, `getmempooldescendants`, `getmempoolentry` and `getrawmempool` RPCs no longer return `startingpriority` and `currentpriority`.
- The `prioritisetransaction` RPC no longer takes a `priority_delta` argument, which is replaced by a `dummy` argument for backwards compatibility with clients using positional arguments. The RPC is still used to change the apparent fee-rate of the transaction by using the `fee_delta` argument.
- `-minrelaytxfee` can now be set to 0. If `minrelaytxfee` is set, then fees smaller than `minrelaytxfee` (per kB) are rejected from relaying, mining and transaction creation. This defaults to 1000 satoshi/kB.
- The `-printpriority` option has been updated to only output the fee rate and hash of transactions included in a block by the mining code.

## Mempool Persistence Across Restarts

Version 0.14 introduced mempool persistence across restarts (the mempool is saved to a `mempool.dat` file in the data directory prior to shutdown and restores the mempool when the node is restarted). Version 0.15 allows this feature to be switched on or off using the `-persistmempool` command-line option (See [PR 9966](https://github.com/bitcoin/bitcoin/pull/9966)). By default, the option is set to true, and the mempool is saved on shutdown and reloaded on startup. If set to false, the `mempool.dat` file will not be loaded on startup or saved on shutdown.

## New RPC methods

Version 0.15 introduces several new RPC methods:

- `abortrescan` stops current wallet rescan, e.g. when triggered by an `importprivkey` call (See [PR 10208](https://github.com/bitcoin/bitcoin/pull/10208)).
- `combinerawtransaction` accepts a JSON array of raw transactions and combines them into a single raw transaction (See [PR 10571](https://github.com/bitcoin/bitcoin/pull/10571)).
- `estimaterawfee` returns raw fee data so that customized logic can be implemented to analyze the data and calculate estimates. See [Fee Estimation Improvements](#fee-estimation-improvements) for full details on changes to the fee estimation logic and interface.
- `getchaintxstats` returns statistics about the total number and rate of transactions
  in the chain (See [PR 9733](https://github.com/bitcoin/bitcoin/pull/9733)).
- `listwallets` lists wallets which are currently loaded. See the *Multi-wallet* section
  of these release notes for full details (See [Multi-wallet support](#multi-wallet-support)).
- `uptime` returns the total runtime of the `verged` server since its last start (See [PR 10400](https://github.com/bitcoin/bitcoin/pull/10400)).

## Low-level RPC changes

- When using VERGE in multi-wallet mode, RPC requests for wallet methods must specify
  the wallet that they're intended for. See [Multi-wallet support](#multi-wallet-support) for full details.

- The new database model no longer stores information about transaction
  versions of unspent outputs (See [Performance improvements](#performance-improvements)). This means that:
  - The `gettxout` RPC no longer has a `version` field in the response.
  - The `gettxoutsetinfo` RPC reports `hash_serialized_2` instead of `hash_serialized`,
    which does not commit to the transaction versions of unspent outputs, but does
    commit to the height and coinbase information.
  - The `getutxos` REST path no longer reports the `txvers` field in JSON format,
    and always reports 0 for transaction versions in the binary format

- The `estimatefee` RPC is deprecated. Clients should switch to using the `estimatesmartfee` RPC, which returns better fee estimates. See [Fee Estimation Improvements](#fee-estimation-improvements) for full details on changes to the fee estimation logic and interface.

- The `gettxoutsetinfo` response now contains `disk_size` and `bogosize` instead of
  `bytes_serialized`. The first is a more accurate estimate of actual disk usage, but
  is not deterministic. The second is unrelated to disk usage, but is a
  database-independent metric of UTXO set size: it counts every UTXO entry as 50 + the
  length of its scriptPubKey (See [PR 10426](https://github.com/bitcoin/bitcoin/pull/10426)).

- `signrawtransaction` can no longer be used to combine multiple transactions into a single transaction. Instead, use the new `combinerawtransaction` RPC (See [PR 10571](https://github.com/bitcoin/bitcoin/pull/10571)).

- `fundrawtransaction` no longer accepts a `reserveChangeKey` option. This option used to allow RPC users to fund a raw transaction using an key from the keypool for the change address without removing it from the available keys in the keypool. The key could then be re-used for a `getnewaddress` call, which could potentially result in confusing or dangerous behaviour (See [PR 10784](https://github.com/bitcoin/bitcoin/pull/10784)).

- `estimatepriority` and `estimatesmartpriority` have been removed. See [Removal of Coin Age Priority](#removal-of-coin-age-priority).

- The `listunspent` RPC now takes a `query_options` argument (see [PR 8952](https://github.com/bitcoin/bitcoin/pull/8952)), which is a JSON object
  containing one or more of the following members:
  - `minimumAmount` - a number specifying the minimum value of each UTXO
  - `maximumAmount` - a number specifying the maximum value of each UTXO
  - `maximumCount` - a number specifying the minimum number of UTXOs
  - `minimumSumAmount` - a number specifying the minimum sum value of all UTXOs

- The `getmempoolancestors`, `getmempooldescendants`, `getmempoolentry` and `getrawmempool` RPCs no longer return `startingpriority` and `currentpriority`. See [Removal of Coin Age Priority](#removal-of-coin-age-priority).

- The `dumpwallet` RPC now returns the full absolute path to the dumped wallet. It
  used to return no value, even if successful (See [PR 9740](https://github.com/bitcoin/bitcoin/pull/9740)).

- In the `getpeerinfo` RPC, the return object for each peer now returns an `addrbind` member, which contains the ip address and port of the connection to the peer. This is in addition to the `addrlocal` member which contains the ip address and port of the local node as reported by the peer (See [PR 10478](https://github.com/bitcoin/bitcoin/pull/10478)).

- The `disconnectnode` RPC can now disconnect a node specified by node ID (as well as by IP address/port). To disconnect a node based on node ID, call the RPC with the new `nodeid` argument (See [PR 10143](https://github.com/bitcoin/bitcoin/pull/10143)).

- The second argument in `prioritisetransaction` has been renamed from `priority_delta` to `dummy` since VERGE no longer has a concept of coin age priority. The `dummy` argument has no functional effect, but is retained for positional argument compatibility. See [Removal of Coin Age Priority](#removal-of-coin-age-priority).

- The `resendwallettransactions` RPC throws an error if the `-walletbroadcast` option is set to false (See [PR 10995](https://github.com/bitcoin/bitcoin/pull/10995)).

- The second argument in the `submitblock` RPC argument has been renamed from `parameters` to `dummy`. This argument never had any effect, and the renaming is simply to communicate this fact to the user (See [PR 10191](https://github.com/bitcoin/bitcoin/pull/10191))
  (Clients should, however, use positional arguments for `submitblock` in order to be compatible with BIP 22.)

- The `verbose` argument of `getblock` has been renamed to `verbosity` and now takes an integer from 0 to 2. Verbose level 0 is equivalent to `verbose=false`. Verbose level 1 is equivalent to `verbose=true`. Verbose level 2 will give the full transaction details of each transaction in the output as given by `getrawtransaction`. The old behavior of using the `verbose` named argument and a boolean value is still maintained for compatibility.

- Error codes have been updated to be more accurate for the following error cases (See [PR 9853](https://github.com/bitcoin/bitcoin/pull/9853)):
  - `getblock` now returns RPC_MISC_ERROR if the block can't be found on disk (for
  example if the block has been pruned). Previously returned RPC_INTERNAL_ERROR.
  - `pruneblockchain` now returns RPC_MISC_ERROR if the blocks cannot be pruned
  because the node is not in pruned mode. Previously returned RPC_METHOD_NOT_FOUND.
  - `pruneblockchain` now returns RPC_INVALID_PARAMETER if the blocks cannot be pruned
  because the supplied timestamp is too late. Previously returned RPC_INTERNAL_ERROR.
  - `pruneblockchain` now returns RPC_MISC_ERROR if the blocks cannot be pruned
  because the blockchain is too short. Previously returned RPC_INTERNAL_ERROR.
  - `setban` now returns RPC_CLIENT_INVALID_IP_OR_SUBNET if the supplied IP address
  or subnet is invalid. Previously returned RPC_CLIENT_NODE_ALREADY_ADDED.
  - `setban` now returns RPC_CLIENT_INVALID_IP_OR_SUBNET if the user tries to unban
  a node that has not previously been banned. Previously returned RPC_MISC_ERROR.
  - `removeprunedfunds` now returns RPC_WALLET_ERROR if `verged` is unable to remove
  the transaction. Previously returned RPC_INTERNAL_ERROR.
  - `removeprunedfunds` now returns RPC_INVALID_PARAMETER if the transaction does not
  exist in the wallet. Previously returned RPC_INTERNAL_ERROR.
  - `fundrawtransaction` now returns RPC_INVALID_ADDRESS_OR_KEY if an invalid change
  address is provided. Previously returned RPC_INVALID_PARAMETER.
  - `fundrawtransaction` now returns RPC_WALLET_ERROR if `verged` is unable to create
  the transaction. The error message provides further details. Previously returned
  RPC_INTERNAL_ERROR.
  - `bumpfee` now returns RPC_INVALID_PARAMETER if the provided transaction has
  descendants in the wallet. Previously returned RPC_MISC_ERROR.
  - `bumpfee` now returns RPC_INVALID_PARAMETER if the provided transaction has
  descendants in the mempool. Previously returned RPC_MISC_ERROR.
  - `bumpfee` now returns RPC_WALLET_ERROR if the provided transaction has
  has been mined or conflicts with a mined transaction. Previously returned
  RPC_INVALID_ADDRESS_OR_KEY.
  - `bumpfee` now returns RPC_WALLET_ERROR if the provided transaction is not
  BIP 125 replaceable. Previously returned RPC_INVALID_ADDRESS_OR_KEY.
  - `bumpfee` now returns RPC_WALLET_ERROR if the provided transaction has already
  been bumped by a different transaction. Previously returned RPC_INVALID_REQUEST.
  - `bumpfee` now returns RPC_WALLET_ERROR if the provided transaction contains
  inputs which don't belong to this wallet. Previously returned RPC_INVALID_ADDRESS_OR_KEY.
  - `bumpfee` now returns RPC_WALLET_ERROR if the provided transaction has multiple change
  outputs. Previously returned RPC_MISC_ERROR.
  - `bumpfee` now returns RPC_WALLET_ERROR if the provided transaction has no change
  output. Previously returned RPC_MISC_ERROR.
  - `bumpfee` now returns RPC_WALLET_ERROR if the fee is too high. Previously returned
  RPC_MISC_ERROR.
  - `bumpfee` now returns RPC_WALLET_ERROR if the fee is too low. Previously returned
  RPC_MISC_ERROR.
  - `bumpfee` now returns RPC_WALLET_ERROR if the change output is too small to bump the
  fee. Previously returned RPC_MISC_ERROR.

  ### BIP173 (Bech32) Address support ("bc1..." addresses)

Full support for native segwit addresses (BIP173 / Bech32) has now been added.
This includes the ability to send to BIP173 addresses (including non-v0 ones), and generating these
addresses (including as default new addresses, see above).

A checkbox has been added to the GUI to select whether a Bech32 address or P2SH-wrapped address should be generated when using segwit addresses. When launched with `-addresstype=bech32` it is checked by default. When launched with `-addresstype=legacy` it is unchecked and disabled.

### HD-wallets by default

Due to a backward-incompatible change in the wallet database, wallets created
with version 0.16.0 will be rejected by previous versions. Also, version 0.16.0
will only create hierarchical deterministic (HD) wallets. Note that this only applies
to new wallets; wallets made with previous versions will not be upgraded to be HD.

### Replace-By-Fee by default in GUI

The send screen now uses BIP125 RBF by default, regardless of `-walletrbf`.
There is a checkbox to mark the transaction as final.

The RPC default remains unchanged: to use RBF, launch with `-walletrbf=1` or
use the `replaceable` argument for individual transactions.

### Wallets directory configuration (`-walletdir`)

Verge now has more flexibility in where the wallets directory can be
located. Previously wallet database files were stored at the top level of the
verge data directory. The behavior is now:

- For new installations (where the data directory doesn't already exist),
  wallets will now be stored in a new `wallets/` subdirectory inside the data
  directory by default.
- For existing nodes (where the data directory already exists), wallets will be
  stored in the data directory root by default. If a `wallets/` subdirectory
  already exists in the data directory root, then wallets will be stored in the
  `wallets/` subdirectory by default.
- The location of the wallets directory can be overridden by specifying a
  `-walletdir=<path>` option where `<path>` can be an absolute path to a
  directory or directory symlink.

Care should be taken when choosing the wallets directory location, as if it
becomes unavailable during operation, funds may be lost.

## Build: Minimum GCC bumped to 4.8.x

The minimum version of the GCC compiler required to compile VERGE is now 4.8. No effort will be
made to support older versions of GCC. See discussion in issue #11732 for more information.
The minimum version for the Clang compiler is still 3.3. Other minimum dependency versions can be found in `doc/dependencies.md` in the repository.

## Support for signalling pruned nodes (BIP159)

Pruned nodes can now signal BIP159's NODE_NETWORK_LIMITED using service bits, in preparation for
full BIP159 support in later versions. This would allow pruned nodes to serve the most recent blocks. However, the current change does not yet include support for connecting to these pruned peers.


## GUI changes

- The option to reuse a previous address has now been removed. This was justified by the need to "resend" an invoice, but now that we have the request history, that need should be gone.
- Support for searching by TXID has been added, rather than just address and label.
- A "Use available balance" option has been added to the send coins dialog, to add the remaining available wallet balance to a transaction output.
- A toggle for unblinding the password fields on the password dialog has been added.

## RPC changes


### New `rescanblockchain` RPC

A new RPC `rescanblockchain` has been added to manually invoke a blockchain rescan.
The RPC supports start and end-height arguments for the rescan, and can be used in a
multiwallet environment to rescan the blockchain at runtime.

### New `savemempool` RPC
A new `savemempool` RPC has been added which allows the current mempool to be saved to
disk at any time to avoid it being lost due to crashes / power loss.

### Safe mode disabled by default

Safe mode is now disabled by default and must be manually enabled (with `-disablesafemode=0`) if you wish to use it. Safe mode is a feature that disables a subset of RPC calls - mostly related to the wallet and sending - automatically in case certain problem conditions with the network are detected. However, developers have come to regard these checks as not reliable enough to act on automatically. Even with safe mode disabled, they will still cause warnings in the `warnings` field of the `getneworkinfo` RPC and launch the `-alertnotify` command.

### Renamed script for creating JSON-RPC credentials

The `share/rpcuser/rpcuser.py` script was renamed to `share/rpcauth/rpcauth.py`. This script can be
used to create `rpcauth` credentials for a JSON-RPC user.

### Validateaddress improvements

The `validateaddress` RPC output has been extended with a few new fields, and support for segwit addresses (both P2SH and Bech32). Specifically:
* A new field `iswitness` is True for P2WPKH and P2WSH addresses ("bc1..." addresses), but not for P2SH-wrapped segwit addresses (see below).
* The existing field `isscript` will now also report True for P2WSH addresses.
* A new field `embedded` is present for all script addresses where the script is known and matches something that can be interpreted as a known address. This is particularly true for P2SH-P2WPKH and P2SH-P2WSH addresses. The value for `embedded` includes much of the information `validateaddress` would report if invoked directly on the embedded address.
* For multisig scripts a new `pubkeys` field was added that reports the full public keys involved in the script (if known). This is a replacement for the existing `addresses` field (which reports the same information but encoded as P2PKH addresses), represented in a more useful and less confusing way. The `addresses` field remains present for non-segwit addresses for backward compatibility.
* For all single-key addresses with known key (even when wrapped in P2SH or P2WSH), the `pubkey` field will be present. In particular, this means that invoking `validateaddress` on the output of `getnewaddress` will always report the `pubkey`, even when the address type is P2SH-P2WPKH.

### Low-level changes

- The deprecated RPC `getinfo` was removed. It is recommended that the more specific RPCs are used:
  * `getblockchaininfo`
  * `getnetworkinfo`
  * `getwalletinfo`
  * `getmininginfo`
- The wallet RPC `getreceivedbyaddress` will return an error if called with an address not in the wallet.
- The wallet RPC `addwitnessaddress` was deprecated and will be removed in version 0.17,
  set the `address_type` argument of `getnewaddress`, or option `-addresstype=[bech32|p2sh-segwit]` instead.
- `dumpwallet` now includes hex-encoded scripts from the wallet in the dumpfile, and
  `importwallet` now imports these scripts, but corresponding addresses may not be added
  correctly or a manual rescan may be required to find relevant transactions.
- The RPC `getblockchaininfo` now includes an `errors` field.
- A new `blockhash` parameter has been added to the `getrawtransaction` RPC which allows for a raw transaction to be fetched from a specific block if known, even without `-txindex` enabled.
- The `decoderawtransaction` and `fundrawtransaction` RPCs now have optional `iswitness` parameters to override the
  heuristic witness checks if necessary.
- The `walletpassphrase` timeout is now clamped to 2^30 seconds.
- Using addresses with the `createmultisig` RPC is now deprecated, and will be removed in a later version. Public keys should be used instead.
- Blockchain rescans now no longer lock the wallet for the entire rescan process, so other RPCs can now be used at the same time (although results of balances / transactions may be incorrect or incomplete until the rescan is complete).
- The `logging` RPC has now been made public rather than hidden.
- An `initialblockdownload` boolean has been added to the `getblockchaininfo` RPC to indicate whether the node is currently in IBD or not.
- `minrelaytxfee` is now included in the output of `getmempoolinfo`

## Other changed command-line options

- `-debuglogfile=<file>` can be used to specify an alternative debug logging file.
- verge-cli now has an `-stdinrpcpass` option to allow the RPC password to be read from standard input.
- The `-usehd` option has been removed.
- verge-cli now supports a new `-getinfo` flag which returns an output like that of the now-removed `getinfo` RPC.