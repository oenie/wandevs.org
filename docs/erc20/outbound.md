---
layout: page
title: ERC20 Outbound Transaction
---

Now that we've successfully done an inbound ERC20 cross-chain transaction,
we can now do an outbound transaction.

<div class="alert alert-info">
  <b>Note</b>: Outbound transactions require that the sending Wanchain account
  has a positive balance of the cross-chain token. To convert back to the DAI
  token on Ethereum, for example, your Wanchain account needs have some WDAI
  token.
</div>

Start by creating a new script file.

```bash
$ vi wdai2dai.js
```

The set up portion of the script will be mostly identical to the one in the
`dai2wdai.js` script covered above, but with a couple minor differences.

First, in the `opts` object, the `to` now should be the receiving Ethereum
account, and the `from` should be the sending Wanchain account. The
`ethKeyObject` should be derived from the `to` account, and the `wanKeyObject`
should be derived from the `from` account.

#### wdai2dai.js
```js
const WanX = require('wanx');
const Web3 = require('web3');
const keythereum = require('keythereum');
const utils = require('./utils');

const web3wan = new Web3(new Web3.providers.HttpProvider('http://localhost:18545'));
const web3eth = new Web3(new Web3.providers.HttpProvider('https://rinkeby.infura.io/<myToken>');

const config = {
  wanchain: { web3: web3wan },
  ethereum: { web3: web3eth },
};

const wanx = new WanX('testnet', config);

const opts = {
  token: {
    eth: '0xdbf193627ee704d38495c2f5eb3afc3512eafa4c', // DAI
    wan: '0xda16e66820a3c64c34f2b35da3f5e1d1742274cb',
  },
  from: '0xa6d72746a4bb19f46c99bf19b6592828435540b0', // Wanchain
  to: '0x4bbdfe0eb33ed498020de9286fd856f5b8331c2c',   // Ethereum
  value: '2101000000000000',
  storeman: {
    wan: '0x06daa9379cbe241a84a65b217a11b38fe3b4b063',
    eth: '0x41623962c5d44565de623d53eb677e0f300467d2',
  },
  redeemKey: wanx.newRedeemKey(),
}

const ethDatadir = '/home/<myUser>/.ethereum/testnet/';
const ethKeyObject = keythereum.importFromFile(opts.to, ethDatadir); // Use `to` address
const ethPrivateKey = keythereum.recover('<myPassword>', ethKeyObject);

const wanDatadir = '/home/<myUser>/.wanchain/testnet/';
const wanKeyObject = keythereum.importFromFile(opts.from, wanDatadir); // Use `from` address
const wanPrivateKey = keythereum.recover('<myPassword>', wanKeyObject);
```

Next we will initialize WanX, though now as an outbound transaction.

```js
// New crosschain transaction
// erc20, outbound
const cctx = wanx.newChain('erc20', false);
```

And again, we will log out the `opts`, in case we need to know and use the
generated `redeemKey`.

```js
console.log('Tx opts:', opts)
```

Now we are ready to kick off the transaction. The format of an outbound
transaction is the same as the format of an inbound transaction, with a couple
of differences.
1. The chains are reversed. That is, the lock is called on Wanchain instead of
   Ethereum, etc.
2. Outbound transactions require an additional fee payed in WAN

These changes can be made within the specific functions, and thus we can use
the same series of promises that was used for the inbound transaction.

```js
Promise.resolve([])
  .then(sendApprove)
  .then(sendLock)
  .then(confirmLock)
  .then(sendRedeem)
  .then(confirmRedeem)
  .catch(err => {
    console.log('Error:', err);
  });
```

And let's go ahead and define these functions.

```js
async function sendApprove() {

  // Get the raw approve tx
  const approveTx = cctx.buildApproveTx(opts);

  // Send the lock tx on Wanchain
  const receipt = await utils.sendRawWanTx(web3wan, approveTx, opts.from, wanPrivateKey);

  console.log('Approve sent:', receipt);
}

async function sendLock() {

  console.log('Starting erc20 outbound tx', opts);

  // Get the current block number on Wanchain
  const fee = await cctx.getOutboundFee(opts);
  opts.outboundFee = fee;

  // Get the raw lock tx
  const lockTx = cctx.buildLockTx(opts);

  // Send the lock transaction on Ethereum
  const receipt = await utils.sendRawWanTx(web3wan, lockTx, opts.from, wanPrivateKey)

  console.log('Lock sent:', receipt);
}

async function confirmLock() {

  // Get the current block number on Wanchain
  const blockNumber = await web3wan.eth.getBlockNumber();

  // Scan for the lock confirmation from the storeman
  const log = await cctx.listenLock(opts, blockNumber);

  console.log('Lock confirmed:', log);
}

async function sendRedeem() {

  // Get the raw redeem tx
  const redeemTx = cctx.buildRedeemTx(opts);

  // Send the redeem transaction on Ethereum
  const receipt = await utils.sendRawEthTx(web3eth, redeemTx, opts.to, ethPrivateKey);

  console.log('Redeem sent:', receipt);
}

async function confirmRedeem() {

  // Get the current block number on Wanchain
  const blockNumber = await web3wan.eth.getBlockNumber();

  // Scan for the redeem confirmation from the storeman
  const log = await cctx.listenRedeem(opts, blockNumber);

  console.log('Redeem confirmed', log);
  console.log('COMPLETE!!!');
}
```

The `sendLock` function includes an additional step to get the outbound fee and
to attach it to the `opts` object before passing the `opts` into the
`buildLockTx` method. Otherwise, as mentioned, the only other change is that
the chains are reversed in respect to the inbound version.

<div class="alert alert-info">
  <b>Note</b>: the <code>confirmRedeem</code> function call is technically not
  needed, since the token are sent to the recipient's account once the
  <code>sendRedeem</code> succeeds. The example adds the final step only for
  completeness.
</div>

That's it. Go ahead and run the script.

```bash
$ node wdai2dai.js
```
