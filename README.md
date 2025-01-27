# Batcher Cypherapp

Created by Kexkey and Francis from bullbitcoin.com

## batcher cypherapp API intro

Batcher is Bitcoin transaction batching and automation plugin for cyphernode users (a cypherapp) designed for high-volume enterprise users performing multiple Bitcoin transactions per day. Instead of making one Bitcoin transaction for each Bitcoin payment (e.g. a user withdrawing Bitcoin from an exchange) it allows you to schedule and manage batches of multiple Bitcoins payments sent to the Bitcoin Blockchain as a single transaction.

Cypherapps can be conceived as cyphernode "plugins". Running a Cyphernode instance is thus required to use the Batcher cypherapp. Instead of communicating directly with the Cyphernode API, users will connect to the Batcher API. Batcher will then manage how and when Cyphernode will be creating Bitcoin transactions.

Batcher is currently implemented in the Bull Bitcoin exchange

### Benefits

- Significantly lower transaction fee expenses
- Fewer change outputs
- Smaller chains of unconfirned ancestor UTXOs

All of this makes you spend up to 80% less on transaction fees overall. The hot wallets have fewer errors. It makes your on-chain non-custodial Bitcoin payout solutions seemlessly automated for optimal results according to your configs.

### Downsides

- Payments are not instant. You should notify the end-users when they can expect the transaction to be done.
- UTXO clusters easier to detect with chainalysis software or human trackers. Users that are in the bitcoin transaction will know that the other addresses of that transaction are highly likely to be users of the same service.

## Concept and workflow

### Step 1: Creating a batching schedule

Create a batching schedule via the configuration file. You can opt for :

- Amount-based batch threshold (e.g. everytime the batch reaches at least 0.5 Bitcoin)
- Time-based batch schedule (e.g. execute the current batches ever 4 hours).
- We recommend using both. For example, execute the batch every time the amount exceeds 1 Bitcoin or every hour, whichever comes first.

Edit the config here <https://github.com/SatoshiPortal/batcher/blob/master/src/config/BatcherConfig.ts>

- `BATCH_TIMEOUT_MINUTES`: set this as the maximum frequency. If the threshold amount is not reached it will execute regardless at this frequency.

- `CHECK_THRESHOLD_MINUTES`: frequency of checking the threshold.

- `BATCH_THRESHOLD_AMOUNT`: the target batch threshold. When this amount is reached, the batch will be executed as a Bitcoin transaction. If it is not reached, the batch will be executed according to the batch timeout setting.

- `BATCH_CONF_TARGET`: when the batch is executed, this setting will determine which network fee level the Bitcoin Core wallet will use for the payments. You can for example have 2 batches, one with  batch_conf_target of 6 for express withdrawals and one of batch_conf_target of 100 for non-urgent transactions. You can override this when you call `executeBatch`

**Sample configs**

This is Batcher 1. Batches every 60 minutes or whenever the batch reaches 1 BTC (whichever is soonest). Check every minute. The conf target is 6 blocks. 

```
{
    "LOG": "DEBUG",
    "BASE_DIR": "/batcher",
    "DATA_DIR": "data",
    "DB_NAME": "batcher.sqlite",
    "URL_SERVER": "http://batcher",
    "URL_PORT": 8000,
    "URL_CTX_WEBHOOKS": "webhooks",
    "SESSION_TIMEOUT": 600,
    "CN_URL": "https://gatekeeper:2009/v0",
    "CN_API_ID": "003",
    "CN_API_KEY": "39b83c35972aeb81a242bfe189dc0a22da5ac6cbb64072b492f2d46519a97618",
    "DEFAULT_BATCHER_ID": 1,
    "BATCH_TIMEOUT_MINUTES": 60,
    "CHECK_THRESHOLD_MINUTES": 1,
    "BATCH_THRESHOLD_AMOUNT": 1,
    "BATCH_CONF_TARGET": 6
}
```


