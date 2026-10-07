> **Experimental. We are just trying this.**
>
> Good intentions, shaky hands. STP does not know what he is doing. We test, we write down what we think we saw, and that is the whole product. A number here is not the truth. A chart is not the truth. Any other sentence that sounds sure of itself is not the truth either. Do not count any of it as a claim.
>
> [Disclaimer](DISCLAIMER.md)

# What limits the transaction rate

This page is the mass ceiling. It is not a run log. The other notes stay separate, and each one keeps its own disclaimer.

| Note | What it is |
|---|---|
| [GROK-BUILD-PROMPT.md](https://github.com/STP-KAS/tn10-storm-throughput-questions/blob/main/plan/GROK-BUILD-PROMPT.md) | Paste-in for Grok Build on 9 or 13 Oct. Ready. It does not start the storm. |
| [tn10-storm-throughput-questions](https://github.com/STP-KAS/tn10-storm-throughput-questions) | The questions, the method, and the empty result sections. Runs listed oldest first. |
| [tn10-build-desk-tps](https://github.com/STP-KAS/tn10-build-desk-tps) | Desk runs, oldest first. The six-hour 2,207 is there. |
| [tn10-build-desk-tps-3500](https://github.com/STP-KAS/tn10-build-desk-tps-3500) | 7 Oct included-rate tries, oldest first. 3,500 was not read. |

## Why

Live Kaspa aims at about 10 blocks a second. That rate is Crescendo. It is the rate the difficulty adjustment holds. rusty-kaspa v2.1.0 is the node pin. That release is not a consensus upgrade.

Each block has three mass budgets in that release: compute 500,000, storage 500,000, transient 1,000,000. A plain one-input, one-output payment is about 1,624 grams of compute mass. 500,000 / 1,624 is about 308 of those payments in one block. At 10 blocks a second the same shape is about 3,080 payments a second.

That number is the ceiling for this shape of transaction. A worker thread, a miner thread, a second RPC hostname, or a higher fee does not add grams to the block.

About 100 blocks a second is not live. DAGKnight is KIP-2, and KIP-2 is Proposed. A teaching wiki does not turn it on.

## What a full block looks like

On 7 Oct 2026 a Testnet-10 desk node was accepting ordinary payments into blocks of about 304 transactions. Compute mass on those samples was about 498,800 of 500,000. The network was already at the ceiling, about 3,040 transactions a second.

One sender's own included transactions stayed around 1,500 to 1,770 a second. A higher fee changed which of that sender's own transactions got the slots. The combined included rate stayed under 1,800. The rest of the full block was other traffic.

While that higher fee was being submitted, public mempools climbed into the 90,000s. One public node touched about 99,000 and kept answering. After the extra senders stopped, the pools fell back to about 60,000. The chain rate did not move.

Several public URLs can be one machine. Count machines before you count nodes.

## How a full mempool panics

In rusty-kaspa v2.1.0, `mining/src/mempool/validate_and_insert_transaction.rs` asserts before it inserts. The insert is refused by that assert when the pool would pass `maximum_transaction_count`, or when the byte cap would break. The message prints the count plus one.

The default count is 1,000,000. `--ram-scale` only scales that count down, and only down to the value you pass. `--ram-scale=0.1` makes the cap 100,000. The next insert then dies as 100,001 against 100,000.

The pool is built to answer that it is full, and to drop a cheaper transaction to make a slot. The assert runs when that eviction did not free a slot and the function continues. The node exits. It does not keep the transactions it already had.

The fix in the node is small. If the slot is still not free, return the mempool-full error and leave the process up. Do not assert. A public node that expects congestion should stay on the default scale of 1. Scale 0.1 is the configuration that dies at 100,000.

This desk has not rebuilt kaspad with that return. The desk's own Testnet-10 node is on the default scale. The panic line is a public node that was started with the low scale.

A sender can avoid the line without that patch. Stop the extra submitters while a public mempool is still climbing, before 100,000. Crossing the line takes the node down. That is lost availability. It is not a higher rate.

## How the rate actually goes up

The network rate goes up in three ways.

- More blocks per second. That is a consensus change. It is not a node flag, and 100 blocks a second is not live.
- A higher mass cap. Same kind of change. Not a flag.
- A lighter transaction, so more of them fit in 500,000 grams. A plain payment is already near that floor. A covenant, a many-input sweep, or a KIP-21 heavy script fits fewer transactions in the same block. A transaction over 500,000 grams does not fit until it is split into smaller ones.

Your own share, once the block is full, goes up only by taking mass that someone else would have used. Pay more than the transactions you need to displace, and keep enough confirmed outputs in flight to fill the mass you actually win. Submitting faster than that fills mempools.

Mining does the same job inside the blocks you find. The network still aims at 10 blocks a second. Extra hash changes who finds the block. A small miner does not move the ceiling.

## Workers, miners, and nodes

Three different things get called a worker.

**Node workers.** `block_processors_num_threads` and `virtual_processor_num_threads` default to 0, which means the logical CPU count. They validate in parallel. Once a node is already finishing blocks that are full of mass, more threads do not add transactions. They help a node that has fallen behind.

**Miner workers.** A CPU thread or a stratum connection. More of them raise the chance of finding the next block. Difficulty spends that hash to hold the block interval. The master file's Testnet-10 miners are CPU miners pointed at a node. They are not the mass budget.

**Application workers.** A wallet that spreads one submit across several sockets and nodes. That helps when one node is slow or its mempool rejects the transaction. A mempool acknowledgement means that node took it. It does not mean a block included it. More sockets do not widen the block.

**Protocol lanes** are a fourth count. v2.1.0 sets 50 lanes per block and a gas limit on each lane. A plain payment has no gas. The full blocks in the sample ran out of compute mass. They did not run out of lanes. Adding application workers does not add protocol lanes.

## Checked against the master file

The live row is about 10 blocks a second. About 100 blocks a second is not live. DAGKnight is Proposed. v2.1.0 is the node pin and is not a consensus activation. KIP-21 is Active. It prices script mass. The master file does not say that more workers or more miners raise the payment rate inside a full block. This note agrees with that.

## Checked against Maxim Biryukov's public repositories

This note is not a review request and not a summons.

KIP-21 and the mass code charge a heavy script in compute mass. Fewer of those scripts fit in the block. The payment ceiling does not rise.

The vprogs notes record a settlement built from many small inputs that passed 500,000 mass (about 516,000) and could not be mined until it was split. Same cap.

`rocksdb-versioned-bench` compares an index read with a seek, and sweeps bloom and ribbon filters. A faster disk read helps a node keep up. It does not change the mass cap.

`kaspa-xmss` carries signatures of about 3 KB and verifies them in the script engine. Those transactions are heavier. Fewer fit.

`dk-wiki` teaches DAGKnight. The live network stays on GHOSTDAG at about 10 blocks a second.

`kaspa-resolver` finds a node. The cpuminer tree mines. One is delivery. The other is a share of the blocks. Neither widens a block.

The rusty-kaspa fork and `workflow-perf-monitor-rs` are a node tree and a performance-stats library. Neither is a live consensus change.

## Who owns which lever

Michael Sutton leads protocol. Ori Newman shipped rusty-kaspa v2.1.0 and SilverScript v1.0.0. Yonatan Sompolinsky is the founder. Maxim Biryukov is core, and his public work here is KIP-21 and the tic-tac-toe guest. Hans Moog, Romain Billot (KCC and SilverScript), and coderofstuff are core. The board does not give coderofstuff a name. Fee research sits with FreshAir08. Aviv Zohar is a GHOST co-author. Shai Wyborski co-authored GHOSTDAG and left core in 2025. He is not a current core pin.

Parker Schmidt's 100-blocks-a-second material explains a rate that is not live. Luke Dunshea, Alexander Safstrom, and Sivan Helfer work on SilverScript and KCC. The master file does not call them core. Kaspa Unchained and Kaspa Global are community. supertypo maintains the DOTK indexer and has no X handle on the board. None of these roles is a switch that raises the rate of a full block.

## Limited hardware, and the public nodes

A Testnet-10 node is the same rusty-kaspa tree as main. There is no second consensus github for TN10. `main` is still v2.1.0, commit `01b532e8` (22 Sep 2026). The flag is `--testnet --netsuffix=10`. Ori Newman's merged pulls on that tag cover version bump `#1139`, arithmetic safety `#1138`, rejecting a coinbase in the mempool `#1137`, and IBD header chunks `#1136`. Maxim Biryukov's merged Toccata cleanup `#1082` through `#1089` is the mass and fee shape that is live: one block mass limit, a relay fee floor, script units. It does not add blocks per second. Hans Moog's merged `#958` fixes a protobuf default so a zero mass field is not rejected. Romain Billot's open DAGKnight issues `#1106` through `#1118` are the proposed network. Michael Sutton has said extremely high block rates and DAGKnight stay apart. coderofstuff's open mempool-notification issue `#339` tells a wallet when a transaction enters or leaves a pool. It does not make the pool larger.

On 7 Oct 2026, about 11:10Z, this desk tried 88 public Testnet-10 wRPC names. Every name that answered was kaspad 2.1.0, synced, with a UTXO index. Those answers were three machines. Several names, and some different addresses, showed one mempool. No fourth public node answered. Adding a hostname did not add a node.

The desk already runs one synced node. Its data is about 105 GB. The process was near 5 GB of RAM. Disk had room for another copy. A second node on this same PC would still relay into those same three public mempools. It would not add block mass, and it would not give a public node a larger pool. The panic line remains on a public node that was started with `--ram-scale=0.1`.

What this hardware can do is narrower. Keep one synced node on the default mempool scale. Submit only while the three public pools stay flat. Stop when they climb, before 100,000. Count machines. A list of names is not a list of mempools. A full block is still about 3,000 ordinary payments a second, and this desk does not raise that by opening more sockets.

Issue `#1134` on rusty-kaspa records IBD stalls and an RPC route that fills up, from a TN10 setup with many CPU miners. Those are ways a node falls behind. They are not a wider block. Issue `#1102` says the 100-blocks-a-second lore is not a KIP-2 activation. The master file already says the same thing. This reading agrees.

## The desk numbers, read against the master file

The desk record is [tn10-build-desk-tps](https://github.com/STP-KAS/tn10-build-desk-tps). Three figures sit next to each other, and they are not the same measurement.

6,321 tx/s is submit-OK for 20 seconds, six processes, fee 100 and 150, zero rejects. The seen-accepted count in that window is 38,822, about 1,900 a second, not 6,321 included. The 9,100 figure is 12 seconds with 28,617 orphans beside it. The long figure is 2,210 submit and 2,207 seen accepted, from 2026-10-06T23:53:27Z to 2026-10-07T05:54:53Z, fee 200 and 300. That is the number a later hold has to pass.

The master file's live row is still about 10 blocks a second, Toccata after DAA 474165565, KIP-16, 17, 20 and 21 Active, and rusty-kaspa v2.1.0 at `01b532e8`. That tag is not a consensus upgrade. GHOSTDAG is the live rule. The paper is Sompolinsky, Wyborski, and Zohar. Shai Wyborski left core in 2025 and is not a current pin. DAGKnight is Proposed. About 100 blocks a second is not live. Luke Dunshea, Alexander Safstrom, and Sivan Helfer are SilverScript and KCC on the board, not core.

`main` of rusty-kaspa has not moved past that tag. Ori Newman's merged `#1136`, `#1137`, `#1138`, and `#1139` are IBD chunks, a coinbase rejected from the mempool, arithmetic and relay checks, and the version bump. Maxim Biryukov's merged `#1082` through `#1089` is the Toccata mass cleanup. Hans Moog's `#958` lets a zero mass field through protobuf. Romain Billot's open `#1106` through `#1118` are DAGKnight work items. coderofstuff's `#339` is a mempool notification. None of those adds grams to a block or adds blocks per second. FreshAir08's fee work, on the board, decides who fills the grams that exist.

A reading at 13:57Z, after the public names had been checked and the block had gone quiet, put 2,800 coins at fee 200 and 300 through the desk node. For the next minute the virtual chain listed about 1,539 of them a second, with no rejects, and the public pools stayed near 23,000 to 26,000. The block then filled again, about 305 transactions and about 497,000 of 500,000 compute mass. A second pair at 1,018 and 1,527, 1,644 coins, took slots from the first pair. Combined seen-accepted fell to about 1,403. The priority quote then read 2,447, above that second fee. The pools sat near 36,000 to 38,000. The signers were stopped at 14:00:28Z. That is not a pass of 2,207, and it is not a pass of 6,321 included transactions.

At 14:09Z the priority quote had fallen to about 720 and the public pools were back near 18,000 to 22,000. One pair at fee 1,600 and 2,400, 2,800 coins, saw about 1,920 a second for about 70 seconds, with no rejects. The block was full again, about 307 transactions and about 497,700 of 500,000 compute mass. The priority quote then read 4,554, above that fee. The public pools were climbing through about 60,000. The pair was stopped at 14:11:26Z. Still under 2,207.

## What can be used later

A node patch returns mempool-full instead of asserting, and a public node under congestion stays off `--ram-scale=0.1`. A sender stops itself while public mempools are climbing toward 100,000. Neither change creates block mass.

A later reading the same morning put about 4,870 ordinary payments, at a fee above the quoted priority rate, through one desk node. For about 43 seconds the virtual chain listed about 2,222 of that sender's transactions per second, with no rejects in that window. That is higher than the earlier reading near 1,700, and still under the mass ceiling near 3,040. Public mempools climbed from the mid-40,000s through about 77,000. The extra senders were stopped. About half a minute later the same public pools were back in the mid-40,000s, and the three nodes still answered as synced. The stop kept them up. It did not hold the higher rate, and it did not fill the block.

Intentions are good; thought process is questionable. STP remains delusional. Si vis pacem, para bellum.
