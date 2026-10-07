> **Experimental. We are just trying this.**
>
> Good intentions, shaky hands. STP does not know what he is doing. We test, we write down what we think we saw, and that is the whole product. A number here is not the truth. A chart is not the truth. Any other sentence that sounds sure of itself is not the truth either. Do not count any of it as a claim.
>
> [Disclaimer](DISCLAIMER.md)

# What limits the transaction rate

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

## What can be used later

A node patch returns mempool-full instead of asserting, and a public node under congestion stays off `--ram-scale=0.1`. A sender stops itself while public mempools are climbing toward 100,000. Neither change creates block mass. The open question is whether enough ordinary payments, priced above the rest of a full block, can take the mass that is already there without crossing that line.

Intentions are good; thought process is questionable. STP remains delusional. Si vis pacem, para bellum.