### API Workflow

- `Add to batch`: submit a Bitcoin address and amount of a payment to a batching queue via API.
- For each payment (output + amount) you should add a callback URL that will receive the webhook notification when the transaction is sent (0-conf). This is useful for notifying users that their withdrawal has been processed. You will receive detailed transaction info.
- You can remove a payment from a batch at any time, for example if the user wants to have an instant withdrawal. We would suggest to then send the Bitcoin using the normal `sendtoaddress` API call. To make the end-user pay for the transaction fee instead of you (for example as a premium for opting out of transaction) you can subtract the fee from the amount.
- Specify which batching schedule you want that payment to be queued in when submitting Bitcoin payments to the Batcher API. You may want to have different batching schedules, some more frequent than others, and some with lower confirmation targets (lower fees) than others.

#### Adding a Bitcoin payment to a batch via API

```curl -d '{"id":1,"method":"queueForNextBatch","params":{"address":"bcrt1q0jrfsg98jakmuz0xc0mmxp2ewmqpz0tuh273fy","amount":0.0001,"webhookUrl":"http://webhookserver:1111/indiv"}}' -H "Content-Type: application/json"  -k -u "<username>:<dd xxd output>" https://localhost/batcher/api | jq```

#### API response after adding a payment to a batch

```TypeScript
{
  result?: {
    batchRequestId: number;
    batchId: number;
    etaSeconds: number;
    cnResult: {
      batcherId?: number;
      batcherLabel?: string;
      outputId?: number;
      nbOutputs?: number;
      oldest?: Date;
      total?: number;
    }
  }
  error?: {
    code: number;
    message: string;
    data?: D;
  }
}
```

- In the API response above, you get the time of the next batch etaSeconds. You can notify the user that the batch will be executed at the latest at that time (and possibly earlier).
- If multiple payments are being made to a single Bitcoin address, batcher will aggregate the amounts and make a single payment to that Bitcoin address. This is not optional because Bitcoin Core would otherwise reject the transaction.
- Once the amount or expiry time thresholds have been reached, Batcher will dispatch a request to the Bitcoin Core instance running in cyphernode to create and broadcast the transaction using the `sendmany` Bitcoin Core RPC call.
- In the API webhook notification, you will receive the information related to each Bitcoin payment as if it had been its own transaction (see below).

#### Webhook notification sent to all the callback URLs submitted with payments to a batch

```TypeScript
{
  "error": null,
  "result": {
    "batchRequestId": 48,
    "batchId": 8,
    "cnBatcherId": 1,
    "txid": "fc02518e32c22574158b96a513be92739ecb02d0caa463bb273e28d2efead8be",
    "hash": "fc02518e32c22574158b96a513be92739ecb02d0caa463bb273e28d2efead8be",
    "spentDetails": {
      "address": "2N8DcqzfkYi8CkYzvNNS5amoq3SbAcQNXKp",
      "amount": 0.0001,
      "firstseen": 1584568841,
      "size": 222,
      "vsize": 141,
      "replaceable": false,
      "fee": 0.00000141,
      "subtractfeefromamount": false
    }
  }
}
```

#### Remove a payment from a batch using the api call below

This is useful for example of one of your users opted for the batch payment option and changes his mind after.

```Request:
dequeueFromNextBatch
{
    batchRequestId: number;
}
```

#### The information you get on a batch is

```TypeScript
getBatchDetails
Request:
{
  batchRequestId?: number;
  batchId?: number;
}
```

#### Info on batch as API response

