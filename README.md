## Requirements

* `lnd`
* `btcd`
* 5 different terminal windows open

## Setup

To run watchtowers at the time of writing, we'll need to checkout some as of yet unmerged code from [this pull request](https://github.com/lightningnetwork/lnd/pull/3133), and recompile LND. If you see the PR as merged, you can do the following on the `master` branch instead of the PR branch.

These commands assume you've already installed LND before. Make sure you've turned off any running nodes before doing this.

```bash
cd $GOPATH/src/github.com/lightningnetwork/lnd
git fetch origin pull/3133/head:wt-polish
git checkout wt-polish
make clean && make && make install
```

## Walthrough

### 1. Initial Setup

We need to setup our two nodes, `attacker` and `victim`. Config files have already been created for them, you just need to get them started.

```sh
# Window 1
cmd/start-btcd.sh

# Window 2
cmd/start-victim.sh

# Window 3
cmd/start-attacker.sh

# Window 4
cmd/lncli-victim.sh create # Enter 'password', 'password', 'n', and hit enter for the last one
cmd/lncli-attacker.sh create # Enter 'password', 'password', 'n', and hit enter for the last one
```

### 2. Configure the Watchtower

In order to configure our watchtower, we need to know our node's pubkey. Let's grab it from our victim:

```sh
# Window 4
cmd/lncli-victim.sh getinfo # Copy the "identity_pubkey" from this
```

Then open up `nodes/victim/lnd.conf`, and uncomment and add the victim pubkey to the indicated spot. It should look something like this:

```conf
[Wtclient]
wtclient.private-tower-uris=037ab06ef760b98c88bb28e4a4959e37ea370a71f2d7a74c7bd8274961b23ed3b5@localhost:30021
```

You'll then need to restart the node with a `CTRL+C` and running the following:

```sh
# Window 2
cmd/start-victim.sh

# Window 4
cmd/lncli-victim.sh unlock # Enter 'password' from the create command
```

You'll know it worked if it starts successfully and you see the following logs in window 4:

```log
[INF] WTWR: Starting watchtower server
[INF] WTWR: Watchtower server started successfully
...
[INF] WTCL: Acquired new session with id=<victim-pubkey>
```

### 3. Getting Funds

In order to try out an attack, we need to fund the attacker with some simnet bitcoin. Due to the way btcd works, we'll need to restart it to get some money going to the attacker. Run the following in order, but in their respective windows:

```sh
# Window 4
cmd/lncli-attacker.sh newaddress p2wkh # Copy the address

# Window 1, CTRL+C then run
cmd/start-btcd.sh --miningaddr=[address] # Use the address from window 4

# Window 4
cmd/btcctl.sh generate 500
```

You can confirm it worked by checking the attacker's node wallet balance:

```sh
# Window 4
cmd/lncli-attacker.sh walletbalance
```

### 4. Running the Attack

Now that everything's in place, we'll begin our attack. The first thing we need to do is connect the two nodes as peers, and open a channel from the attacker to the victim.

```sh
# Window 4
cmd/lncli-victim.sh getinfo # Copy the "identity_pubkey" from this
cmd/lncli-attacker.sh connect <pubkey>@localhost:30011 # Use the pubkey from the previous command

cmd/lncli-attacker.sh openchannel <pubkey> 2000000
cmd/btcctl.sh generate 10 # To confirm blocks for channel open
cmd/lncli-attacker.sh listchannels # To view the open channel
```

You should see the channel in the results of `listchannels` after this. Now we need to make a quick backup of the attacker's current state, where all of the channel capacity is on their side. Run the following:

```sh
# Window 4
cp -r nodes/attacker nodes/attacker-evil
```

We'll then make a large payment to the victim, that we'll try to steal back:

```sh
# Window 4
cmd/lncli-victim.sh addinvoice 1000000 # Copy "pay_req" from this
cmd/lncli-attacker.sh sendpayment --pay_req [pay_req] # Use the pay_req from before
cmd/lncli-attacker.sh listchannels # Confirm the "remote_balance" is now 1000000
```

With that payment made, you'll want to turn off your attacker node in window 3 with a `ctrl+c`, and copy the old directory back in:

```sh
# Window 3
mv nodes/attacker nodes/attacker-honest
mv nodes/attacker-evil nodes/attacker
```

Now let's assume some time has passed, and our attacker has been waiting for the victim's node to go offline. Go ahead and `ctrl+c` the node to stop it in window 2, and then let's start up the attacker in window 3 again:

```sh
# Window 3
cmd/start-attacker.sh

# Window 4
cmd/lncli-attacker.sh unlock # Enter 'password' from create
cmd/lncli-attacker.sh listchannels
```

You should see in the output of `listchannels` that the entire balance is back in `local_balance`, ready for the attack. Now all we need to do is broadcast the old state by doing a force close. Grab the `"channel_point"` from `listchannels` for the arguments to the next command, split where the colon is:

```sh
# Window 4
cmd/lncli-attacker.sh closechannel --force <channel-point-txid> <channel-point-index>
```

For instance, `"channel_point": "4c53040:0"` would be `closechannel --force 4c53040 0`.

Now if we bring our victim node back online, its watchtower will spring into action and rectify the situation. Go ahead and bring it back online:

```sh
# Window 2
cmd/start-victim.sh

# Window 4
cmd/lncli-victim.sh unlock # Enter 'password' from create
```

You'll know it all worked if you see the following in victim's logs in window 2:

```
[INF] WTWR: Found 1 breach in (height=1366, hash=<hash>)
[INF] WTWR: Dispatching punisher for client <victim-pubkey>, breach-txid=<breach-txid>
[INF] WTWR: Publishing justice transaction for client=<victim-pubkey> with txid=<justice-txid>
```

Mine a few more blocks, and you'll find the reward (minus fees) sitting in the victim's wallet:

```sh
# Window 4
cmd/btcctl.sh generate 10
cmd/lncli-victim.sh walletbalance
```

---

If you want to run through this all again, or you made a mistake along the way, just shut down your lightning nodes and run `cmd/restart.sh` to revert back to the original state.

## Troubleshooting

### `lnd / lncli / btcd / btcctl: command not found`

Make sure you installed all of the requirements, and they're accessible in your `$PATH`.

### `unable to parse private watchtower address`

Make sure you copied the victim's pubkey into `nodes/watchtower/lnd.conf` correctly.

### `Peer <pubkey> is not online`

Just rerun the `connect` command to have the node check if the other is online. When you restart one of your nodes, it can take a little time for them to see each other as online again.

### `Block number out of range`

Just generate a bunch of new blocks on btcd using `cmd/btcctl.sh generate 100`. It should fix itself.

### `cannot force close channel with state: ChanStatusDefault|ChanStatusBorked|ChanStatusLocalDataLoss`

Your attacker node somehow got informed that their state was out of date, so it won't let you try to do a malicious force close. This is probably because you left your victim node online when swapping in the malicious state.
