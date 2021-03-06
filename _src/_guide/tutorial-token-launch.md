---
layout: guide

title: "Tutorial: How to launch your own token on BigchainDB"
tagline: Learn how to use divisible assets in BigchainDB for token generating events
header: header-token.jpg
order: 3

learn: >
    - How to use divisible assets on BigchainDB

    - How assets in BigchainDB can represent tokens

    - How tokens can be distributed to participants using TRANSFER transactions
    
    - How various BigchainDB transactions can be combined together
---

Hi there! Welcome to our next tutorial about divisible assets. For this tutorial, we assume that you are familiar with the BigchainDB primitives (assets, inputs, outputs, transactions etc.). If you are not, familiarize yourself with the [Key concepts of BigchainDB](../key-concepts-of-bigchaindb/). We also assume that you have completed our [first tutorial](../tutorial-car-telemetry-app/).

# About token generating events

In the last 12 months we have witnessed exponential growth in token generating events. Many of them have been launched on Ethereum. Since we are experiencing rising interest in potential token launches on BigchainDB, this tutorial aims at showing a very simple approach on how to launch your own token on BigchainDB.

This tutorial just aims at illustrating the usage of one building block, namely divisible assets. An actual token launch is much more complex and requires other components which are not discussed here. Furthermore, BigchainDB is currently not yet ERC20 compatible.

{% include_relative _setup.md %}

# Usage of divisible assets to create tokens

In BigchainDB, a token generation can be represented as a divisible asset. A divisible asset is an asset that has a fixed number of sub-assets linked to it. This means that the create transaction contains more than that asset. The linked fixed sub-assets represent your tokens. When creating a divisible asset in BigchainDB, the number of the sub-assets (tokens) that you want to create needs to be specified in the beginning. That number represents your fixed total supply of tokens.

The code below illustrates how to create a divisible asset with 10 000 tokens associated to it.

```js
const nTokens = 10000
let tokensLeft
const tokenCreator = new BigchainDB
.Ed25519Keypair(bip39.mnemonicToSeed('seedPhrase').slice(0,32))
let createTxId
function tokenLaunch() {
    // Construct a transaction payload
    const tx = BigchainDB.Transaction.makeCreateTransaction({
            token: 'TT (Tutorial Tokens)',
            number_tokens: nTokens
        },
        // Metadata field, contains information about the transaction itself
        // (can be `null` if not needed)
        {
            datetime: new Date().toString()
        },
        // Output: Divisible asset, include nTokens as parameter
        [BigchainDB.Transaction.makeOutput(BigchainDB.Transaction
          .makeEd25519Condition(tokenCreator.publicKey), nTokens.toString())],
        tokenCreator.publicKey
    )

    // Sign the transaction with the private key of the token creator
    const txSigned = BigchainDB.Transaction
      .signTransaction(tx, tokenCreator.privateKey)

    // Send the transaction off to BigchainDB
    conn.postTransactionCommit(txSigned)
        .then(res => {
            createTxId = res.id
            tokensLeft = nTokens
            document.body.innerHTML ='<h3>Transaction created</h3>';
            // txSigned.id corresponds to the asset id of the tokens
            document.body.innerHTML +=txSigned.id
        })
}
```

With these commands, you have minted 10000 tokens. For that, give an extra parameter to the `makeOutput()` function. Pay attention to give the function a string instead of a plain number. With the `tokenCreator` keypair you indicate who will be the owner of the tokens. This could for instance be the foundation issuing the tokens. Once this transaction is accepted by BigchainDB, you update the value of the tokens left in the possession of the creator. Right now, all the tokens created are associated with the public key of the creator (`tokenCreater.publicKey`).

Now that the tokens have been minted, you can start distributing them to the owners.

# Distribution of tokens

Tokens can be transferred to an unlimited number of participants. In this example, you are now going to make a transfer transaction to transfer 200 tokens to a new user called John. For that, you first need to create a new user and then do the transfer. The code below shows that.

