# Step by Step guide for a 1:1 SegWit2x and SegWit1x with minimal trust

This is a guide to exchange SegWit1X and SegWit2x 1:1 without 3rd party and with minimal trust.

Let Alice and Bob agree to an amount X and that 

* Alice gives Bob X coins that are valid on the chain of the current ruleset
* Bob gives Alice X coins that are valid on the chain of the SegWit2X ruleset

They also agree to a small "leverage" amount for the deal Y, for instance 10%.

They should also agree to terms defining the deal that are out of scope of this swap. For example:

* Any chain with a different ruleset, including new rules published by the Core implementation are out-of-scope
* If the SegWit2X is canceled and no SegWit2X blocks are mined, the deal is canceled.

## Concept

We trade with the only required trust being the trust not to "hostage" both coins. This is done in 3 steps, pre-fork:

1. Alice and Bob create and publish a 2-2 multisig transaction, with a single output and which spends X+Y from Alice and X+Y from Bob.
2. Alice and Bob create and sign a transaction that spends the output of (1), spending 2X+Y to Bob and Y to Alice, which includes a locktime 494785 and the S2X replay-protection address. This is safely held by Bob.
3. Alice and Bob create and sign a transaction that spends the output of (1), spending 2X+Y to Alice and Y to Bob, with a locktime of 495085, and no replay protection. This is safely held by Alice.

Directly after the fork, Bob can publish (2) to claim his coins, on the legacy chain and two days later Alice can publish (3) to claim her coins on the S2X chain.

## Risks

Some trust is still involved and must be noted. We are **assuming**:

* Neither Alice nor Bob stops after 1 to hold both their coins hostage
* Bob doesn't stop after 2, to be able to claim his Legacy coins, while holding both their SegWit2x and leverage hostage.
* If Bob's transaction (2) takes more than 2 days to confirm, Alice still waits before publishing (3).
* Both uphold additional terms. Most importantly, upholding a "no S2X, no deal" term, requires Alice to trust Bob in refunding.

## Details using bitcoind

Let X be the amount agreed upon, Y be a leverage amount (eg 10% of X), and F be a high fee amount (eg 0.001)

Both Alice and Bob must have ready a transaction output with *exactly* X+Y+F. (**A_TXID**, **A_VOUT**, **B_TXID**, **B_VOUT**). (This can be setup this using sendtoaddres, then listunspent)

### Step 1: The multisig transaction

**NOTE: All the hex-strings are samples and should not be the same as presented here (unless you want to tip me:)**

**NOTE: Don't just copy-paste the commands if you don't have a clue what they mean. Please ask if needed.**


First Alice creates a keypair **A_PUB**, **A_PRIV**:

    A> bitcoin-cli getnewaddress
    149bZBW9SsUstbVrmsh6XqrveUTAvEKSqN
    
    A> bitcoin-cli validateaddress  149bZBW9SsUstbVrmsh6XqrveUTAvEKSqN
    {
      ...
      "pubkey": "02f46422061b2f468355562f6d9048b893d1a5844843473f91eed7089d5689e7a5",  # A_PUB
      ...
    }
    
    A> bitcoin-cli dumpprivkey  149bZBW9SsUstbVrmsh6XqrveUTAvEKSqN
    L5QVxYoh3dx6qRF6XnKT7CHMwuLXd73XTDQHPjPZXVq4ZfjzJy5p # A_PRIV


Bob creates a keypair **B_PUB** and **B_PRIV**:

    B> bitcoin-cli getnewaddress
    1FomUjGBayq4EzUKbZFq7dhYsXxZpA65Pz
    
    B> bitcoin-cli validateaddress  1FomUjGBayq4EzUKbZFq7dhYsXxZpA65Pz
    {
      ...
      "pubkey": "03f60816185299b3723ee233d1b4ba75ca4f4d7fcda5f321de672174fc284046f2",  # B_PUB
      ...
    }
    
    B> bitcoin-cli dumpprivkey  1FomUjGBayq4EzUKbZFq7dhYsXxZpA65Pz
    KxBdCeUSf8Bdx8Qis4VwuVx8PvKgqo1yiiumsEmpqoR2CUqaJPtW # B_PRIV


