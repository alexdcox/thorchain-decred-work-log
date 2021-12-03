# Thorchain Decred Work Log

**Research**  
[23.11.2021 Tuesday](#23112021-tuesday)  
[24.11.2021 Wednesday](#24112021-wednesday)  
[25.11.2021 Thursday](#25112021-thursday)  

**Work**  
[29.11.2021 Monday 8hrs](#29112021-monday)  
[30.11.2021 Tuesday 7hrs](#30112021-tuesday)  
[01.12.2021 Wednesday 3hrs](#01122021-wednesday)  
[02.11.2021 Thursday 8hrs](#02122021-thursday)  

## 23.11.2021 Tuesday

Posted on the TC forum if anyone would be interested in me doing further development / looking into other chains.

Decred guys were interested.  
Said I'd estimate the work that needs to be done...  

The main branch is 125 commits behind the master, quite a lot of work needed there.  
Also seeing merge conflicts.  

https://gitlab.com/-/snippets/2147764  
https://github.com/decred/dcrd  
https://gitlab.com/vikingshield/decred  
https://docs.decred.org/  

Looks like he's aiming at `v1.6.2`.  
Is that the wallet or the protocol?  
The latest wallet version is currently `v1.6.3`  

There is `dcrd-testnet-insight` and `decred-insight-testnet`. A little confusing.  

`CMD ./dcrd -u user -P pass --notls --txindex --listen=127.0.0.1`

• Okay can I run decred in docker easily with vikingshield's image?

```bash
docker run \
  --rm \
  -it \
  registry.gitlab.com/vikingshield/decred
```

> mkdir: can't create directory '': No such file or directory

In contrast to the dash image:

```
docker run \
  --rm \
  -it \
  registry.gitlab.com/alexdcox/dash-core
```

That just runs. (admittedly needs the `-printtoconsole` arg to actually see that it's running but that's the dashd default, so I'm okay with that)

What's happening with the dcred image?

Using this to poke around a bit:
```
docker run \
  --rm \
  -it \
  --entrypoint /bin/bash \
  registry.gitlab.com/vikingshield/decred
```

```
docker run \
  --rm \
  -it \
  --entrypoint dcrd \
  registry.gitlab.com/vikingshield/decred
```

Okay that works fine, must be the entrypoint script.
It's trying to `mkdir $DECRED_DATA`, and that hasn't been given a sensible default.

```
docker run \
  --rm \
  -it \
  -e DECRED_DATA=/root/.dcrd \
  registry.gitlab.com/vikingshield/decred
```

> /scripts/entrypoint.sh: setting data directory to /root/.dcrd
> loadConfig: the specified subsystem [atadir] is invalid -- supported subsystems [ADXR AMGR BCDB BMGR CHAN CMGR DCRD DISC FEES INDX MINR PEER RPCS SCRP SRVR STKE TRSY TXMP]
> Use dcrd -h to show usage

Hmm, well I'm going to skip the entrypoint script and just use the binary for now.
`dcrd` binary running with default config (i.e. no user-defined config)

### 24.11.2021 Wednesday

```bash
docker run \
  --rm \
  --name decred \
  -it \
  --entrypoint dcrd \
  registry.gitlab.com/vikingshield/decred \
    --testnet \
    --rpcuser thorchain \
    --rpcpass thorchain
```

`dcrd version 1.6.2+release (Go version go1.16.4 linux/amd64)`  
So the image is currently one minor version behind.

• What are the default ports for Decred?

|                   | mainnet | testnet | simnet |
| ---               | ---     | ---     | --- |
| `dcrd` p2p        | 9108    | 19108   | 18555 |
| `dcrd` rpc        | 9109    | 19109   | 19556 |
| `dcrdwallet` rpc  | 9110    | 19110   | 19557 |
| `dcrdwallet` grpc | 9111    | 19111   | 19558 |

• Where are the default configuration files?  
/root/.dcrd/dcrd.conf  
/root/.dcrctl/dcrctl.conf  

```
docker exec -it decred dcrctl help
```

### 25.11.2021 Thursday

https://gitlab.com/vikingshield/thornode  
branch: 991-add-decred-chain  
https://gitlab.com/thorchain/thornode/-/merge_requests/1806  
gitlab.com/vikingshield/thornode  

• What does the PR situation look like with outstanding comments?

The tasks seem to be:
- Update to keep one day's block data in the cache
- Remove todos / redundant comments
- Use approproate constants where defined
- Switch string manipluated json with go structs/builtins and marshalJSON
- Node request id is hardcoded in one place
- 1851 PR changes requested
- Context.TODO left in
- 1757 PR changes requested
- Add fee rate estimation checks
- 1953 PR changes requested
- Remove a bitcoin txid check within the decred signer
- Remove version check

Most of these are just trying to maintain coding standards such as:
- Using constants where appropriate
- Correcting find and replace mistakes
- Using objects instead of string manipulation
- Removing redundant code/comments
- Addressing remaining TODOs.

I reckon I could go through every file that has changed from top to bottom and
clean things up within a day.

The big things on the list are the 3 PRs that have already been merged into dev,
that they're asking to be added. Going through those:
- 1851
  Ensure the UTXO to asgard is spendable
  around an hour
- 1757
  Bug: Parallelise mempool scan for BTC/BCH/LTC
  maybe a couple of hours or so
- 1953
  Cherry pick the changes in 0.69.3 back to develop branch
  not sure what they're actually asking from this one
  it doesn't seem to apply to decred

• How easy would a fast-forward merge from develop be?

There are 500 commits that vikingshield needs to catch up on,
everything since the 15th June 2021.

I pulled decred onto the current official develop branch and re-ran git vendor.
Then the tests:

```
go test -tags mocknet ./...
```

Great news, tests are passing.

So the bulk of the work on the actual `gitlab.com/thorchain/thornode` repo involves
tidying up what is already there, and then going through all the changes that have
been applied to the other bifrost clients since June, seeing which changes need to
be applied to Decred, and updating the PR with those changes.

Doing this will include the 3 requests from Heimdal plus any new changes since
he looked at the decred PR.

As a rough estimate I'll look at the diff between the bifrost bitcoin client on
vikingshield's branch compared to the latest official bifrost bitcoin client:

```
gitk --left-right origin/develop...vikingshield/develop -- ./bifrost/pkg/chainclients/bitcoin
```

Okay of those 500 commits only 34 seem to modify the bitcoin bifrost client.
The main things I see skimming through them are:
- log improvements +10m
- switch from error string to error constant/variable +10m
- parallelise mempool scan +2h
- add client method `isValidUTXO` and test `TestIsValidUTXO` +1h
- updates to how txscripts are parsed `txscript.ExtractPkScriptAddrs` +1h
- add locktime and spendable checks +10m
- add `reportSolvency` +20m
- removing spendable checks - in some places? +10m (need to question why now, can I skip, is this a different place, needs further research)
- add `isFromAsgard` check +20m
- gosec fixes, not sure why but doesn't look involved +10m
- remove redundant checks from signer +10m
- HAL-05 +10m
- add `getAddressesFromScriptPubKey` and tests +5h
- replace `getOutput` with `ParseMemo` +10m
- allow mimir to control UTXO (I don't think this applies to Decred from looking at the PR, need to confirm and research) +1h
- move signer cache into chain client +20m
- move signer cache store +10m

So around 12.5hrs of pure coding and we're update to date with the last change
made on the 8th November.

Those are very rough estimates, the easy changes might be even quicker, the more
complicated ones might require more research on my part to understand why the
change was made and how it should be applied to decred.

• What additional work is required to have this PR merged in:

- Updating `https://gitlab.com/thorchain/devops/node-launcher`

This has actually been the most costly part of getting Dash ready in terms of
development time and aws bills.

The code for thornode/bifrost took weeks, getting the node-launcher ready has
taken over 4 months now and I still can't say it's finished because other parts
of that infrastructure aren't working properly so my changes can't be tested yet.

Dash is a little different due to the requirement of multiple masternodes forming
a quorum before chainlocks can be tested.

I found aws to require quite a bit of wrestling. The terraform scripts would often
not teardown my dev environment properly, requiring a bit of manual intervention.
I dare say having worked with it before could bring this to weeks instead of months,
but it's difficult to put a time on this. It's already sprawled far beyond my
expectations with Dash.

I've spoken with Heimdal, he said lets focus on getting `thornode` changes merged
and then turn our attention back to the launcher later.

- Adding smoke tests to `Heimdall`:

Again, I haven't reached this point with dash yet.  
This is an example of the work required:  
https://gitlab.com/thorchain/heimdall/-/merge_requests/145/diffs  

Unsure how long this should take, more than a week or so would suprise me.

### 29.11.2021 Monday

Okay so I've estimated 2 days to get the PR updated and have been given the green
light to work on it. Let's go.

I already have the `991-add-decred-chain` branch from the `vikingshield` origin.  
Merging the develop branch in.

Not sure that commit `668f1fdbc8b21026995f159b2446867edfd2c527` actually needs adding  
The diff doesn't seem as clear as I first hoped, been undoing and redoing some changes  
Perhaps I had the direction round the wrong way:

```
gitk --left-right vikingshield/develop...origin/develop -- ./bifrost/pkg/chainclients/bitcoin
```

That looks better.  
Next step, apply all the changes that the bitcoin client has received that are applicable to decred:

```
git diff --color-words vikingshield/develop...origin/develop -- ./bifrost/pkg/chainclients/bitcoin
```

• Does Decred have RBF? I'm assuming no...  
Suppose I should confirm with the official repo...  
No mention of `^RBF$` or `^replace.by.fee$` so I'm assuming that logic is safe to skip.  
This site also says it "wont" have the RBF feature:  
https://blockgeeks.com/guides/decred/  

Okay I'm ripping out RBF logic for DCR... done.

• Decred doesn't support segwit either, good to know.

Opened up a new PR just to see where I'm at with things:  
https://gitlab.com/thorchain/thornode/-/merge_requests/2012/diffs#b2465110d55fe50c16ef9fc04f0b2d4d68fe7ca4  

Made some silly F&R mistakes myself and updated the bitcoin client, reverting to origin develop...
```
git reset develop --hard -- bifrost/blockscanner/blockscanner.go
git reset develop --hard -- bifrost/pkg/chainclients/bitcoin/client.go
git reset develop --hard -- test/fixtures/btc/sendrawtransaction.json
```
Sorted.

Now onto the todos, of which there are 18 accross 4 files:

--------------------------------------------------------------------------------

`decred.go`
- [x]  504:  check the logic of ExistsMempoolTxs against GetMempoolEntry!

Okay so there's actually no mention of `ExistsMempoolTxs` in any of the other clients.  
Looks like perhaps it's just a method renamed in decred to `GetMempoolEntry`?  
https://docs.decred.org/wallets/cli/dcrctl-rpc-commands/  
Yeah no getmempoolentry according to the docs. This is where I really need a node to compare raw output...  

```
docker run \
  -d \
  -it \
  --name decred \
  -v decred:/root \
  -p 9108:9108 \
  --entrypoint dcrd \
  registry.gitlab.com/vikingshield/decred \
    --rpcuser thorchain \
    --rpcpass thorchain

# watch container logs
docker logs -f decred

# watch block verification progress
watch -n 5 "docker exec decred dcrctl --rpcuser thorchain --rpcpass thorchain getblockchaininfo | jq '.verificationprogress'"

# watch docker volume size
watch -n 5 "docker system df -v | grep decred | tail -n 1"
```

Okay a day later, the node is running. The `existsmempooltxs` command doesn't
return anything other than yes/no, but that's all we need here, so all gravy
and biscuits.

- [x]  710:  check the encoding of tx---is it hexadecimal string of a byte-reversed hash, but any missing characters result in zero padding at the end of the Hash??

Following the source code brings you to a comment above  
`github.com/decred/dcrd/chaincfg/chainhash:Decode` which says:  

> Decode decodes the byte-reversed hexadecimal string encoding of a Hash to a
> destination.

As for the padding, if the hash is not an even length, a null byte is prepended,
before the whole thing is decoded into hex and then reversed.  
Not sure what the confusion regarding the padding is. We're going from a hex
string, which can be compressed - 0 hex char removed - to save a character.  
The result is a 32 byte hash regardless.

I get it. The `ExistsMempoolTxs` func signature needs a chainhash.Hash struct
and the question is how to convert a typical hex txid into that. Donezo.

- [x]  717:  check the logic: result != ""

That's already happening in the next line down. Perhaps this was addressed
without removing the todo, easy fix.

- [x]  740:  this just relies on dcrd command estimatefee. Is this good estimate?

How did I do it for dash?  
`git show 982-add-dash-chain:bifrost/pkg/chainclients/dash/client.go | subl`

Hmm, how do the other clients do it?  
BTC/DASH/LTC/BCH use the `averageFeeRate` from block stats, reported by the node.  
DOGE uses a similar estimatefee command to the proposed decred way.

We're not checking against the min relay fee though, all the others do.

That said the logic to resolve a less than min fee is to increment one satoshi,
which in a lot of cases I'd imagine would still not cut it. I don't follow that.

I think this is fine but will TODO: RAISE THIS WITH HEIMDALL

- [x]  926:  check what we do if get multiple addresses

All the chainclients have this. Must be copypasta. I'm going to leave this as it
ought to be addressed accross all chains at the same time.

- [x]  1080: verbose is ture?

That would surprise me. It is set to verbose, and I'm going to leave it like that
because there are `rawtx` references later on and I'm guessing we need verbose
for that data to be returned in the rpc response (it's often that way for dash)
I'll follow the adage: If it aint broke, don't fix it.

--------------------------------------------------------------------------------

`decred_test.go`
- [x]  194:  To address is the first vout that has value?

If the test passes, I'm happy. If not, I'll find out about it when I get there.
Removed.

- [x]  632:  independently translate this address according to (theres a link)

The start of this test sets the network to `mainnet`, and the link says the
mainnet prefix is `Ds` which is what I'm seeing in the test assertion below.
I don't see why we'd need to transalate this address into any other
network prefix unless we also change the network.
I'll leave the url and remove the todo (perhaps it was already addressed)

--------------------------------------------------------------------------------

`signer.go`
- [x]  38:   copied from BTC. is this good for DCR?

A fantastic question, I have no idea, it's talking about the `defaultMaxDCRFeeRate`.

Can I get this from the node? Wow google is absolutely no help for this query.  
Will have a look at the dcrd codebase quickly... okay found this in `mempool.go`:

```go
  // maxRelayFeeMultiplier is the factor that we disallow fees / kB above the
  // minimum tx fee.  At the current default minimum relay fee of 0.0001
  // DCR/kB, this results in a maximum allowed high fee of 1 DCR/kB.
  maxRelayFeeMultiplier = 1e4
```

Lets try math (see the next todo about estimating tx size for where I get the 250 estimation):
```
maxFeePerKb = 1e8
maxFeePerByte = maxFeePerKb / 1000 = 1e4
estimatedAverageTxSizeInBytes = 250

estimatedAverageTxSizeInBytes * maxFeePerByte = 250 * 1e4 = 2500000 (or DCR 0.025)
```

We're configuring a max fee per transaction, not per byte, so that gives us this
much room:
```
dcrutil.AtomsPerCoin / 10 = 1e8 / 10 = 1e7 (or DCR 0.1)
0.1 / 0.025 = 4
```

So we've picked a maximum fee that I'd guess is around 4 times bigger than the
fee's we should be paying. That seems alright to me.

- [x]  222:  Confirm these calculations remain accurate with Decred.  

This is about `estimateTxSize`  
The btc calc uses a base overhead, then a per-input/output cost, followed by null
data/memo cost 1 satoshi per byte. Apparently it's better to underestimate and 
guess 1 output.  
I suppose it depends on how many fields a raw decred transaction has compared to
bitcoin.  
https://developer.bitcoin.org/reference/transactions.html  
https://devdocs.decred.org/developer-guides/transactions/transaction-format/  
At a glance the inputs seem simplified with decred, there are a few extra
version and format fields in general, but it looks like witness data can be within
a transaction. I'm guessing that's not relevant here.  

Well, there is a golang dcrd fee estimator:  
https://github.com/decred/dcrd/blob/81ae286f2347/internal/fees/estimator.go  
It's not exposed as a public package though, which makes me think RPC is what
they're wanting you to use.  
We could do what Trustwallet does:  
https://github.com/trustwallet/wallet-core/pull/1000/commits/b386d6b0c651e3bd304fde9293976ea47aa53c53

```
mkdir -p ~/code/github.com/trustwallet/wallet-core && cd $_
git clone git@github.com:trustwallet/wallet-core.git .
```

They actually have specific rules for Decred, perfect :)

```cpp
int64_t LinearFeeCalculator::calculate(int64_t inputs, int64_t outputs, int64_t byteFee) const {
    const auto txsize = int64_t(std::ceil(bytesPerInput * (double)inputs + bytesPerOutput * (double)outputs + bytesBase));
    return txsize * byteFee;
}

...

class DecredFeeCalculator : public LinearFeeCalculator {
public:
    DecredFeeCalculator(): LinearFeeCalculator(166, 38, 12) {}
};

...

LinearFeeCalculator(double bytesPerInput, double bytesPerOutput, double bytesBase)
```

```
bytesPerInput  166
bytesPerOutput  38
bytesBase       12

inputs  x
outputs 1

txSize = ceil((bytesPerInput * inputs) + (bytesPerOutput * outputs) + bytesBase + bytesMemo)
txSize = ceil((166 * 1) + (38 * 1) + 12) = 216 + bytesMemo

txFee = txSize * byteFee
```

Okay that'll do nicely.

- [x]  272:  the tree argument: should this be always be 0?

Looking into the source I can see 3 `tree` options: unknown, regular, and stake.
We can rule out the other 2, so... deffo.

- [x]  274:  the valuein argument: should this be always be 0?

This is within `SignTx`
All the other clients set this to `nil`, but the func signature for decred requires
an int for the second argument, so I'm going to go with: yes. I also removed
`[]byte{txscript.OP_0, txscript.OP_0}` in place of `nil`, which I may regret.

- [x]  301:  check: is this true for Decred?

This is the following block of commented text:
> dcrd has a default rule max fee rate should less than 0.1 DCR / kb ??
> the MaxGas coming from THORChain doesn't follow this rule , thus the MaxGas might be over the limit
> as such , signer need to double check, if the MaxGas is over the limit , just pay the limit
> the rest paid to customer to make sure the total doesn't change

We're looking at `decred/signer:SignTx`. I'll try and break this down so I can
better understand what I'm looking at here...

`client.getGasCoin` returns the sats per byte fee for the last transaction
recorded in the thornode local storage for the chain in question.
If the fetched feePerByte multiplied by the estimated tx size to sign is greater
than the max fee, we use the max fee instead.
If the fee estimate is less than the minimum relay fee, we take some funds from
the payout to customer to cover up to the minimum fee.

`SignTx` is only called from `bifrost/signer/sign.go:Signer.signAndBroadcast`
That is called by `Signer.processTransactions`  called by
`Signer.signTransactions` which is looped when `Signer.Start` is called, during
bifrost startup.

The max gas is set on the txout, and where that happens, along with how the gas
is actually observed and set is a bit of a can of worms for my mind right now.
I think many places. Then perhaps thrown into the global transaction queue ready
to be handled by the loop.
h.mgr.GasMgr().AddGasAsset
hander_common_outbound
handler_observed_txout.go
handler_observed_txout_archive.go

Right what am I trying to do again... I think it will work for Decred. All the
other clients are working in the same way. We'll all know if this is wrong when
we smoke test.

- [x]  399:  Check this: Decred does not have virtual size

That's okay, the BCH client doesn't use this either.

- [x]  415:  check: why is the last argument not amount?

I can't admit I understand this either.  
The `redeemTx` argument will already have the utxo values, I don't see the need
for passing in the `amount` again.  
My best guess is the vin signatures for BTC/BCH are constructed using the input
amount, whereas DCR uses the references to the amount instead, so we don't
actually need to say what the amount is, that's checked later by the
node/txscript engine. Regardless of my understanding, the official decred golang
library `txscript.RawTxInSignatureSignable` func doesn't ask for the amount, so
I'm not going to provide it, and assume that they know what they're doing :)

Removing this argument in turn simplifies the `for idx := range redeemTx.TxIn`
loop in `SignTx`. This is another thing I may regret removing in time, looking
forward to testing this...

- [x]  451:  is this flag good enough?

We have `txscript.ScriptDiscourageUpgradableNops` used in `txscript.NewEngine`
within `signUTXO`.

> This flag must not be used for consensus
> critical code nor applied to blocks as this flag is only for stricter
> standard transaction checks.

I don't think it's relevant. Relaced with `nil`.

- [x]  493:  check: btcjson.ErrRPCTxAlreadyInChain == dcrjson.ErrRPCDuplicateTx?

From dcrd:
> TestDuplicateTxError ensures that attempting to add a transaction to the
> pool which is an exact duplicate of another transaction fails with the
> appropriate error.

Good enough for me.

--------------------------------------------------------------------------------

`walletrpc.go`

- [ ] 216:  check: rescan is set to false. Is this correct behavior?

This is related to importing a public key for the node to watch, a command
which decred doesn't actually seem to have.

https://docs.decred.org/wallets/cli/dcrctl-rpc-commands/

Oh vikingshield modified `dcrwallet`, gotcha. I FORGOT ABOUT THIS ONE! TODO: GO BAAACK

`importpubkey`

There's absolutely no mention of this in dcrd. Hold up, what about the wallet?

`go get github.com/decred/dcrwallet`

> go get: github.com/decred/dcrwallet@v1.7.1: parsing go.mod:
>   module declares its path as: decred.org/dcrwallet
>           but was required as: github.com/decred/dcrwallet

Oh :( That's not what you want. They're ignoring the golang module name rules?

```
mkdir -p ~/code/decred/dcrwallet && cd $_
git clone git@github.com:decred/dcrwallet.git .
```

No `importpubkey` found. I did find `importscript` and `importxpub`, also
`CreateWatchingOnlyWallet` but that doesn't look like rpc accessible.
Maybe importscript is the one we need.

Just asked in the decred discord, apparently, create watching only wallet is the
one we need!

Alright lets try and get that working...  
This is the final todo and I'm approaching the end of the day so I'll just
continue the todos.

### 30.11.2021 Tuesday

I'm still continuing with the todo's above from yesterday, just going to leave
them where they are instead of writing in today's log for readability.

• Just want to note that my decred container has caught up and the block size is
10.77GB for mainnet with default config (i.e. no indexes specified)

Okay lets get some info for the `check with node`s above:
```
docker exec -it decred bash
alias dcrctl="dcrctl --rpcuser thorchain --rpcpass thorchain"
```

```
dcrctl getrawmempool
```

```
[
  "f1cc4968774b59d3f1f37713327db474baf362fb40f7e7f9ddcea11a352799da",
  "6282d1f2bd57e083c001199df6e22f2a7915223d35ff4aab25bbdb63637006d0",
  "f4989b85955d0a60ae65408709e6ffd68b563f92bae30f03cedfb7190ea94114",
  "a9f5bf17dd147f4422a2482295100ffb86042582a5686d564153443413e70f51",
  "d6a868618cb5bf1a86c0f843c30273712b453e951995c08aaaa7f050459cd387",
  "f8479c5ac5db60554787a5b787840dd56022f5b75949dee4938e6bb48abcecd5",
  "d9e1d6d3bcc0abbf4fdccf556fecd3e36ec36b91954cd61b175c695c4dea036f",
  "132b8640ca10d8e02d3a2c9dca9bb0a77d04beda54e53689f9656909fa7e9040",
  "01b64926d6a29a8459a52ffa96965ee6fb60feea0a8c9231b72a2fcef49dc78a",
  "dd22aa03345c10054085f4cc008f9f7ee09f259c1cd5f1ac72b2922326db9299",
  "a656030f354346272dcd8070e03a173c4158b9e12c8dcc382a4f129b410249b0",
  "ca32b348ff7fa77fcdbe7d28cf77aa7bc78ba1e8117bf7ecd4a4bf809be281a7",
  "1adc43ba7f4caf132857bf7bcfcaf18f3cfb7c99890f6cb755bd3acd81468d12",
  "c38782a81d09e81c324527022b6524914d4b2480abe872c094fd95cb6eac0b6a",
  "c3062ac0b398102b80b994eed4fdbbb63020a2237f37882f9f95702bdca540f4",
  "7fe58bfd31ee3a43283447959524a086444b0c9f7f01b0b508c8cb027440197c",
  "39dfb8a04791645bdc0dc8f8998e563f8cad76c1322ecc299e5532dec5a7ea01",
  "a90b499c386c3f677fc152825082d34ff4fd0fb6891d6e97afaae1b111fea935",
  "75d23f48ba2c28c0e550a7b492c70667ab0dac34202bde0b2dd8940d4a7ab5a9",
  "16dcdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e",
  "8260bcc88fec6a0045f5338f75a12c90a3a616fdafa3eeb522e6def11e6726cc",
  "658170f3bcee2592f38bdd2793d68497a7a8d4c0692bd9b459083b3e707e553d",
  "854aa49e680020414982ce73fc5025730b8bcc26b7e05a1989ffe85bfe572200",
  "b6437be393e61df02c47778d5d00ea9acf954d853299e98288dfdb1eca36414f",
  "d3cd480a9ac41fbef4c1389caa54718d0161d50aa5934b750b322be957af8a67",
  "a4519880dcbbdb3ef53f25d01608108298866b980f5e58311e10332935f04ad0",
  "2ac9d93d1b83c4ea66e1f4ced0176449b9632a5c720f97c17a1b91fab03a45d7",
  "84a9e958e0fbe72b10ccde640b4a4da22aaf6fac69b816dd6437a90d6269cfa3",
  "3150c611b68f53c1f21ffd9d3c656f23475697dd9a869b8746ef029fd6bb2c0b",
  "851d4524e74580f87fc8904827fb4b8e1903840ff857aae3c714d0a1012ca7ad",
  "c1c4a6b95fd7efa66bf374bae423c70826eee9cf86fca34f2802f329db6c9373",
  "2840caad84a22c965a9ffaec0fc5592c942c9bed2c1a60f4ddb0302da0279e27",
  "86fac87e4b030503ae05122bb3586dbca6345122125e16ab9bf412e85d273924",
  "8613fde5cb04e2572d7dbc85492838e3bb5d381adb200f33ebd6e56f173a7010",
  "16d6dd2ce0ba98da98b47f7f319323f116f6a5acd197a97194e932971355103d",
  "a867d6b82cbbdea8e1c19c7373b5d0a7019e279a56c7f9f18b6bb49371e8f677",
  "1ae15ec61528964e7be858e18ea7a033208052f074ee739b4dbe42e2341e9aa1",
  "2b7bff50af83fec9f4fa8bfc94cb9b537dd4f1748582b4fa8647c6e4b0e90227",
  "f14837f2ce2b984ac96066a468db5b4525b24b0c535f7efacb8ab59530f755cd",
  "d57341964f43753adf09e51b79122ee74b788fee03795cd660db8509b2985fe8",
  "b6a22ea303629d70082ac93e0a05388303bc817e96381d6cc7f593ca921fd3f0",
  "a287c74feb48cbbf27226f4b982a617fcd78616f5955b30608e19c14966825b3",
  "2145cbfda3a341da7cf9451ff7e99ea245f41ddfab9291e9a4fe56ec70193ab0"
]
```

```
dcrctl existsmempooltxs 16dcdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e
dcrctl existsmempooltxs '[\"16dcdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e\"]'
dcrctl existsmempooltxs \[\"16dcdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e\"\]
```

Third time lucky, it returned:
```
01
```

And for a missing txid:
```
dcrctl existsmempooltxs \[\"111cdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e\"\]
```

```
00
```

Strange there are two digits returned there.

```
dcrctl getmempoolentry \[\"16dcdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e\"\]
```

> Unrecognized command "getmempoolentry"

Well, of course, silly me.

```
dcrctl getrawmempool true
```

```
{
  "47f03fbda26773cae25dea43260a86123bf4e478d4bd053822f302a8aceec2fd": {
    "size": 296,
    "fee": 0.0000298,
    "time": 1638317301,
    "height": 611873,
    "startingpriority": 0,
    "currentpriority": 160020187.6818182,
    "depends": []
  },
  "51661732f202987c6bc14d236086936ab6d08d8550766e0cba48f64e528a17df": {
    "size": 296,
    "fee": 0.0000298,
    "time": 1638317301,
    "height": 611873,
    "startingpriority": 0,
    "currentpriority": 160020187.6818182,
    "depends": []
  },
  "55bbc884166062b74b192bc29861fe4bad8e19ffff34ffa2051a794273c740c3": {
    "size": 297,
    "fee": 0.0000298,
    "time": 1638317301,
    "height": 611873,
    "startingpriority": 0,
    "currentpriority": 160020187.6818182,
    "depends": []
  },
  ...
}
```

```
dcrctl getrawtransaction 16dcdb32b7d003cb39bbe61490cc9ccb7a62ed35b323bb04fc9845f64abf1d3e
```

> -32603: The transaction index must be enabled to query the blockchain (specify --txindex)

Hmm. Lets check `thornode/test/fixtures/btc/...` oh - it's not there.
Was expecting a `getmempoolentry.json` fixture.

```
dash-cli getrawmempool
dash-cli getmempoolentry \[\"ce83854ad6461c56f06c092199fdc64c1480654e3478f52aee49e1fc4945a965\"\]
dash-cli getmempoolentry ce83854ad6461c56f06c092199fdc64c1480654e3478f52aee49e1fc4945a965
```

```
{
  "fees": {
    "base": 0.00000226,
    "modified": 0.00000226,
    "ancestor": 0.00000226,
    "descendant": 0.00000226
  },
  "size": 225,
  "fee": 0.00000226,
  "modifiedfee": 0.00000226,
  "time": 1638317598,
  "height": 1580628,
  "descendantcount": 1,
  "descendantsize": 225,
  "descendantfees": 226,
  "ancestorcount": 1,
  "ancestorsize": 225,
  "ancestorfees": 226,
  "depends": [
  ],
  "spentby": [
  ],
  "instantlock": true
}
```

At first glance it looks like the decred getrawmempool with verbose output will
do the trick.

• Decred DOES have the concept of `witness` to a transaction, but it is stored
within transactions on the block instead of segregated as is the case with
bitcoin. I'm assuming this is thanks to a greater block size limit.

Alright there's only one todo that I haven't been able to fully address at the
moment and that is because it requires a bit more research.

There's `dcrd`, the node software, and a separate `dcrwallet` deamon/program for
wallet functionality. Intersting design choice, most chains implement wallet code
along with the core platform code.

They both have separate rpc interfaces. Quite different ones on first glance.
I assume the node I've spun up doesn't even have the `dcrwallet` binary running.
Unless running `dcrd` also has the logic for forking a child process for
`dcrwallet`.

This leads me with quite a few questions. I'm now thinking launching the docker
container with `--entrypoint dcred` is not enough for testing with thorchain as
we also need the wallet binary running.
- Does the `dcrwallet` binary exist in the container?
- Does the entrypoint script start both the wallet and the node?
- What port does the wallet run on? (so I can expose that port)

Alright time to work through those...

```
which dcrwallet
```

> /usr/local/bin/dcrwallet


Okay, that's good. The wallet binary is built into the container.

```
grep -ir "dcrwallet" /scripts/entrypoint-mock.sh
```

> dcrwallet --nogrpc --noservertls  --simnet --username=${SIGNER_NAME} --password=${SIGNER_PASSWD} --pass=${SIGNER_PASSWD} --rpclisten=$(grep $(hostname) /etc/hosts | cut -f 1) --rpclisten=127.0.0.1

Yep. Fair enough. The entrypoint script already starts the wallet.

Alright now to get the container running via the default entrypoint script
because running it without any arguments fails.

First things first: can I actually configure the current image to work at all?

So to refresh our memory: running it without any environment variables resulted
in the error:

> mkdir: can't create directory '': No such file or directory

Running with the `DECRED_DATA` env. variable configured - the only variable
referenced in the default `/scripts/entrypoint.sh` script - results in:

```
docker run \
  --rm \
  -it \
  -e DECRED_DATA=/root/.dcrd \
  registry.gitlab.com/vikingshield/decred
```

> loadConfig: the specified subsystem [atadir

The problem is this line:
```
  set -- "$@" -datadir="$DECRED_DATA"
```

`datadir` requires a double dash `--` as it's a multiple character argument.
Easy mistake, but it does mean I need to rebuil the image with a valid script
built in at some point.

For now I'll just override that script with one from my dev machine.

```
docker run \
  --rm \
  -it \
  -e DECRED_DATA=/root/.dcrd \
  -v decred:/root \
  -v /Users/adc/Desktop/thorchain-decred/entrypoint.sh:/scripts/entrypoint.sh \
  registry.gitlab.com/vikingshield/decred
```

Getting this:

> 2021-12-01 05:19:16.653 [ERR] DCRD: stat /root/.dcrd/mainnet/blocks_ffldb/metadata: permission denied: stat /root/.dcrd/mainnet/blocks_ffldb/metadata: permission denied


### 01.12.2021 Wednesday

Okay I need this node working with the modified wallet binary running and accessible.

```
# testnet entry

docker run \
  -it \
  -p 19108-19111:19108-19111 \
  -e DECRED_DATA=/root/.dcrd \
  -v decred:/root \
  -v /Users/adc/Desktop/thorchain-decred/entrypoint-testnet.sh:/scripts/entrypoint-testnet.sh \
  --rm \
  --name decred \
  --entrypoint /scripts/entrypoint-testnet.sh \
    registry.gitlab.com/vikingshield/decred

# no volume, bash entry

docker run \
  -it \
  -p 19108-19111:19108-19111 \
  -e DECRED_DATA=/root/.dcrd \
  -v /Users/adc/Desktop/thorchain-decred/entrypoint-testnet.sh:/scripts/entrypoint-testnet.sh \
  --rm \
  --name decred \
  --entrypoint bash \
    registry.gitlab.com/vikingshield/decred
```

```
grep -ir "testnet" /scripts/
```

No entrypoint script for testnet. We'll need that for... testing.
Also seeing the `--nogrpc` argument supplied to `dcrwallet`, so I'll have to
remove that if I want to use `CreateWatchingOnlyWallet`.

Getting this error:

> loadConfig: mining address 'SsXciQNTo3HuV5tX3yy4hXndRWgLMRVC7Ah' failed to decode: unknown address type

Probably was a missing testnet flag.

```
curl \
-i \
-u thorchain:password \
-X POST \
-H "Content-Type: application/json" \
--data  '{"jsonrpc":"1.0", "method":"createwatchingonlywallet"}' \
localhost:19110 
```

> {"jsonrpc":"1.0","result":null,"error":{"code":-32601,"message":"Method not found"},"id":null}


```
curl \
-i \
-u thorchain:password \
-X POST \
-H "Content-Type: application/json" \
--data  '{"jsonrpc":"1.0", "method":"walletrpc.WalletLoaderService/CreateWatchingOnlyWallet"}' \
localhost:19110 
```


> Unrecognized command "importpubkey"


Looks like he may have only added the `importpubkey` command to the legacy
jsonrpc interface.

```
curl \
-i \
-u thorchain:password \
-X POST \
-H "Content-Type: application/json" \
--data  '{"jsonrpc":"1.0", "id": 1, "method":"importpubkey", "params": ["test1"]}' \
localhost:19110 
```

```
dcrctl \
  --rpcuser=thorchain \
  --rpcpass=password \
  importpubkey
```

```
alias dcrctl="dcrctl --testnet --rpcuser=thorchain --rpcpass=password --notls --wallet"
dcrctl createnewaccount alex
dcrctl getmasterpubkey alex
```

```
curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"importpubkey", "params": ["tpubVogJaho4zn3z5xZTqe8FZitorpFhb4Dp8JAVtmBZuw26QaQEf2yUj6n3vA8hDKNES8ZJYhwbgfSBDZAiymo6EpfvdbFimLQf7vJ8ywid9K6"]}' \
  localhost:19110
```

> {"jsonrpc":"1.0","result":null,"error":null,"id":1}

No error, no result. Interesting. Perhaps it needs the pubkey in a different
format?

Was distracted by other tasks, half day today.


### 02.12.2021 Thursday

So, the `importpubkey` response was a bit confusing. Let's try with a simple
hex pubkey.

Here's a random new ecdsa keypair:
```
private:               d2b15503374495a660249e203fdbb3a0196fc214b9d22fa1ff1f869a989e9c86
public (uncompressed): 048e75c235571830610035a69c0b5122cc03dac1323f98ff9f5eb11347844431d4b8ff4fa094209d7cea32548149b7879b5e7c9e9501e47d7817fd11a64ea3d10f
public (compressed):   038e75c235571830610035a69c0b5122cc03dac1323f98ff9f5eb11347844431d4
```

Does it take the pub hex compressed?

```
curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"importpubkey", "params": ["038e75c235571830610035a69c0b5122cc03dac1323f98ff9f5eb11347844431d4"]}' \
  localhost:19110
```

> {"jsonrpc":"1.0","result":null,"error":{"code":-32600,"message":"Invalid request"},"id":null}

Okay so perhaps it does need the tpub/pub format. 

I have a feeling that isn't going to be enough, we need the following methods:
- importpubkey
- listunspent
- getnetworkinfo

```
curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"getnetworkinfo", "params": []}' \
  localhost:19110
```

`getnetworkinfo` works fine.


```
alias dcrctl="dcrctl --testnet --rpcuser=thorchain --rpcpass=password --notls --wallet"
dcrctl createnewaccount alex
dcrctl getmasterpubkey alex

```

`dcrctl listaccounts`
```
{
  "alex": 0,
  "default": 0,
  "imported": 0
}
```

Probably important to note `imported` becomes a separate `account`.

```
dcrctl getnewaddress alex
```

```
curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"listunspent", "params": [0, 99999, ["TskLJuhkrBSqxbL2pa9oCnwTP7RzB7mkNsz"], "imported"]}' \
  localhost:19110
```

> {"jsonrpc":"1.0","result":null,"error":null,"id":1

Looks okay. I'm going to leave the grpc stuff here for a mo

```
  // "localhost:19111"

  conn, err := grpc.Dial(c.walletcfg.WalletRPCServer)
  if err != nil {
    fmt.Println(err)
    return
  }
  defer conn.Close()

  walletClient := pb.NewWalletLoaderServiceClient(conn)
  _, err = walletClient.CreateWatchingOnlyWallet(
    c.defaultRPCContext(),
    &pb.CreateWatchingOnlyWalletRequest{
      ExtendedPubKey:   pubkeyHex,
      PublicPassphrase: nil,
    },
  )
  if err != nil {
    return err
  }
```

Okay on with seeing how things went...

```
go test -tags mocknet ./...
```

TestEstimateTxSize broke, understandably, because I updated the logic there.

Beaut, all now passing. Going to double check I've addressed all the PR comments
from Heimdall.

Just found this in `TestRPCClient`:
> This test needs Decred mockent dcrd/dcrwallet running at standard ports

We should be mocking http responses from dcrd/dcrwallet so the tests can run
without dependencies. I'm wary. Re-running the tests without a local decred node
accessible... all is well. Glossing over...

Sent Heimdall a message about: `EstimateAverageTxSize*uint64(feeRate) < c.minRelayFeeSats`

I removed walletrpc_test because it wasn't actually testing anything that is used
by the decred module.

Okay testing with my stack now...


```
# no volume, bash entry

docker run \
  -it \
  -p 19108-19111:19108-19111 \
  -e DECRED_DATA=/root/.dcrd \
  --rm \
  --name decred \
  --entrypoint bash \
    registry.gitlab.com/vikingshield/decred
```

I'm going to re-save the vikingshield image with `socat` and `jq` installed,
for use with my scripts:

```
docker run --name decred --rm -it --entrypoint bash registry.gitlab.com/vikingshield/decred

apk add socat jq

docker commit decred registry.gitlab.com/vikingshield/decred
```

`dcrdctl --rpcserver=$1 getblockchaininfo 2>/dev/null | jq -r '.verificationprogress' 2>/dev/null`

So I've noticed that the `notls` option cannot be used when allowing connections
from any ip other than localhost. Other nodes don't enforce this. I don't really
want to be using TLS with all the additional configuration and complications that
goes with it for https, I'd rather handle this via network security (or not at
all for development)

I COULD just tell it to listen on a different port and forward all external
traffic to that port through `socat`, so it looks like local traffic, thereby
circumnavigating the tls requirement.

Is this wise? I'm not preventing it from being configured in the official
thornode repo, so I think it is wise for this dev cluster. Keep it simple.

19556 -> 20556

```
socat TCP-LISTEN:19556,fork TCP:localhost:20556 &
```

```
while true; do clear; docker logs -f decred1; sleep 1; done
while true; do clear; docker exec -it decred1 bash; sleep 1; done

while true; do clear; docker logs -f decred2; sleep 1; done
while true; do clear; docker exec -it decred2 bash; sleep 1; done

while true; do clear; docker logs -f thornode1; sleep 1; done
while true; do clear; docker logs -f thornode2; sleep 1; done

while true; do clear; docker logs -f bifrost1; sleep 1; done
while true; do clear; docker exec -it bifrost1 bash; sleep 1; done

while true; do clear; docker logs -f bitcoincash1; sleep 1; done

cat /root/.dcrd/dcrd.log
```

Had to write the .dcrctl config file too.
Now I have a cluster, 2 thornodes, 2 bifrost, 2 decred, 2 bifrost.

Actually that's a lie.
> standard_init_linux.go:228: exec user process caused: no such file or directory

thornode and bifrost are reporting that error. Maybe needs an image rebuild?

```
docker run \
  -it \
  --rm \
  --name thornode1 \
  --entrypoint /bin/bash \
  github.com/alexdcox/thornode:mocknet

thornode start --log_level debug --log_format json --rpc.laddr tcp://0.0.0.0:26657

```

Oh I probably built my docker container with bash, socat and jq installed...

```
docker run -it --rm --name thornode1 --entrypoint sh github.com/alexdcox/thornode:mocknet
apk add bash socat jq
docker commit thornode1 github.com/alexdcox/thornode:mocknet
```

For some reason `DASH` is still showing up in the inbound addressses list, despite
me not configuring that with the bifrost.

`cat /etc/bifrost/config.json`

Okay here are some errors from the bifrost logs:

> {
>     "level": "fatal",
>     "service": "bifrost",
>     "module": "bifrost",
>     "error": "fail to create block scanner: Post \"http://decred1:19557\": dial tcp 172.32.62.1:19557: connect: connection refused",
>     "chain_id": "DCR",
>     "time": "2021-12-03T05:51:53Z",
>     "message": "fail to load chain"
> }

> {
>     "level": "fatal",
>     "service": "bifrost",
>     "module": "bifrost",
>     "error": "fail to create block scanner: Post \"http://decred1:19557\": EOF",
>     "chain_id": "DCR",
>     "time": "2021-12-03T06:10:11Z",
>     "message": "fail to load chain"
> }

• The chainclients actually match the `BlockScannerFetcher` interface. Important
realisation there. The importance of that is the three main methods:
- FetchMemPool
- FetchTxs
- GetHeight

One of these is causing the EOF error above, and I'm assuming it's get height.

```
curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"getblockcount"}' \
  localhost:19556

curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"getblockcount"}' \
  localhost:19557

curl \
  -i \
  -u thorchain:password \
  -X POST \
  -H "Content-Type: application/json" \
  --data  '{"jsonrpc":"1.0", "id": 1, "method":"importpubkey"}' \
  localhost:19556
```

The above curl requests demonstrate the issue here. For `decred` we need to configure
both the node rpc and the wallet rpc. Thornode/bifrost isn't designed for having two
rpc endpoints. I'm not entirely sure the best way to continue.

The `BlockScannerConfiguration` struct only has one field for the rpc host:
```go
type BlockScannerConfiguration struct {
  RPCHost                    string        `json:"rpc_host" mapstructure:"rpc_host"`
  StartBlockHeight           int64         `json:"start_block_height" mapstructure:"start_block_height"`
  BlockScanProcessors        int           `json:"block_scan_processors" mapstructure:"block_scan_processors"`
  HttpRequestTimeout         time.Duration `json:"http_request_timeout" mapstructure:"http_request_timeout"`
  HttpRequestReadTimeout     time.Duration `json:"http_request_read_timeout" mapstructure:"http_request_read_timeout"`
  HttpRequestWriteTimeout    time.Duration `json:"http_request_write_timeout" mapstructure:"http_request_write_timeout"`
  MaxHttpRequestRetry        int           `json:"max_http_request_retry" mapstructure:"max_http_request_retry"`
  BlockHeightDiscoverBackoff time.Duration `json:"block_height_discover_back_off" mapstructure:"block_height_discover_back_off"`
  BlockRetryInterval         time.Duration `json:"block_retry_interval" mapstructure:"block_retry_interval"`
  EnforceBlockHeight         bool          `json:"enforce_block_height" mapstructure:"enforce_block_height"`
  DBPath                     string        `json:"db_path" mapstructure:"db_path"`
  ChainID                    common.Chain  `json:"chain_id" mapstructure:"chain_id"`
}
```

This is a generic config struct used for all block scanners across all chains, so
it doesn't feel right adding another field just for decred.

Other options...?

Okay well vikingshield has already created the `WalletConfig` struct with 
an `RPCServer` field. Is there logic to read this config value from somewhere?

Hmm, he actually sets that field from the block scanner config, using the same
`cfg.RPCHost` value for both:
```go
  walletcfg := WalletConfig{
    RPCUser:         cfg.UserName,
    RPCPassword:     cfg.Password,
    RPCServer:       cfg.RPCHost,
    WalletRPCServer: cfg.RPCHost,
    Wallet:          true,
    AuthType:        authTypeBasic,
    NoTLS:           cfg.DisableTLS,
    PrintJSON:       false,
    TLSSkipVerify:   true,
  }
```

There's also a generic `ChainConfiguration` struct with `RPCHost`. Anything chain
specific anywhere?

Maybe that's the answer. Set the block scanner and chain rpc host differently.
Although chain rpc to me says block info, so does block scanner. I'd rather not
confuse other developers if I can help it...

vikingshield set `localhost:19557` as the `ChainConfiguration.RPCHost`, which is
the wallet rpc endpoint.

Hmmmmm.

1. Create a facade/proxy which redirects requests according to the rpc `method`
2. Configure chainconfig.rpchost and blockscanner.rpchost differently, document
   the decred necessesity, be careful to use the right one in the right place
3. Hard-code a wallet rpc port, or derive it from the node rpc port
   (i.e. tell people the wallet RPC HAS to be one port higher than the node port,
   which is the default)
4. Modify thorchain to allow for additional chain-specific config

</br>

1. Is time consuming and another codebase to maintain
2. Isn't super clear, might confuse devs, probably breaks other devops flows which
  are coded with the assumption that they're the same value, as is the case with
  all the other chain clients
3. Involves a new convention, which needs to be clearly communicated and followed
  with the risk of breaking things otherwise
4. Is the approach I'd take if I was the maintainer, but I'm not, and it
   introduces changes outside of the decred bifrost client which would require
   more intense review when it comes to PR this

I'm thinking something like this on the `ChainConfiguration`:
```go
Extra           interface{} `json:"extra" mapstructure:"extra"`
```

If I'm guessing correctly, adding just that single line would be enough. Setting
`extra` in the bifrost config like:
```
    {
      "chain_id": "DCR",
      "rpc_host": "decred1:19556",
      "username": "thorchain",
      "password": "password",
      "http_post_mode": 1,
      "disable_tls": 1,
      "block_scanner": {
        "rpc_host": "decred1:19556",
        "enforce_block_height": false,
        "block_scan_processors": 1,
        "block_height_discover_back_off": "5s",
        "block_retry_interval": "10s",
        "chain_id": "DCR",
        "http_request_timeout": "30s",
        "http_request_read_timeout": "30s",
        "http_request_write_timeout": "30s",
        "max_http_request_retry": 10,
        "start_block_height": "0",
        "db_path": "/var/data/bifrost/observer/"
      },
      "extra": {
        "wallet_rpc_host": "decred1:19557"
      }
    }
```
Would be automatically picked up when the main bifrost configuration object is
loaded. Have 30mins on the clock left for today, lets see how viable (4) actually is...

Yep. That worked. I'll send the options to heimdall and see what he thinks.