```js
const amountToSend = 200

const newUser = new BigchainDB
    .Ed25519Keypair(bip39.mnemonicToSeed('newUserseedPhrase')
        .slice(0, 32))

function transferTokens() {
    // User who will receive the 200 tokens
    const newUser = new BigchainDB.Ed25519Keypair()

    // Search outputs of the transactions belonging the token creator
    // False argument to retrieve unspent outputs
    conn.getTransaction(createTxId)
        .then((txOutputs) => {
            // Create transfer transaction
            const createTranfer = BigchainDB.Transaction
                .makeTransferTransaction(
                    [{
                        tx: txOutputs,
                        output_index: 0
                    }],
                    // Transaction output: Two outputs, because the whole input
                    // must be spent
                    [BigchainDB.Transaction.makeOutput(
                            BigchainDB.Transaction
                            .makeEd25519Condition(tokenCreator.publicKey),
                            (tokensLeft - amountToSend).toString()),
                        BigchainDB.Transaction.makeOutput(
                            BigchainDB.Transaction
                            .makeEd25519Condition(newUser.publicKey),
                            amountToSend)
                    ],
                    // Metadata (optional)
                    {
                        transfer_to: 'john',
                        tokens_left: tokensLeft
                    }
                )

            // Sign the transaction with the tokenCreator key
            const signedTransfer = BigchainDB.Transaction
                .signTransaction(createTranfer, tokenCreator.privateKey)

            return conn.postTransactionCommit(signedTransfer)
        })
        .then(res => {
            // Update tokensLeft
            tokensLeft -= amountToSend
            document.body.innerHTML += '<h3>Transfer transaction created</h3>'
            document.body.innerHTML += res.id
        })

}
```
You have now transferred 200 tokens to the user John. You could repeat the same with multiple other users.
With `listOutputs` using `false` as the second argument you retrieved all the outputs belonging to the user `tokenCreator`, that were not spent yet. There will just be one output that fulfills these characteristics, because when you transfer tokens to another user, you are spending this output and giving the ownership to the other user. Then, you queried for that transaction and made a transfer to John with it. Note however, that there is also a transaction back to `tokenCreator.publicKey`, as you need to 'give back change' due to BigchainDB's transaction model. It is designed in a way that all of the inputs have to be spent in a transaction. That means that if you send part of the `tokensLeft` (200 tokens) to John, you have to send the rest (9800 tokens) back to the `tokenCreator` to preserve that amount.

# Combination of different BigchainDB transactions

Imagine now that you have received several transactions of tokens and you want to combine all of the balances and transfer them to another user (e.g. your best friend, who keeps some tokens in escrow). In BigchainDB, this is possible as well. The code below illustrates how to do that.

```js
const amountToSend = 200
const bestFriend = new driver.Ed25519Keypair()

function combineTokens(transaction1, outputIndex1, transaction2, outputIndex2,
    totalTokens) {
    const combineTranfer = BigchainDB.Transaction.makeTransferTransaction(
        [{
            tx: transaction1,
            output_index: outputIndex1
        }, {
            tx: transaction2,
            output_index: outputIndex2
        }],
        // Output
        [BigchainDB.Transaction.makeOutput(
            BigchainDB.Transaction.makeEd25519Condition(
                bestFriend.publicKey),
            (totalTokens).toString())], {
            transfer_to: 'my best friend'
        }
    )

    // Sign the transaction with the newUser key
    const signedTransfer = BigchainDB.Transaction
        .signTransaction(combineTranfer, newUser.privateKey)

    return conn.postTransactionCommit(signedTransfer)

}
```

You just made a transfer transaction combining two different transactions into one output. Note that the `totalTokens` quantity is a required variable. It is the sum of the tokens of the two outputs being spent. As you have seen before, if this quantity is not correct, the transaction will fail, as you literally need to spend all of the outputs in a transaction.
`transaction1` and `transaction2` can look like the transaction `createTranfer` that you did before. In that case, the `outputIndex1` and `outputIndex2` would be `0`.

Note that in our example, the supply of your tokens was fixed and cannot be changed anymore after creation. So, you would need to clearly define for yourself, how many tokens you will need. However, BigchainDB does offer the option of refillable, divisible assets that allow for a more dynamic token supply. You can learn more about that [here](https://github.com/bigchaindb/bigchaindb/issues/1741).

That's it! Now you know, how divisible assets in BigchainDB can be used as a potential building block for token launches. Of course, in practice a token distribution event is much more complex and requires other important building blocks like smart contracts etc. But, this tutorial showed you how divisible assets can play a part of that.  