```TypeScript
{
  result?: {
    batchId: number;
    cnBatcherId: number;
    txid?: string;
    spentDetails?: string;
    spentTimestamp?: Date;
    createdAt?: Date;
    updatedAt?: Date;
    batchRequests: [
      {
        batchRequestId: number;
        externalId?: number;
        description?: string;
        address: string;
        amount: number;
        cnBatcherId?: number;
        cnBatcherLabel?: string;
        webhookUrl?: string;
        calledback?: boolean;
        calledbackTimestamp?: Date;
        cnOutputId?: number;
        mergedOutput?: boolean;
        createdAt?: Date;
        updatedAt?: Date;
      },
    ]
  }
  error?: {
    code: number;
    message: string;
    data?: D;
  }
}
```

See the [technical doc and installation notes](doc/INSTALL.md).

See the [technical architecture document](doc/CONTRIBUTING.md).

## Discussion: an on-chain solutions for Bitcoin scaling

The Bitcoin network's transaction throughput is limited to ~ 2500 transactions per block because of the maximum block weight limit of about 4MB. The supply of block space is thus limited to 576MB per day, which can fit ~360,000 transactions.

This limit keeps Bitcoin decentralized and preserves its core features: scarcity, censorship-resistance, auditability and sovereignty.
Because of the growing demand for block space, and the fact that blocks are produced every 10 minutes, users that wish to have their transactions included have two options:

- Pay a higher transaction fee to get priority (faster confirmation)
- Wait a longer amount of time to get your transaction confirmed

There are generally four things we can do to alleviate this problem:

1. **Transactions off-chain using second-layer networks (Lightning, Liquid)**: these technologies are great, but they are still experimental and have their own issues. We believe they are the right long-term approach for Bitcoin scaling, but will take time before they are mature enough for wider network effects to take hold.

2. **Transactions off-chain using custodial platforms** this is completely against the point of Bitcoin. This is a lazy, bad solution.

3. **Optimize the transactions to make their size smaller**: this is what Bitcoin Core is working on, on many levels, Segwit was a big step in the right direction. More technologies will come over the next years.

👉 4. **Batch multiple Bitcoin payments are a single transaction**: this is the lowest hanging fruit. It is an obvious and intuitive solution (e.g. fit more people in a bus on the highway instead of adding more lanes). However, it requires a lot of hard work to build software that allows this. Most exchange operators are lazy or simply don't care about the Bitcoin network outside of their immediate short-term interests.

There was a market niche for a solution that automates all aspects of transaction batching, which is self-hosted and free, connected to Bitcoin Core (via Cyphernode). So we built it!

Before moving on to 2nd-layer networks, we think other exchanges should first optimize as much as possible their transaction batching. Batcher is free, open-source and self-hosted (your own keys, your own node). So just use it!

## Why Bitcoin transaction batching is helpful

The concept of entreprise Bitcoin transaction batching is simple: instead of doing a single transaction each time you sent a payment to a recipient, you queue these payments and aggregate them as different outputs (recipients) of a single Bitcoin transaction. There are three main benefits

### Do the math: save up to 80%

- You do 100 transactions per hour.
- 2400 transactions per day!
- Each of them costs you 50 sat/byte.
- Your average transaction size per payment is 250 bytes.
- You're adding at minimum 600,000 bytes to the blockchain every day!
- Paying 30,000,000 sats per day! (0.3 BTC)
- But instead, now you batch them every hour.
- Now you do 24 transactions per day
- Each with 100 outputs. Size of 4000 bytes.
- You only add 100,000 bytes to the blockchain.
- It costs you only 5,000,000 sats per day. (0.05 BTC)
- And you save 0.5MB of block space for everyone else, every day!

**No brainer!**

### Minimize your change UTXOs

Same math as above. You go from adding 2400 change utxos per day to 24 change utxos per day.
This makes the hot wallet management so much easier and less prone to errors.

### Minimize ancestor count

This is a very niche problem that only high-volume Bitcoin services operators will understand. Basically, if you do a lot of transactions and the previous inputs don't get confirmed, and you go over 25 unconfirmed transactions in a chain of transactions, other nodes will consider it as invalid and it won't be included in a block. Batcher helps significantly here, because there are far fewer utxos created.
