---
layout: page
title: Revoke a Bitcoin Inbound Transaction
---

If the locked funds are not redeemed by a certain amount of time, then the
cross-chain transaction changes to a state of `Revoked` within the smart
contracts. Once it changes to `Revoked`, the funds can no loger be redeemed by
the receiver, but instead must be revoked by the sender.

The amount of time before the cross-chain transaction expires is defined by the
cross-chain smart contracts. For Bitcoin it is 72 hours on mainnet and 8 hours
on testnet, while for Ethereum it is 8 hours on mainnet and 4 hours on testnet.

Unlike with inbound Ethereum transactions, inbound Bitcoin transactions are
revoked not by calling a smart contract, but instead by just spending the
bitcoin and sending them to an address that you fully own. The timelock for the
Bitcoin lock address, the user generated P2SH address, expires after a certain
period, after which the cross-chain transaction will be `Revoked` and you will
be able to spend from the lock address.

Thus, to revoke the transaction we just need to build and send a Bitcoin
transaction.

Let's start by making a new script file for the revoke.

```bash
$ vi btc2wbtc-revoke.js
```

The top portion of the script can be copied from the `btc2wbtc.js` script. The
only things we need to do are:
1. Add in the redeemKey of the expired cross-chain transaction, instead of
   creating a new redeemKey.
2. Add the txid of the Bitcoin transaction that funded the lock address.
3. Add the WIF of the revoker address.
2. Change the chain of promises so that it just calls the revoke function.

First, let's make sure to add in the `redeemKey` generated by the expired inbound
transaction, as well as the `lockTime`, `txid`, and `wif`.

```js
const opts = {
  ...
  redeemKey: {
    x: '1327484f41d1990247f5b7e180d0fe4f77e10e1fd414b4e871155b0aa30e44bc',
    xHash: 'd577db3fb3d8718648b4809938f72452c3b7f6613ac1f9159b06b7535b0197b8'
  },

  // Add lockTime used for P2SH address, and txid that funded it
  lockTime: 1542322930,
  txid: '34cbc29ec4edde4415260800561cb318b883b98629881a7d75ed85aa6eb41b03',

  // private key of from address
  wif: 'cNggJXP2mMNSHA1r9CRd1sv65uykZNyeH8wkH3ZPZVUL1nLfTxRM',
};
```

Finally, we need to specify the amount of miner fee we want to pay for our
revoke transaction.

```js
// Total miner fee for revoke (in satoshis)
const minerFee = '600';
```

Now we can set up the revoke call.

```js
Promise.resolve([])
  .then(sendRevoke)
```

Finally, we just need to define the `sendRevoke` function.

```js
function sendRevoke() {

  console.log('Starting btc inbound revoke', opts);

  // Build the contract to get the redeemScript
  const contract = cctx.buildHashTimeLockContract(opts);

  console.log('P2SH contract', contract);

  opts.redeemScript = contract.redeemScript;

  // Subtract miner fee
  const redeemValue = (new BigNumber(opts.value)).minus(minerFee).toString();

  // Get signed revoke tx
  const signedTx = cctx.buildRevokeTxFromWif(
    Object.assign({}, opts, {
      value: redeemValue,
    }),
  );

  console.log('Signed revoke tx:', signedTx);

  // Send the revoke tx to the network
  return utils.sendRawBtcTx(bitcoinRpc, signedTx);
}
```

In this `sendRevoke` function, basically two things happen. First, the Bitcoin
P2SH address is reconstructed using the `buildHashTimeLockContract` method, so
that the redeemScript can be captured and added to the transaction `opts`.
Second, the revoke transaction is constructed using the `buildRevokeTxFromWif`
method. The miner fee of 600 satoshis is subtracted from the previously defined
`value` before being passed to the `buildRevokeTxFromWif` method.

Go ahead and run the script.

```bash
$ node btc2wbtc-revoke.js
```

You should just see some output once the revoke call confirms. If you see an
error, it is more than likely due to one of two following problems.
1. You did not correctly set the redeemKey, lockTime, txid, or wif in the
   transaction opts.
2. The timelock has not yet expired.

Now that we have successfully sent and revoked a inbound Bitcoin cross-chain
transaction, we are ready to move on to discussing outbound Bitcoin cross-chain
transactions.