Bob now sends **B_PUB**, **B_TXID** and **B_VOUT** to Alice. Then Alice creates an address for the 2-2 multisig output using **A_PUB** and **B_PUB**


    A> bitcoin-cli createmultisig 2  '["02f46422061b2f468355562f6d9048b893d1a5844843473f91eed7089d5689e7a5","03f60816185299b3723ee233d1b4ba75ca4f4d7fcda5f321de672174fc284046f2"]'
    {
      "address": "3NQyLyB2oZMsg9cDm6wJDGunJM8aBz4aiW",
      "redeemScript": "522102f4...6f252ae"
    }

Alice then creates and signs the raw transaction, using the address just created, **A_TXID**, **A_VOUT**, **B_TXID**, **B_VOUT**, and the amount 2X+2Y+F 

    A> bitcoin-cli createrawtransaction '[{"txid":"$A_TXID$", "vout":$A_VOUT$}, {"txid":"$B_TXID$", "vout":$B_VOUT$}]' '{"3NQyLyB2oZMsg9cDm6wJDGunJM8aBz4aiW", "amount":$2X+2Y+F}'
    020000...00000
    
    A> bitcoin-cli signrawtransaction 020000...00000
    {
      "hex": "020000...00000",
      "complete": false
    }

Alice then sends this resulting raw hex transaction and her **A_PUB**, **A_TXID** and **A_VOUT** to Bob. 

Bob then **verifies** the multisig address,  the transaction , and signs and sends it.

    B> bitcoin-cli createmultisig 2  '["02f46422061b2f468355562f6d9048b893d1a5844843473f91eed7089d5689e7a5","03f60816185299b3723ee233d1b4ba75ca4f4d7fcda5f321de672174fc284046f2"]'
    {
      "address": "3NQyLyB2oZMsg9cDm6wJDGunJM8aBz4aiW",
      "redeemScript": "52210...252ae"
    }
    
    B>  bitcoin-cli decoderawtransaction 02000...00000
    { ... }
    
    B> # verifies above results
    
    B> bitcoin-cli signrawtransaction 020000...0000
    {
      "hex": "02000...0000",
      "complete": true
    }
    
    # send the resulting hash
    B> bitcoin-cli sendrawtransaction 0200000.....0000

Step 1 is complete if the multisig is confirmed. The **MULTISIG_TXID** is then used in the following steps

### Step 2: The transaction paying Bob

Alice now creates a transaction that spends **MULTISIG_TXID** to Bob and is *only* valid on the legacy chain, and only  after the fork. For this she will need a new address of herself **A_ADR** and of Bob **B_ADR**. 

She will also need the redeemscript from `createmultisig` above, and the `scriptPubKey` from `decoderawtransaction` of the final hex. She can then pay 2X+Y to Bob, and Y to herself

    A> bitcoin-cli createrawtransaction '[{"txid":"$MULTISIG_TXID$","vout":0,"scriptPubKey":"...","redeemScript":"..."}]' '{"$A_ADR$":Y, "$B_ADR":2X+Y, "3Bit1xA4apyzgmFNT2k8Pvnd6zb6TnwcTi":0.000001}' 494785
    020000...00000
    
    A> bitcoin-cli signrawtransaction 0200000...0000
    {
      "hex": "02000...0000",
      "complete": false
    }

She can then send this hex transaction to Bob. Bob can than verify (`decoderawtransaction`) and

    B> bitcoin-cli signrawtransaction 0200000...0000
    {
      "hex": "02000...0000",
      "complete": true
    }
	
Bob now has a transaction which he can and should publish just after block 494785 using software compatible with the legacy chain. This claims his funds. 

### Step 3: The transaction paying Alice

This transaction is the same as step 2, except it has a later locktime and no replay protection. The reason the transaction won't be replayed on the legacy chain is because they are already spend there. This time Bob creates it:

    B> bitcoin-cli createrawtransaction '[{"txid":"$MULTISIG_TXID$","vout":0,"scriptPubKey":"...","redeemScript":"..."}]' '{"$A_ADR$":2X+Y, "$B_ADR":Y}' 495085
    020000...00000
    
    B> bitcoin-cli signrawtransaction 0200000...0000
    {
      "hex": "02000...0000",
      "complete": false
    }

He sends it to Alice, who verifies (`decoderawtransaction`) and then signs:

    A> bitcoin-cli signrawtransaction 0200000...0000
    {
      "hex": "02000...0000",
      "complete": true
    }

Now Alice has her redeeming transaction which she can publish after block 495085 on the new chain. She should wait in publishing her transaction not only until 495085, but also until Bob confirmed his Step 2 transaction went through, as this is required to split the coins.

For any mistakes both in writing above and in typing them, and for any exceptional circumstances, please be kind and be sure not to lose your Knight's Honor.
