#codebase Change Log


#####Version .11.1


##Fix buffer overflow in bundled upnp

Bundled miniupnpc was updated to 1.9.20151008. This fixes a buffer overflow in the XML parser during initial network discovery.

Details can be found here: http://talosintel.com/reports/TALOS-2015-0035/

This applies to the distributed executables only, not when building from source or using distribution provided packages.

Additionally, upnp has been disabled by default. This may result in a lower number of reachable nodes on IPv4, however this prevents future libupnpc vulnerabilities from being a structural risk to the network.



##Test for LowS signatures before relaying

Make the node require the canonical ‘low-s’ encoding for ECDSA signatures when relaying or mining. This removes a nuisance malleability vector.

Consensus behavior is unchanged.

If widely deployed this change would eliminate the last remaining known vector for nuisance malleability on SIGHASH_ALL P2PKH transactions. On the down-side it will block most transactions made by sufficiently out of date software.

Unlike the other avenues to change txids on transactions this one was randomly violated by all deployed bitcoin software prior to its discovery. So, while other malleability vectors where made non-standard as soon as they were discovered, this one has remained permitted. Even BIP62 did not propose applying this rule to old version transactions, but conforming implementations have become much more common since BIP62 was initially written.

Bitcoin Core has produced compatible signatures since a28fb70e in September 2013, but this didn’t make it into a release until 0.9 in March 2014; Bitcoinj has done so for a similar span of time. Bitcoinjs and electrum have been more recently updated.

This does not replace the need for BIP62 or similar, as miners can still cooperate to break transactions. Nor does it replace the need for wallet software to handle malleability sanely[1]. This only eliminates the cheap and irritating DOS attack.

[1] On the Malleability of Bitcoin Transactions Marcin Andrychowicz, Stefan Dziembowski, Daniel Malinowski, Łukasz Mazurek http://fc15.ifca.ai/preproceedings/bitcoin/paper_9.pdf

##Minimum relay fee default increase

The default for the ```-minrelaytxfee``` setting has been increased from ```0.00001``` to ```0.00005```.

This is necessitated by the current transaction flooding, causing outrageous memory usage on nodes due to the mempool ballooning. This is a temporary measure, bridging the time until a dynamic method for determining this fee is merged (which will be in 0.12).



######Version .11.2


##BIP113 mempool-only locktime enforcement using GetMedianTimePast()

Bitcoin transactions currently may specify a locktime indicating when they may be added to a valid block. Current consensus rules require that blocks have a block header time greater than the locktime specified in any transaction in that block.

Miners get to choose what time they use for their header time, with the consensus rule being that no node will accept a block whose time is more than two hours in the future. This creates a incentive for miners to set their header times to future values in order to include locktimed transactions which weren’t supposed to be included for up to two more hours.

The consensus rules also specify that valid blocks may have a header time greater than that of the median of the 11 previous blocks. This GetMedianTimePast() time has a key feature we generally associate with time: it can’t go backwards.

BIP113 specifies a soft fork (not enforced in this release) that weakens this perverse incentive for individual miners to use a future time by requiring that valid blocks have a computed GetMedianTimePast() greater than the locktime specified in any transaction in that block.

Mempool inclusion rules currently require transactions to be valid for immediate inclusion in a block in order to be accepted into the mempool. This release begins applying the BIP113 rule to received transactions, so transaction whose time is greater than the GetMedianTimePast() will no longer be accepted into the mempool.

Implication for miners: you will begin rejecting transactions that would not be valid under BIP113, which will prevent you from producing invalid blocks if/when BIP113 is enforced on the network. Any transactions which are valid under the current rules but not yet valid under the BIP113 rules will either be mined by other miners or delayed until they are valid under BIP113. Note, however, that time-based locktime transactions are more or less unseen on the network currently.

Implication for users: GetMedianTimePast() always trails behind the current time, so a transaction locktime set to the present time will be rejected by nodes running this release until the median time moves forward. To compensate, subtract one hour (3,600 seconds) from your locktimes to allow those transactions to be included in mempools at approximately the expected time.


##Windows bug fix for corrupted UTXO database on unclean shutdowns

