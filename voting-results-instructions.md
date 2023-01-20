# On-Chain Voting Results for [MIPs](https://github.com/MinaProtocol/MIPs)

> [Granola Systems](https://granola.team/) is a software consultancy and Mina ecosystem partner. We make winning teams using our expertise in leadership, DevOps, Web3, distributed systems, functional programming, and data engineering. Granola has developed a dashboard for displaying on-chain [voting results](https://mina.vote/mainnet/mip1/results?start=1672848000000&end=1673685000000&hash=jxQXzUkst2L9Ma9g9YQ3kfpgB5v5Znr1vrYb1mupakc5y7T89H8). We operate an archive node and do regular [staking ledger dumps](https://github.com/Granola-Team/mina-ledger/tree/main/mainnet).

This article details the calculation of Mina's on-chain voting results. Equipped with the tools and knowledge, you can independently verify our code, logic, and results. Public blockchains like Mina Protocol are truly remarkable and equally as complex! We always strive for correctness in our code and exhaustively test.

All the code for this project is open source ❤️ and available on [GitHub](https://github.com/Granola-Team/mina-governance).

Find an issue or a bug? Have a question or suggestion? We'd love to get your [feedback](https://docs.google.com/forms/d/e/1FAIpQLSeKoyUIVU3OrJ7hkakwHnOeWz9R8gRe-pUeduXeMyfFsmW6iQ/viewform?pli=1)!

## Calculation of Mina's On-Chain Voting Results

We will describe in detail how to calculate the results of Mina's on-chain stake-weighted voting for `MIP1` during epoch 44

## Overview

At a high level, we will

<ol>
  <li>
    Obtain the <i>next staking ledger</i> of the <i>next</i> epoch
    <ul>
      <li>
        The <i>staking ledger</i> of epoch 46 contains the necessary info about delegations
      </li>
      <li>
        Available circa block 290 of the epoch
      </li>
    </ul>
  </li>
  <li>
    Calculate aggregated voter stake as a float (Rust <code>f64</code>)
    <ul>
      <li>
        Sum all delegations to each voting public key
      </li>
      <li>
        Voter stake weight is calculated with respect to the total <i>voting</i> stake
      </li>
    </ul>
  </li>
  <li>
    Obtain transaction data for the voting period, need <i>start</i> and <i>end</i> times</li>
  <li>
    Filter the voting (self) transactions, i.e. those with <code>source = receiver</code>
  </li>
  <li>
    Base58 decode the <code>memo</code> field of all votes
  </li>
  <li>
    Calculate yse/no weight
    <ul>
      <li>
        Sum yes/no vote stakes
      </li>
      <li>
        Divide by the total <i>voting stake</i>
      </li>
    </ul>
  </li>
</ol>

## How to use this document

There are several appropriate levels of engagement with this documentation. Readers are welcome to (from the highest level of technical knowledge/effort required to the least):

- Run a local archive node (the way Granola does it for the dashboard)
  - Query local PostgreSQL database for all relevant ledger and transaction data
- Export next staking ledger and hash from local mina daemon
  - Get transaction data via public GraphQL queries
- Download staking ledger from [Granola-Team/mina-ledger](https://github.com/Granola-Team/mina-ledger/tree/main/mainnet), [zkvalidator/mina-graphql-rs](https://github.com/zkvalidator/mina-graphql-rs/tree/main/data/epochs), or [Mina Explorer's Data Archive](https://docs.minaexplorer.com/minaexplorer/data-archive)
- Write an independent program in your favorite language implementing the calculation steps
- Just try a few queries
  - Try [Altair GraphQL browser extension](https://altairgraphql.dev/)
  - Try [Mina Explorer GraphiQL GUI](https://graphql.minaexplorer.com/)
- Generate a report using the [`mina_voting` utility](https://github.com/Isaac-DeFrain/mina-utils/tree/main/scripts/voting-results)
- Simply read along and learn

## Where to obtain the data

#### Staking ledgers

- Export from a local daemon
- [Granola-Team/mina-ledger](https://github.com/Granola-Team/mina-ledger/tree/main/mainnet)
- [zkvalidator/mina-graphql-rs](https://github.com/zkvalidator/mina-graphql-rs/tree/main/data/epochs)
- [Mina Explorer Data Archive](https://docs.minaexplorer.com/minaexplorer/data-archive)

#### Blocks and transactions

- Run your own archive node
  - [Docker](https://github.com/MinaProtocol/mina/tree/develop/dev)
  - [Build from source](https://github.com/MinaProtocol/mina/blob/develop/README-dev.md)
  - [Build from deb packages](https://docs.minaprotocol.com/node-operators/generating-a-keypair)
  - [Build from source with nix flakes](https://github.com/MinaProtocol/mina/blob/develop/nix/README.md)
- [Mina Explorer GraphiQL GUI](https://graphql.minaexplorer.com)

## Results Calculation Instructions

We calculate the results of [MIP1](https://github.com/MinaProtocol/MIPs/blob/main/MIPS/mip-remove-supercharged-rewards.md) voting

- Start: *January 4 16:00 UTC* (epoch 44, slot 2000)
- End: *January 14 08:30 UTC* (epoch 44, slot 6650)

| Data | Value |
|:-:|:-:|
| *Epoch* | `44` |
| *Keyword* | `MIP1` |
| *Start time* | `Jan 4, 2023 16:00 UTC` |
| *End time* | `Jan 14, 2023 08:30 UTC` |

## Calculation steps

### Obtain staking ledger

Since we are calculating the results for MIP1 voting (epoch 44), we need the *next staking ledger* of the *next* epoch, i.e. the *staking ledger* of epoch 46.

<ol type="a">
  <li>
    If you are not running a daemon locally, you will first need the ledger <i>hash</i>. Use the query

<pre><code>
query NextLedgerHash {
  blocks(query: {canonical: true, protocolState: {consensusState: {epoch: 45}}}, limit: 1) {
    protocolState {
      consensusState {
        nextEpochData {
          ledger {
            hash
          }
        }
        epoch
      }
    }
  }
}
</code></pre>

<pre><code>
response = {
  'data': {
    'blocks': [
      {
        'protocolState': {
          'consensusState': {
            'epoch': 45,
            'nextEpochData': {
              'ledger': {
                'hash': 'jxQXzUkst2L9Ma9g9YQ3kfpgB5v5Znr1vrYb1mupakc5y7T89H8'
              }
            }
          }
        }
      }
    ]
  }
}
</code></pre>

Extract the value corresponding to the deeply nested <code>hash</code> key

<pre><code>
response['data']['blocks'][0]['protocolState']['consensusState']['nextEpochData']['ledger']['hash']
</code></pre>

</li>
<li>
  Now that we have the appropriate ledger hash, we can acquire the corresponding staking ledger, in fact, the <i>next staking ledger</i> of epoch 45, via

<pre><code>
wget https://raw.githubusercontent.com/Granola-Team/mina-ledger/main/mainnet/jxQXzUkst2L9Ma9g9YQ3kfpgB5v5Znr1vrYb1mupakc5y7T89H8.json -O path/to/ledger.json
</code></pre>

You can use any of the following sources (extra credit: use them all and check diffs)
<ol type="i">
  <li>
    <a href="https://docs.minaexplorer.com/minaexplorer/data-archive">
      Mina Explorer’s data archive
    </a>
  </li>
  <li>
    <a href="https://github.com/zkvalidator/mina-graphql-rs/blob/main/data/epochs/">
      Zk Validator's mina-graphql-rs repo
    </a>
  </li>
  <li>
    <a href="https://github.com/Granola-Team/mina-ledger/tree/main/mainnet">
      Granola’s mina-ledger repo
    </a>
  </li>
<li>
  If you’re running a local daemon, you can export the next staking ledger (while we are in epoch 45) by

<pre><code>
mina ledger export next-staking-ledger > path/to/ledger.json
</code></pre>

and confirm the hash using

<pre><code>
mina ledger hash --ledger-file path/to/ledger.json
</code></pre>

This calculation may take several minutes!

</li></ol></ol>

### Aggregate voting stake

<ol type="a">
  <li>
    Calculate each voter's <i>stake</i> from the staking ledger. Aggregate all delegations to each voter (by default, an account is delegated to itself)

<pre><code>
agg_stake = {}
delegators = set()

for account in ledger:
    pk = account['pk']
    dg = account['delegate']
    bal = float(account['balance'])

    # pk delegates
    if pk != dg:
        delegators.add(pk)

    try:
        agg_stake[dg] += bal
    except:
        agg_stake[dg] = bal
</code></pre>

  </li>
  <li>
    Drop delegator votes

<pre><code>
for d in delegators:
    try:
        del agg_stake[d]
    except:
        pass
</code></pre>

  </li>
  <li>
    Now <code>agg_stake</code> is a Python dict containing each voter's aggregated stake
  </li>
</ol>

### Obtain and parse votes

To obtain all MIP1 votes, we need to get all transactions corresponding to the MIP1 voting period (votes are just special transactions after all). It would be nice to be able to prefilter the transactions more and only fetch what is required, but since `memo` fields are *base58 encoded* and *any capitalization of the keyword is valid*, prefiltering will be complex and error-prune.

<ol type="a">
  <li>
    Multiple data sources
    <ol type="i">
      <li>
        Run a local archive node
      </li>
      <li>
        Mina Explorer has
        <a href="https://docs.minaexplorer.com/minaexplorer/bigquery-public-dataset">
          BigQuery
        </a>,
        <a href="https://docs.minaexplorer.com/minaexplorer/graphql-getting-started">
          GraphQL
        </a>, and
        <a href="https://docs.minaexplorer.com/rest-api/ref">
          REST
        </a>
        APIs
      </li>
    </ol>
  </li>
  <li>
    Obtain the unique <i>voting transactions</i>
    <ol type="i">
      <li>
        A <i>vote</i> is a transaction satisfying:
        <ol>
          <li>
            <code>kind = PAYMENT</code>
          </li>
          <li>
            <code>source = receiver</code>
          </li>
          <li>
            <i>Valid</i> <code>memo</code> field (either <code>mip1</code> or <code>no mip1</code>)
          </li>
        </ol>
      </li>
      <li>
        Fetch all transactions for the voting period
        <ol>
          <li>
            To avoid our requests getting too big and potentially timing out, we will request the transactions from each block individually
          </li>
          <li>
            Block production varies over time; sometimes many blocks are produced in a slot, sometimes no blocks are produced. A priori, we do not know the exact block heights which constitute the voting period. We fetch all <i>canonical</i> block heights for the voting period, determined by the <i>start</i> and <i>end</i> times
<pre><code>
query BlockHeightsInVotingPeriod {
  blocks(query: {canonical: true, dateTime_gte: "2023-01-04T16:00:00Z", dateTime_lte: "2023-01-14T08:30:00Z"}, limit: 7140) {
    blockHeight
  }
}
</code></pre>

The max number of slots, hence blocks, in an epoch is `7140`. The response in includes block heights `213195` to `216095`

<pre><code>
{
  "data": {
    "blocks": [
      {
        "blockHeight": 216095
      },
      ...
      {
        "blockHeight": 213195
      }
    ]
  }
}
</code></pre>
</li>
<li>
  For each canonical block height in the voting period, query the block’s <i>PAYMENT</i> transactions (votes are payments)

<pre><code>
query TransactionsInBlockWithHeight($blockHeight: Int!) {
  transactions(query: {blockHeight: $blockHeight, canonical: true, kind: "PAYMENT"}, sortBy: DATETIME_DESC, limit: 1000) {
    blockHeight
    memo
    nonce
    receiver {
      publicKey
    }
    source {
      publicKey
    }
  }
}
</code></pre>

where `$blockHeight` is substituted with each of the voting period’s canonical block heights (automation is highly recommended). Again, we include a limit which far exceeds the number of transactions in any block so we don’t accidentally get short-changed by a default limit. This process will take several minutes if done sequentially. Performance improvements are left as an exercise to the reader.

For example, the response for block `216063`

<pre><code>
{
  "data": {
    "transactions": [
      {
        "blockHeight": 216063,
        "memo": "E4Yxu8shUhP1SMV5fUoGZb4sqEPREUCLErpYVJMQD1pY5iuocbibr",
        "nonce": 41977,
        "receiver": {
          "publicKey": "B62qqJ1AqK3YQmEEALdJeMw49438Sh6zuQ5cNWUYfCgRsPkduFE2uLU"
        },
        "source": {
          "publicKey": "B62qnXy1f75qq8c6HS2Am88Gk6UyvTHK3iSYh4Hb3nD6DS2eS6wZ4or"
        }
      },
      ...
            {
        "blockHeight": 216063,
        "memo": "E4YM2vTHhWEg66xpj52JErHUBU4pZ1yageL4TVDDpTTSsv8mK6YaH",
        "nonce": 3,
        "receiver": {
          "publicKey": "B62qjt1rDfVjGX6opVnLpshRigH5U6UFjyMNYdWCjo99im2v7VrzqF6"
        },
        "source": {
          "publicKey": "B62qpZMGRKrse3mVxf3SNMfzWh5c4TM2PfvgL98oVadwNWb6S1tJ8Te"
        }
      }
    ]
  }
}
</code></pre>

Notice the base58 encoded `memo` field

</li>
              <li>
                Concatenate transactions for all canonical blocks in the voting period
              </li>
            </ol>
          </li>
          <li>
            Filter the votes
            <ol>
              <li>
                <code>memo.lower()</code> <i>exactly equal</i> to <code>mip1</code> or <code>no mip1</code>
              </li>
              <li>
                <code>source = receiver</code> (self transaction)
              </li>
            </ol>
          </li>
            <li>
              The memo field is base58 encoded
            </li>
          <li>
            If there are multiple votes associated with a single public key, only the <i>latest</i> vote is counted; <i>latest</i> being defined:
          </li>
          <ol>
            <li>
              For multiple votes from the same account across several blocks, take the vote in the highest block.
            </li>
            <li>
              For multiple votes from the same account in the same block, take the vote with the highest nonce.
            </li>
          </ol>
        </ol>
      </li>
    </ol>
  </li>
</ol>

### Calculate weighted voting results

<ol type="a">
  <li>
    Sum all aggregated voter stake to get the <i>total voting stake</i>
  </li>
  <li>
    Sum stake for yes/no votes
  </li>
  <li>
    Divide yes/no vote stakes by the total voting stake, as a float in Python, f64 in Rust
  </li>
</ol>

Check agreement with the <a href="https://mina.vote/mainnet/mip1/results?start=1672848000000&end=1673685000000&hash=jxQXzUkst2L9Ma9g9YQ3kfpgB5v5Znr1vrYb1mupakc5y7T89H8">voting results dashboard</a> and/or `mina_voting` utility

</ol>
</li>
</ol>

## MIP1 Voting Results

[Granola's MIP1 results dashboard](https://mina.vote/mainnet/mip1/results?start=1672848000000&end=1673685000000&hash=jxQXzUkst2L9Ma9g9YQ3kfpgB5v5Znr1vrYb1mupakc5y7T89H8)

[`mina_voting` MIP1 report](https://github.com/Isaac-DeFrain/mina-utils/blob/main/scripts/voting-results/mip1_report.txt)

```sh
# summary from mina_voting MIP1 report

Keyword: mip1
Outcome: YES

Yes vote stake:  224148426.55565
No vote stake:   3465078.8547372

Yes vote weight: 0.98477647954813
No vote weight:  0.015223520451871

Vote stake:      227613505.41039
Total stake:     994001117.83985
Turnout:         22.90%

Num epoch txns:  56838
Num "vote" txns: 251
Num delegated:   181
Num yes votes:   66
Num no votes:    4
```

## Conclusion

MIPs are a substantial addition to Mina's ever-evolving suite of on-chain governance functionalities. This is a huge milestone which advances the community and decentralization of the protocol. Granola is incredibly honored to be a part of the technical implementation of such a major development in the Mina ecosystem.
