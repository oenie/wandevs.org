---
layout: page
title: Revoke an Ethereum Inbound Transaction
---

If the locked funds are not redeemed by a certain amount of time, then the
cross-chain transaction changes to a state of `Revoked` within the smart
contracts. Once it changes to `Revoked`, the funds can no longer be redeemed by
the receiver, but instead must be revoked by the sender.

The amount of time before the cross-chain transaction expires is defined by the
cross-chain smart contracts. For Ethereum it is 8 hours on mainnet and 4 hours
on testnet, while for Bitcoin it is 72 hours on mainnet and 8 hours on testnet.

Revoking a transaction involves only making a single contract call. For our
example, going inbound from Ethereum to Wanchain, we need to make the call on
Ethereum so that we can get our funds sent back to the Ether account.

Let's start by making a new script file for the revoke.

```bash
$ vi eth2weth-revoke.js
```

The top portion of the script can be copied from the `eth2weth.js` script. The
only things we need to do are:
1. Add in the redeemKey of the expired cross-chain transaction, instead of
   creating a new redeemKey.
2. Change the chain of promises so that it just calls the revoke function.

First, let's make sure to add in the redeemKey generated by the expired inbound
transaction.

```js
const opts = {
  ...
  redeemKey: {
    x: '<the x value>',
    xHash: '<the xHash value>',
  }
};
```

Then we can set up the revoke call.

```js
Promise.resolve([])
  .then(sendRevoke)
  .catch(err => {
    console.log('Error:', err);
  });
```

Then, we just need to define the `sendRevoke` function.

```js
function sendRevoke() {

  // Get the raw revoke tx
  const revokeTx = cctx.buildRevokeTx(opts);

  // Send the revoke transaction on Ethereum
  const receipt = await utils.sendRawEthTx(web3eth, revokeTx, opts.from, ethPrivateKey)

  console.log('Revoke sent:', receipt);
}
```

Like with the `sendLock` and `sendRedeem` functions, our `sendRevoke` function
uses WanX to build the revoke transaction, and then uses the helper function to
sign and send the transaction to the network.

Go ahead and run the script.

```bash
$ node eth2weth-revoke.js
```

You should just see some output once the revoke call confirms. If you see an
error, it is more than likely due to one of two following problems.
1. You did not correctly set the redeemKey in the transaction opts.
2. The timelock has not yet expired.

Now that we have successfully sent and revoked inbound Ethereum cross-chain
transactions, we are ready to move on to discussing outbound Ethereum
cross-chain transactions.