Several Windows users reported that they often need to reindex the entire blockchain after an unclean shutdown of Bitcoin Core on Windows (or an unclean shutdown of Windows itself). Although unclean shutdowns remain unsafe, this release no longer relies on memory-mapped files for the UTXO database, which significantly reduced the frequency of unclean shutdowns leading to required reindexes during testing.

Other fixes for database corruption on Windows are expected in the next major release.


######Version 0.12.0


##Signature validation using libsecp256k1

ECDSA signatures inside Bitcoin transactions now use validation using libsecp256k1 instead of OpenSSL.

Depending on the platform, this means a significant speedup for raw signature validation speed. The advantage is largest on x86_64, where validation is over five times faster. In practice, this translates to a raw reindexing and new block validation times that are less than half of what it was before.

Libsecp256k1 has undergone very extensive testing and validation.

A side effect of this change is that libconsensus no longer depends on OpenSSL.


##Reduce upload traffic

A major part of the outbound traffic is caused by serving historic blocks to other nodes in initial block download state.

It is now possible to reduce the total upload traffic via the ```-maxuploadtarget``` parameter. This is not a hard limit but a threshold to minimize the outbound traffic. When the limit is about to be reached, the uploaded data is cut by not serving historic blocks (blocks older than one week). Moreover, any SPV peer is disconnected when they request a filtered block.

This option can be specified in MiB per day and is turned off by default (```-maxuploadtarget=0```). The recommended minimum is 144 * MAX_BLOCK_SIZE (currently 144MB) per day.

Whitelisted peers will never be disconnected, although their traffic counts for calculating the target.

A more detailed documentation about keeping traffic low can be found in [reduce-traffic](https://github.com/bitcoin/bitcoin/blob/v0.12.0/doc/reduce-traffic.md)


##Direct headers announcement (BIP 130)

Between compatible peers, [BIP 130](https://github.com/bitcoin/bips/blob/master/bip-0130.mediawiki) direct headers announcement is used. This means that blocks are advertised by announcing their headers directly, instead of just announcing the hash. In a reorganization, all new headers are sent, instead of just the new tip. This can often prevent an extra roundtrip before the actual block is downloaded.

With this change, pruning nodes are now able to relay new blocks to compatible peers.


##Memory pool limiting

Previous versions of Bitcoin Core had their mempool limited by checking a transaction’s fees against the node’s minimum relay fee. There was no upper bound on the size of the mempool and attackers could send a large number of transactions paying just slighly more than the default minimum relay fee to crash nodes with relatively low RAM. A temporary workaround for previous versions of Bitcoin Core was to raise the default minimum relay fee.

Bitcoin Core 0.12 will have a strict maximum size on the mempool. The default value is 300 MB and can be configured with the ```-maxmempool``` parameter. Whenever a transaction would cause the mempool to exceed its maximum size, the transaction that (along with in-mempool descendants) has the lowest total feerate (as a package) will be evicted and the node’s effective minimum relay feerate will be increased to match this feerate plus the initial minimum relay feerate. The initial minimum relay feerate is set to 1000 satoshis per kB.

Bitcoin Core 0.12 also introduces new default policy limits on the length and size of unconfirmed transaction chains that are allowed in the mempool (generally limiting the length of unconfirmed chains to 25 transactions, with a total size of 101 KB). These limits can be overriden using command line arguments; see the extended help (```--help -help-debug```) for more information.


##Opt-in Replace-by-fee transactions

It is now possible to replace transactions in the transaction memory pool of Bitcoin Core 0.12 nodes. Bitcoin Core will only allow replacement of transactions which have any of their inputs’ ```nSequence``` number set to less than ```0xffffffff - 1```. Moreover, a replacement transaction may only be accepted when it pays sufficient fee, as described in [BIP 125](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki).

Transaction replacement can be disabled with a new command line option, ```-mempoolreplacement=0```. Transactions signaling replacement under BIP125 will still be allowed into the mempool in this configuration, but replacements will be rejected. This option is intended for miners who want to continue the transaction selection behavior of previous releases.

The ```-mempoolreplacement``` option is not recommended for wallet users seeking to avoid receipt of unconfirmed opt-in transactions, because this option does not prevent transactions which are replaceable under BIP 125 from being accepted (only subsequent replacements, which other nodes on the network that implement BIP 125 are likely to relay and mine). Wallet users wishing to detect whether a transaction is subject to replacement under BIP 125 should instead use the updated RPC calls ```gettransaction``` and ```listtransactions```, which now have an additional field in the output indicating if a transaction is replaceable under BIP125 (“bip125-replaceable”).

Note that the wallet in Bitcoin Core 0.12 does not yet have support for creating transactions that would be replaceable under BIP 125.


##RPC: Random-cookie RPC authentication

When no ```-rpcpassword``` is specified, the daemon now uses a special ‘cookie’ file for authentication. This file is generated with random content when the daemon starts, and deleted when it exits. Its contents are used as authentication token. Read access to this file controls who can access through RPC. By default it is stored in the data directory but its location can be overridden with the option ```-rpccookiefile```.

This is similar to Tor’s CookieAuthentication: see https://www.torproject.org/docs/tor-manual.html.en

This allows running bitcoind without having to do any manual configuration.


##Relay: Any sequence of pushdatas in OP_RETURN outputs now allowed

Previously OP_RETURN outputs with a payload were only relayed and mined if they had a single pushdata. This restriction has been lifted to allow any combination of data pushes and numeric constant opcodes (OP_1 to OP_16) after the OP_RETURN. The limit on OP_RETURN output size is now applied to the entire serialized scriptPubKey, 83 bytes by default. (the previous 80 byte default plus three bytes overhead)


##Relay and Mining: Priority transactions

Bitcoin Core has a heuristic ‘priority’ based on coin value and age. This calculation is used for relaying of transactions which do not pay the minimum relay fee, and can be used as an alternative way of sorting transactions for mined blocks. Bitcoin Core will relay transactions with insufficient fees depending on the setting of ```-limitfreerelay=<r>``` (default: ```r=15``` kB per minute) and ```-blockprioritysize=<s>```.

In Bitcoin Core 0.12, when mempool limit has been reached a higher minimum relay fee takes effect to limit memory usage. Transactions which do not meet this higher effective minimum relay fee will not be relayed or mined even if they rank highly according to the priority heuristic.

The mining of transactions based on their priority is also now disabled by default. To re-enable it, simply set ```-blockprioritysize=<n>``` where is the size in bytes of your blocks to reserve for these transactions. The old default was 50k, so to retain approximately the same policy, you would set ```-blockprioritysize=50000```.

Additionally, as a result of computational simplifications, the priority value used for transactions received with unconfirmed inputs is lower than in prior versions due to avoiding recomputing the amounts as input transactions confirm.

External miner policy set via the ```prioritisetransaction``` RPC to rank transactions already in the mempool continues to work as it has previously. Note, however, that if mining priority transactions is left disabled, the priority delta will be ignored and only the fee metric will be effective.

This internal automatic prioritization handling is being considered for removal entirely in Bitcoin Core 0.13, and it is at this time undecided whether the more accurate priority calculation for chained unconfirmed transactions will be restored. Community direction on this topic is particularly requested to help set project priorities.


##Automatically use Tor hidden services

Starting with Tor version 0.2.7.1 it is possible, through Tor’s control socket API, to create and destroy ‘ephemeral’ hidden services programmatically. Bitcoin Core has been updated to make use of this.

This means that if Tor is running (and proper authorization is available), Bitcoin Core automatically creates a hidden service to listen on, without manual configuration. Bitcoin Core will also use Tor automatically to connect to other .onion nodes if the control socket can be successfully opened. This will positively affect the number of available .onion nodes and their usage.

This new feature is enabled by default if Bitcoin Core is listening, and a connection to Tor can be made. It can be configured with the ```-listenonion```, ```-torcontrol``` and ```-torpassword``` settings. To show verbose debugging information, pass ```-debug=tor```.


##Notifications through ZMQ

Bitcoind can now (optionally) asynchronously notify clients through a ZMQ-based PUB socket of the arrival of new transactions and blocks. This feature requires installation of the ZMQ C API library 4.x and configuring its use through the command line or configuration file. Please see [ZeroMQ](https://github.com/bitcoin/bitcoin/blob/v0.12.0/doc/zmq.md) for details of operation.

##Wallet: Transaction fees

Various improvements have been made to how the wallet calculates transaction fees.

Users can decide to pay a predefined fee rate by setting ```-paytxfee=<n>``` (or ```settxfee <n>``` rpc during runtime). A value of ```n=0``` signals Bitcoin Core to use floating fees. By default, Bitcoin Core will use floating fees.

Based on past transaction data, floating fees approximate the fees required to get into the mth block from now. This is configurable with ```-txconfirmtarget=<m>``` (default: 2).

Sometimes, it is not possible to give good estimates, or an estimate at all. Therefore, a fallback value can be set with ```-fallbackfee=<f>``` (default: ```0.0002``` BTC/kB).

At all times, Bitcoin Core will cap fees at ```-maxtxfee=<x>``` (default: 0.10) BTC. Furthermore, Bitcoin Core will never create transactions paying less than the current minimum relay fee. Finally, a user can set the minimum fee rate for all transactions with ```-mintxfee=<i>```, which defaults to 1000 satoshis per kB.

##Wallet: Negative confirmations and conflict detection

The wallet will now report a negative number for confirmations that indicates how deep in the block chain the conflict is found. For example, if a transaction A has 5 confirmations and spends the same input as a wallet transaction B, B will be reported as having -5 confirmations. If another wallet transaction C spends an output from B, it will also be reported as having -5 confirmations. To detect conflicts with historical transactions in the chain a one-time ```-rescan``` may be needed.

Unlike earlier versions, unconfirmed but non-conflicting transactions will never get a negative confirmation count. They are not treated as spendable unless they’re coming from ourself (change) and accepted into our local mempool, however. The new “trusted” field in the ```listtransactions``` RPC output indicates whether outputs of an unconfirmed transaction are considered spendable.

##Wallet: Merkle branches removed

Previously, every wallet transaction stored a Merkle branch to prove its presence in blocks. This wasn’t being used for more than an expensive sanity check. Since 0.12, these are no longer stored. When loading a 0.12 wallet into an older version, it will automatically rescan to avoid failed checks.

##Wallet: Pruning

With 0.12 it is possible to use wallet functionality in pruned mode. This can reduce the disk usage from currently around 60 GB to around 2 GB.

However, rescans as well as the RPCs ```importwallet```, ```importaddress```, ```importprivkey``` are disabled.

To enable block pruning set ```prune=<N>``` on the command line or in ```bitcoin.conf```, where N is the number of MiB to allot for raw block & undo data.

A value of 0 disables pruning. The minimal value above 0 is 550. Your wallet is as secure with high values as it is with low ones. Higher values merely ensure that your node will not shut down upon blockchain reorganizations of more than 2 days - which are unlikely to happen in practice. In future releases, a higher value may also help the network as a whole: stored blocks could be served to other nodes.

For further information about pruning, you may also consult the release notes of [v0.11.0](https://github.com/bitcoin/bitcoin/blob/v0.11.0/doc/release-notes.md#block-file-pruning).

##```NODE_BLOOM``` service bit

Support for the ```NODE_BLOOM``` service bit, as described in [BIP 111](https://github.com/bitcoin/bips/blob/master/bip-0111.mediawiki), has been added to the P2P protocol code.

BIP 111 defines a service bit to allow peers to advertise that they support bloom filters (such as used by SPV clients) explicitly. It also bumps the protocol version to allow peers to identify old nodes which allow bloom filtering of the connection despite lacking the new service bit.

In this version, it is only enforced for peers that send protocol versions ```>=70011```. For the next major version it is planned that this restriction will be removed. It is recommended to update SPV clients to check for the ```NODE_BLOOM``` service bit for nodes that report versions newer than 70011.

##Option parsing behavior

Command line options are now parsed strictly in the order in which they are specified. It used to be the case that ```-X -noX``` ends up, unintuitively, with X set, as ```-X``` had precedence over ```-noX```. This is no longer the case. Like for other software, the last specified value for an option will hold.

##RPC: Low-level API changes

* Monetary amounts can be provided as strings. This means that for example the argument to sendtoaddress can be “0.0001” instead of 0.0001. This can be an advantage if a JSON library insists on using a lossy floating point type for numbers, which would be dangerous for monetary amounts.

* The ```asm``` property of each scriptSig now contains the decoded signature hash type for each signature that provides a valid defined hash type.

* OP_NOP2 has been renamed to OP_CHECKLOCKTIMEVERIFY by BIP 65

The following items contain assembly representations of scriptSig signatures and are affected by this change:

* RPC ```getrawtransaction```
* RPC ```decoderawtransaction```
* RPC ```decodescript```
* REST ```/rest/tx/``` (JSON format)
* REST ```/rest/block/``` (JSON format when including extended tx details)
* ```bitcoin-tx -json```

For example, the scriptSig.asm property of a transaction input that previously showed an assembly representation of:
```
304502207fa7a6d1e0ee81132a269ad84e68d695483745cde8b541e3bf630749894e342a022100c1f7ab20e13e22fb95281a870f3dcf38d782e53023ee313d741ad0cfbc0c509001 400000 OP_NOP2
```
now shows as:
```
304502207fa7a6d1e0ee81132a269ad84e68d695483745cde8b541e3bf630749894e342a022100c1f7ab20e13e22fb95281a870f3dcf38d782e53023ee313d741ad0cfbc0c5090[ALL] 400000 OP_CHECKLOCKTIMEVERIFY
```
Note that the output of the RPC ```decodescript``` did not change because it is configured specifically to process scriptPubKey and not scriptSig scripts.


##RPC: SSL support dropped

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

<Location /bitcoinrpc>
    ProxyPass http://127.0.0.1:8332/
    ProxyPassReverse http://127.0.0.1:8332/
    # optional enable digest auth
    # AuthType Digest
    # ...

    # optional bypass bitcoind rpc basic auth
    # RequestHeader set Authorization "Basic <hash>"
    # get the <hash> from the shell with: base64 <<< bitcoinrpc:<password>
</Location>

# Or, balance the load:
# ProxyPass / balancer://balancer_cluster_name

</VirtualHost>
```


##Mining Code Changes

The mining code in 0.12 has been optimized to be significantly faster and use less memory. As part of these changes, consensus critical calculations are cached on a transaction’s acceptance into the mempool and the mining code now relies on the consistency of the mempool to assemble blocks. However all blocks are still tested for validity after assembly.


##Other P2P Changes

The list of banned peers is now stored on disk rather than in memory. Restarting bitcoind will no longer clear out the list of banned peers; instead a new RPC call (```clearbanned```) can be used to manually clear the list. The new ```setban``` RPC call can also be used to manually ban or unban a peer.


######Version 0.12.1


##First version bits BIP9 softfork deployment

This release includes a soft fork deployment to enforce BIP68, BIP112 and BIP113 using the BIP9 deployment mechanism.

The deployment sets the block version number to 0x20000001 between midnight 1st May 2016 and midnight 1st May 2017 to signal readiness for deployment. The version number consists of 0x20000000 to indicate version bits together with setting bit 0 to indicate support for this combined deployment, shown as “csv” in the ```getblockchaininfo``` RPC call.

For more information about the soft forking change, please see https://github.com/bitcoin/bitcoin/pull/7648

This specific backport pull-request can be viewed at https://github.com/bitcoin/bitcoin/pull/7543


##BIP68 soft fork to enforce sequence locks for relative locktime

BIP68 introduces relative lock-time consensus-enforced semantics of the sequence number field to enable a signed transaction input to remain invalid for a defined period of time after confirmation of its corresponding outpoint.

For more information about the implementation, see https://github.com/bitcoin/bitcoin/pull/7184


##BIP112 soft fork to enforce OP_CHECKSEQUENCEVERIFY

BIP112 redefines the existing OP_NOP3 as OP_CHECKSEQUENCEVERIFY (CSV) for a new opcode in the Bitcoin scripting system that in combination with BIP68 allows execution pathways of a script to be restricted based on the age of the output being spent.

For more information about the implementation, see https://github.com/bitcoin/bitcoin/pull/7524


##BIP113 locktime enforcement soft fork

Bitcoin Core 0.11.2 previously introduced mempool-only locktime enforcement using GetMedianTimePast(). This release seeks to consensus enforce the rule.

Bitcoin transactions currently may specify a locktime indicating when they may be added to a valid block. Current consensus rules require that blocks have a block header time greater than the locktime specified in any transaction in that block.

Miners get to choose what time they use for their header time, with the consensus rule being that no node will accept a block whose time is more than two hours in the future. This creates a incentive for miners to set their header times to future values in order to include locktimed transactions which weren’t supposed to be included for up to two more hours.

The consensus rules also specify that valid blocks may have a header time greater than that of the median of the 11 previous blocks. This GetMedianTimePast() time has a key feature we generally associate with time: it can’t go backwards.

BIP113 specifies a soft fork enforced in this release that weakens this perverse incentive for individual miners to use a future time by requiring that valid blocks have a computed GetMedianTimePast() greater than the locktime specified in any transaction in that block.

Mempool inclusion rules currently require transactions to be valid for immediate inclusion in a block in order to be accepted into the mempool. This release begins applying the BIP113 rule to received transactions, so transaction whose time is greater than the GetMedianTimePast() will no longer be accepted into the mempool.

####Implication for miners: 
you will begin rejecting transactions that would not be valid under BIP113, which will prevent you from producing invalid blocks when BIP113 is enforced on the network. Any transactions which are valid under the current rules but not yet valid under the BIP113 rules will either be mined by other miners or delayed until they are valid under BIP113. Note, however, that time-based locktime transactions are more or less unseen on the network currently.

####Implication for users: 
GetMedianTimePast() always trails behind the current time, so a transaction locktime set to the present time will be rejected by nodes running this release until the median time moves forward. To compensate, subtract one hour (3,600 seconds) from your locktimes to allow those transactions to be included in mempools at approximately the expected time.

For more information about the implementation, see https://github.com/bitcoin/bitcoin/pull/6566


##Miscellaneous

The p2p alert system is off by default. To turn on, use ```-alert``` with startup configuration.

#####Version 0.13.0


##Database cache memory increased

As a result of growth of the UTXO set, performance with the prior default database cache of 100 MiB has suffered. For this reason the default was changed to 300 MiB in this release.

For nodes on low-memory systems, the database cache can be changed back to 100 MiB (or to another value) by either:

* Adding ```dbcache=100``` in bitcoin.conf

* Changing it in the GUI under ```Options → Size of database cache```

######Note that the database cache setting has the most performance impact during initial sync of a node, and when catching up after downtime.


##bitcoin-cli: arguments privacy

The RPC command line client gained a new argument, ```-stdin``` to read extra arguments from standard input, one per line until EOF/Ctrl-D. For example:
```
$ src/bitcoin-cli -stdin walletpassphrase
mysecretcode
120
..... press Ctrl-D here to end input
$
```

It is recommended to use this for sensitive information such as wallet passphrases, as command-line arguments can usually be read from the process table by any user on the system.


##C++11 and Python 3

Various code modernizations have been done. The Bitcoin Core code base has started using C++11. This means that a C++11-capable compiler is now needed for building. Effectively this means GCC 4.7 or higher, or Clang 3.3 or higher.

When cross-compiling for a target that doesn’t have C++11 libraries, configure with ```./configure --enable-glibc-back-compat ... LDFLAGS=-static-libstdc++.```

For running the functional tests in qa/rpc-tests, Python3.4 or higher is now required.
Linux ARM builds

Due to popular request, Linux ARM builds have been added to the uploaded executables.

The following extra files can be found in the download directory or torrent:

* ```bitcoin-${VERSION}-arm-linux-gnueabihf.tar.gz```: Linux binaries for the most common 32-bit ARM architecture.


* ```bitcoin-${VERSION}-aarch64-linux-gnu.tar.gz```: Linux binaries for the most common 64-bit ARM architecture.


ARM builds are still experimental. If you have problems on a certain device or Linux distribution combination please report them on the bug tracker, it may be possible to resolve them.

Note that Android is not considered ARM Linux in this context. The executables are not expected to work out of the box on Android.


##Compact Block support (BIP 152)

Support for block relay using the Compact Blocks protocol has been implemented in PR 8068.

The primary goal is reducing the bandwidth spikes at relay time, though in many cases it also reduces propagation delay. It is automatically enabled between compatible peers. BIP 152

As a side-effect, ordinary non-mining nodes will download and upload blocks faster if those blocks were produced by miners using similar transaction filtering policies. This means that a miner who produces a block with many transactions discouraged by your node will be relayed slower than one with only transactions already in your memory pool. The overall effect of such relay differences on the network may result in blocks which include widely- discouraged transactions losing a stale block race, and therefore miners may wish to configure their node to take common relay policies into consideration.


##Hierarchical Deterministic Key Generation

Newly created wallets will use hierarchical deterministic key generation according to BIP32 (keypath m/0’/0’/k’). Existing wallets will still use traditional key generation.

Backups of HD wallets, regardless of when they have been created, can therefore be used to re-generate all possible private keys, even the ones which haven’t already been generated during the time of the backup. Attention: Encrypting the wallet will create a new seed which requires a new backup!

Wallet dumps (created using the dumpwallet RPC) will contain the deterministic seed. This is expected to allow future versions to import the seed and all associated funds, but this is not yet implemented.

HD key generation for new wallets can be disabled by -usehd=0. Keep in mind that this flag only has affect on newly created wallets. You can’t disable HD key generation once you have created a HD wallet.

There is no distinction between internal (change) and external keys.

HD wallets are incompatible with older versions of Bitcoin Core.


##Segregated Witness

The code preparations for Segregated Witness (“segwit”), as described in BIP 141, BIP 143, BIP 144, and BIP 145 are finished and included in this release. However, BIP 141 does not yet specify activation parameters on mainnet, and so this release does not support segwit use on mainnet. Testnet use is supported, and after BIP 141 is updated with proposed parameters, a future release of Bitcoin Core is expected that implements those parameters for mainnet.

Furthermore, because segwit activation is not yet specified for mainnet, version 0.13.0 will behave similarly as other pre-segwit releases even after a future activation of BIP 141 on the network. Upgrading from 0.13.0 will be required in order to utilize segwit-related features on mainnet (such as signal BIP 141 activation, mine segwit blocks, fully validate segwit blocks, relay segwit blocks to other segwit nodes, and use segwit transactions in the wallet, etc).


##Mining transaction selection (“Child Pays For Parent”)

The mining transaction selection algorithm has been replaced with an algorithm that selects transactions based on their feerate inclusive of unconfirmed ancestor transactions. This means that a low-fee transaction can become more likely to be selected if a high-fee transaction that spends its outputs is relayed.

With this change, the ```-blockminsize``` command line option has been removed.

The command line option ```-blockmaxsize``` remains an option to specify the maximum number of serialized bytes in a generated block. In addition, the new command line option ```-blockmaxweight``` has been added, which specifies the maximum “block weight” of a generated block, as defined by [BIP 141 (Segregated Witness)](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki).

In preparation for Segregated Witness, the mining algorithm has been modified to optimize transaction selection for a given block weight, rather than a given number of serialized bytes in a block. In this release, transaction selection is unaffected by this distinction (as BIP 141 activation is not supported on mainnet in this release, see above), but in future releases and after BIP 141 activation, these calculations would be expected to differ.

For optimal runtime performance, miners using this release should specify ```-blockmaxweight``` on the command line, and not specify ```-blockmaxsize```. Additionally (or only) specifying ```-blockmaxsize```, or relying on default settings for both, may result in performance degradation, as the logic to support ```-blockmaxsize``` performs additional computation to ensure that constraint is met. (Note that for mainnet, in this release, the equivalent parameter for ```-blockmaxweight``` would be four times the desired ```-blockmaxsize```. See [BIP 141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) for additional details.)

In the future, the ```-blockmaxsize``` option may be removed, as block creation is no longer optimized for this metric. Feedback is requested on whether to deprecate or keep this command line option in future releases.

##Reindexing changes

In earlier versions, reindexing did validation while reading through the block files on disk. These two have now been split up, so that all blocks are known before validation starts. This was necessary to make certain optimizations that are available during normal synchronizations also available during reindexing.

The two phases are distinct in the Bitcoin-Qt GUI. During the first one, “Reindexing blocks on disk” is shown. During the second (slower) one, “Processing blocks on disk” is shown.

It is possible to only redo validation now, without rebuilding the block index, using the command line option ```-reindex-chainstate``` (in addition to ```-reindex```which does both). This new option is useful when the blocks on disk are assumed to be fine, but the chainstate is still corrupted. It is also useful for benchmarks.
Removal of internal miner

As CPU mining has been useless for a long time, the internal miner has been removed in this release, and replaced with a simpler implementation for the test framework.

The overall result of this is that ```setgenerate``` RPC call has been removed, as well as the ```-gen``` and ```-genproclimit``` command-line options.

For testing, the ```generate``` call can still be used to mine a block, and a new RPC call ```generatetoaddress``` has been added to mine to a specific address. This works with wallet disabled.


##New bytespersigop implementation

The former implementation of the bytespersigop filter accidentally broke bare multisig (which is meant to be controlled by the ```permitbaremultisig``` option), since the consensus protocol always counts these older transaction forms as 20 sigops for backwards compatibility. Simply fixing this bug by counting more accurately would have reintroduced a vulnerability. It has therefore been replaced with a new implementation that rather than filter such transactions, instead treats them (for fee purposes only) as if they were in fact the size of a transaction actually using all 20 sigops.

##Low-level P2P changes

* The optional new p2p message “feefilter” is implemented and the protocol version is bumped to 70013. Upon receiving a feefilter message from a peer, a node will not send invs for any transactions which do not meet the filter feerate. BIP 133

* The P2P alert system has been removed in PR #7692 and the alert P2P message is no longer supported.

* The transaction relay mechanism used to relay one quarter of all transactions instantly, while queueing up the rest and sending them out in batch. As this resulted in chains of dependent transactions being reordered, it systematically hurt transaction relay. The relay code was redesigned in PRs <a href=”https://github.com/bitcoin/bitcoin/pull/7840”>#7840</a> and #8082, and now always batches transactions announcements while also sorting them according to dependency order. This significantly reduces orphan transactions. To compensate for the removal of instant relay, the frequency of batch sending was doubled for outgoing peers.

* Since PR #7840 the BIP35 mempool command is also subject to batch processing. Also the mempool message is no longer handled for non-whitelisted peers when NODE_BLOOM is disabled through -peerbloomfilters=0.

* The maximum size of orphan transactions that are kept in memory until their ancestors arrive has been raised in PR #8179 from 5000 to 99999 bytes. They are now also removed from memory when they are included in a block, conflict with a block, and time out after 20 minutes.

* We respond at most once to a getaddr request during the lifetime of a connection since PR #7856.

* Connections to peers who have recently been the first one to give us a valid new block or transaction are protected from disconnections since PR #8084.


##Low-level RPC changes

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
        * ```bitcoin-tx -json```

* The sorting of the output of the ```getrawmempool``` output has changed.

* New RPC commands: ```generatetoaddress```, ```importprunedfunds```, ```removeprunedfunds```, ```signmessagewithprivkey```, ```getmempoolancestors```, ```getmempooldescendants```, ```getmempoolentry```, ```createwitnessaddress```, ```addwitnessaddress```.

* Removed RPC commands: ```setgenerate```, ```getgenerate```.

* New options were added to fundrawtransaction: ```includeWatching```, ```changeAddress```, ```changePosition``` and ```feeRate```.


##Low-level ZMQ changes

* Each ZMQ notification now contains an up-counting sequence number that allows listeners to detect lost notifications. The sequence number is always the last element in a multi-part ZMQ notification and therefore backward compatible. Each message type has its own counter. PR #7762.

