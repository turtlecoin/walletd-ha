# TurtleCoin Walletd High-Availability Wrapper

This project is designed to wrap the walletd process on a *nix system and monitor it for hangups, locks, and etc that cause the wallet to stop responding.

The sample service.js is includes a VERY basic example of how the wrapper can be used. For all options, please continue reading as it provides a VERY robust interface.

It also provides *easy access* to the walletd RPC API via native [Javascript Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises).

## Table of Contents

1. [To Do](#to-do)
2. [Dependencies](#dependencies)
3. [Easy Start](#easy-start)
4. [Keep it Running](#keep-it-running)
5. [Documentation](#documentation)
   1. [Methods](#methods)
   2. [Events](#events)
   3. [Walletd RPC API Interface](#walletd-rpc-api-interface)
   4. [WebSocket Connections](#websocket-connections)
 
## To Do

2. Expose all walletd RPC APIs in the package to the WebSocket
3. After the wallet container is synced, compare the wallet height to the network_height of the daemon (or public node) to detect if the wallet is out of sync.

## Dependencies

* NodeJS v8.x
* walletd v0.5.0 or higher [https://github.com/turtlecoin/turtlecoin](https://github.com/turtlecoin/turtlecoin)

## Easy Start

You *must* copy walletd into the ```walletd-ha``` folder for the easy start process to occur.

```text
git clone https://github.com/brandonlehmann/walletd-ha.git
cd walletd-ha
cp <walletd> .
./walletd -g -w container.walletd
npm i & node service.js
```

**It is highly recommended that you create a container with a password and pass that into the wrapper. For your own security, please make sure that you use passwords for both the RPC server and the container itself. To do otherwise will fill you with regret.**

## Keep it Running

I'm a big fan of PM2 so if you don't have it installed, the setup is simple.

```text
npm install -g pm2

pm2 startup
pm2 install pm2-logrotate

pm2 start service.js --watch --name walletd
pm2 save
```

## Documentation

### Initilization

Practically all of the walletd command line arguments are exposed in the constructor method. Simply include them in your list of options to activate or use them. Default values are defined below. As always, please use values that make sense for your implementation.

There are a lot of different options available so reading through the full list is to your advantage.

```javascript
var wallet = new Walletd({
  appName: 'default', // This defines an application name used to store some settings.
  pollingInterval: 10, // Check to make sure that walletd is alive every x seconds
  maxPollingFailures: 3, // After the polling checks fail x times, report the walletd process down
  saveInterval: 10, // issue an automatic save request every x seconds as long as the wallet is synced
  scanInterval: 5, // scan the wallet for new transactions every x seconds as long as the wallet is synced
  timeout: 2000, // consider RPC calls timed out after x milliseconds
  path: './walletd', // the path to the walletd binary
  enableWebSocket: true, // enable the WebSocket server at bindPort + 1
  
  // Standard walletd options start here
  config: false, // the path to a walletd config file -- if you so choose
  localNode: false, // If set to true, activates the in-process walletd daemon/node
  bindAddress: '127.0.0.1', // The IP address that walletd will bind to
  bindPort: 8070, // The port that walletd will bind to
  rpcPassword: false, // You really should use an RPC password
  rpcLegacySecurity: false, // Turning this to true, removes the requirement for a RPC password, either rpcPassword or rpcLegacySecurity MUST be set
  containerFile: false, // The path to your walletd container file
  containerPassword: false, // The password to your walletd container file
  logFile: false, // The path to the log file you would like walletd to keep
  logLevel: 4, // The log level to use with walletd
  syncFromZero: false, // If set to true, will tell walletd to always sync the container from zero.
  daemonAddress: '127.0.0.1', // When using a remote node (localNode === false), provide the IP address or hostname of the daemon here
  daemonPort: 11898, // Remote daemon port
  
  // RPC API default values
  defaultMixin: 6, // the default mixin to use for transactions
  defaultFee: 0.1, // the default transaction fee for transactions
  defaultBlockCount: 1, // the default number of blocks when blockCount is required
  decimalDivisor: 100, // Currency has many decimal places?
  defaultFirstBlockIndex: 1, // the default first block index we will use when it is required
  defaultUnlockTime: 0, // the default unlockTime for transactions
  defaultFusionThreshold: 10000000, // the default fusionThreshold for fusion transactions
  
  // The following options are for when we run the in-process node
  testnet: false, // If true, activates the testnet logic -- only important if localNode is true
  dataDir: '~/.TurtleCoin', // If the localNode is used, this is where the blockchain data will be stored
  p2pBindIp: '0.0.0.0', // What IP to bind the P2P network to
  p2pBindPort: 11897, // What Port to bind the P2P network to
  p2pExternalPort: 0, // What External Port to bind the P2P network to for those behind NAT
  allowLocalIp: false, // Add our own IP to the peer list?
  peers: false, // Manually add the peer(s) to the list. Allows for a string or an Array of strings.
  priorityNodes: false, // Manually add the priority node(s) to the peer list. Allows for a string or an Array of strings.
  exclusiveNodes: false, // Only add these node(s) to the peer list. Allows for a string or an Array of strings.
  seedNode: false, // Connect to this node to get the peer list then quit. Allows for a string.
  hideMyPort: false, // Hide from the rest of the network
})
```

## Methods

### wallet.start()

Starts walletd and starts monitoring the process.

```javascript
wallet.start()
```

### wallet.stop()

Stops walletd and halts all monitoring processes.

```javascript
wallet.stop()
```

### wallet.write(text)

If you really must write text to the actual walletd console, you can do so with this method. You'll need to parse the output of the *data* event as defined below.

```javascript
wallet.write('help')
```

## Events

### Event - *close*

This event is fired when the underlying walletd process closes, dies, is killed for whatever reason. Usually, when this occurs we want to restart the process with ```wallet.start()```

```javascript
wallet.on('close', (exitcode) => {
  wallet.start()
})
```

### Event - *data*

Provides the actual console output of walletd on a per line basis.

```javascript
wallet.on('data', (data) => {
  // do something
})
```

### Event - *down*

This event is fired when we detected that walletd appears **down**. It means that it is not responding to RPC requests. If you want to make it automatically restart walletd when this occurs, simply ```wallet.stop()``` in the event.

```javascript
wallet.on('down', () => {
  wallet.stop()
})
```

### Event - *error*

Provides the event for when an error event is encountered. These are bad, something isn't working right.

```javascript
wallet.on('error', (err) => {
  // do something
})
```

### Event - *info*

Provides the event for when an informational event is encountered.

```javascript
wallet.on('info', (info) => {
  // do something
})
```

### Event - *save*

This event is fired every save interval.

```javascript
wallet.on('save', () => {
  // do something
})
```

### Event - *scan*

This event is fired every scan interval to let us know that the service is currently scanning blocks for transactions that we need to handle.

```javascript
wallet.on('scan', (fromBlock, toBlock) => {
  // do something
})
```

### Event - *status*

This event is fired every polling cycle. It returns the equivalent of the walletd *getStatus* API call.

```javascript
wallet.on('status', (status) => {
  // do something
})
```

#### Example Data

```javascript
{
  "blockCount": 491136,
  "knownBlockCount": 491137,
  "lastBlockHash": "bc25f55db114fbe99720cb776b2c3b4787803edf1fb4cdd1851772087113b8eb",
  "peerCount": 8
}
```

### Event - *synced*

This event is fired when walletd is fully synchronized with the network.

```javascript
wallet.on('synced', () => {
  // do something
})
```

### Event - *transaction*

This event is fired for **each** transaction that walletd says belongs to us. It contains the information for each ***transfer*** in the transaction rolled up into a single object for each ***transfer*** with one small addition of ```inbound``` which is a *boolean*. It will ***only*** fire for the portion(s) of the transaction where the transfer **belong** to the wallet container.

***Special Note: Any and all amounts/fees will already be in HUMAN readable units. DO NOT DIVIDE THEM AGAIN unless you've specified ```decimalDivisor``` as ```1``` in the options. You have been warned.***

```javascript
wallet.on('transaction', (transaction) => {
  // do something very useful
})
```

#### Example Data

```javascript
{
  "blockHash": "f98d6bbe80a81b3aa0aebd004096e2223524f58f347a1f21be122450f244b948",
  "transactionAmount": 10,
  "blockIndex": 469419,
  "extra": "014fa15a893c92e040fc97c8bda6d811685a269309b37ad444755099cbed6d8438",
  "fee": 0.1,
  "isBase": false,
  "paymentId": "",
  "state": 0,
  "timestamp": 1526876765,
  "transactionHash": "d01e448f7b631cebd989e3a150258b0da59c66f96adecec392bbf61814310751",
  "address": "TRTLv2MXbzaPYVYqtdNwYpKY7azcVjBjsETN188BpKwi2q83NibqJWtFYL9CHxpWph2wCPZcJ6tkPfUxVZcUN8xmYsSDJYpcE3D",
  "amount": 10,
  "type": 0,
  "unlockTime": 0,
  "inbound": true
}
```

### Event - *warning*

Provides the event for when a warning event is encountered. These are warnings saying that it might still work but you really should fix whatever the problem is.

```javascript
wallet.on('warning', (warn) => {
 // do something
})
```

## Walletd RPC API Interface

As we can actually run this wrapper inside another nodeJS project, we expose all of the walletd RPC API commands via the ```wallet.api``` property. Each of the below methods are [Javascript Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises). For safety sake, **always** handle your promise catches as we do use them properly.

***Special Note: Any and all amounts/fees will already be in HUMAN readable units. DO NOT DIVIDE THEM AGAIN unless you've specified ```decimalDivisor``` as ```1``` in the options. You have been warned.***

Unless otherwise noted, all methods will resolve the promise upon success and sample return data is supplied below. Any errors will reject the promise with an error condition.

Methods noted having options have parameters that may be *optional* or *required* as documented.

### wallet.api.reset()

### wallet.api.save()

### wallet.api.getViewKey()

#### Example Data

```javascript
{
  "viewSecretKey": "12345678901234567890"
}
```

### wallet.api.getSpendKeys(options)

#### Parameters

```options.address```: Public Wallet Address - *required*

#### Example Data

```javascript
{
  "spendPublicKey": "9e50b808f1e2522b7c6feddd8e2f6cdcd89ff33b623412de2061d78c84588eff33b6d9",
  "spendSecretKey": "c6639a75a37f63f92e2f096fa262155c943b4fdc243ffb02b8178ab960bb5d0f"
}
```

### wallet.api.getStatus()

#### Example Data

```javascript
{
  "blockCount": 491214,
  "knownBlockCount": 491215,
  "lastBlockHash": "fc33b0fcdb8a3ed8e2de3cb36df325d67e9926d59f02d164baacf3ddefe8df12",
  "peerCount": 8
}
```

### wallet.api.getAddresses()

#### Example Data

```javascript
[
  "TRTLux9QBmzCYEGgdWXHEQCAm6vY9vZHkbGmx8ev5LxhYk8N71Pp7PWFYL9CHxpWph2wCPZcJ6tkPfUxVZcUN8xmYsSDJZ25i9n",
  "TRTLv1mPerM2ckUuNvxrkzDE7QKd9PFVUXYbVfbvx8YxB5BYEdSqQvUFYL9CHxpWph2wCPZcJ6tkPfUxVZcUN8xmYsSDJbQMVgF"
]
```

### wallet.api.createAddress(options)

#### Parameters

```options.secretSpendKey```: Address secret spend key - *optional*

```options.publicSpendKey```: Address public spend key - *optional*

**Note:** Both ```secretSpendKey``` and ```publicSpendKey``` are optional; however, you can only supply one or the other. Both are given below as **examples**.

#### Example Data

```javascript
{
  "address": "TRTLv3rnGMvAdUUPZZxUmm2jSe8j9U4EfXoAzT3NByLTKD4foK6JuH2FYL9CHxpWph2wCPZcJ6tkPfUxVZcUN8xmYsSDJYidUqc"
}
```

### wallet.api.deleteAddress(options)

#### Parameters

```options.address```: Public address to delete - *required*

### wallet.api.getBalance(options)

#### Parameters

```options.address```: Public address - *optional*

#### Example Data

```javascript
{
  "availableBalance": 60021.54,
  "lockedAmount": 0
}
```

### wallet.api.getBlockHashes(options)

#### Parameters

```options.firstBlockIndex```: The height to start with - *required*

```options.blockCount```: How many blocks to return at maximum - *required*

#### Example Data

```javascript
{
  "blockHashes": [
    "8c9738f961a278486f27ce214d1e4d67e08f7400c8b38fe00cdd571a8d302c7d",
    "2ef060801dd27327533580cfa538849f9e1968d13418f2dd2535774a8c494bf4",
    "3ac40c464986437dafe9057f73780e1a3a6cd2f90e0c5fa69c5caab80556a68a",
    "ac821fcb9e9c903abe494bbd2c8f3333602ebdb2f0a98519fc84899906a7f52b",
    "4dcffeea7aec064ec5c03e1cb6cf58265a2b76c4f2db9e5fc4afbaf967b77bba",
    "1b82b0df589cb11aa5a96ea97d79699af7bc54b5d2b8333847d38da660aaf9e0",
    "007de12510667a1d56b61720257f07a3905abb3a8b479bdff926bb17d1a9e766",
    "8f0d10ddf23aafb755e682291d56d38a20bbc17ce1d5081c15067865b6867260",
    "5585c6bac11925fc762d0a8e6b95b3a3bd66379e74e8711e432fda3f6966bf08",
    "ea531b1af3da7dc71a7f7a304076e74b526655bc2daf83d9b5d69f1bc4555af0"
  ]
}
```

### wallet.api.getTransactionHashes(options)

#### Parameters

```options.addresses```: Array of public addresses to scan for - *optional*

```options.blockHash```: Block hash to scan *optional/required*

```options.firstBlockIndex```: The height to start with - *optional/required*

```options.blockCount```: How many blocks to return at maximum - *required*

```options.paymendId```: Payment ID to scan for - *optional*

***Note:*** Only **one** of either ```blockHash``` or ```firstBlockIndex``` may be supplied, but not both.

#### Example Data

```javascript
{
  "items": [
    {
      "blockHash": "f98d6bbe80a81b3aa0aebd004096e2223524f58f347a1f21be122450f244b948",
      "transactionHashes": [
        "d01e448f7b631cebd989e3a150258b0da59c66f96adecec392bbf61814310751"
      ]
    }
  ]
}
```

### wallet.api.getTransactions(options)

#### Parameters

```options.addresses```: Array of public addresses to scan for - *optional*

```options.blockHash```: Block hash to scan *optional/required*

```options.firstBlockIndex```: The height to start with - *optional/required*

```options.blockCount```: How many blocks to return at maximum - *required*

```options.paymendId```: Payment ID to scan for - *optional*

***Note:*** Only **one** of either ```blockHash``` or ```firstBlockIndex``` may be supplied, but not both.

#### Example Data

```javascript
[
  {
    "blockHash": "f98d6bbe80a81b3aa0aebd004096e2223524f58f347a1f21be122450f244b948",
    "transactionAmount": 10.5,
    "blockIndex": 469419,
    "extra": "014fa15a893c92e040fc97c8bda6d811685a269309b37ad444755099cbed6d8438",
    "fee": 0.1,
    "isBase": false,
    "paymentId": "",
    "state": 0,
    "timestamp": 1526876765,
    "transactionHash": "d01e448f7b631cebd989e3a150258b0da59c66f96adecec392bbf61814310751",
    "address": "TRTLv2MXbzaPYVYqtdNwYpKY7azcVjBjsETN188BpKwi2q83NibqJWtFYL9CHxpWph2wCPZcJ6tkPfUxVZcUN8xmYsSDJYpcE3D",
    "amount": 10.5,
    "type": 0,
    "unlockTime": 0,
    "inbound": true
  }
]
```

### wallet.api.getUnconfirmedTransactionHashes(options)

#### Parameters

```options.addresses```: Array of public address to scan for - *optional*

#### Example Data

```javascript
{
  "transactionHashes": [
    "80185093fj029jv029j3g092jb32904j0b34jb34gb",
    "j09213fj20vjh02vb2094jb0394jgb039bj03jb34b"
  ]
}
```

### wallet.api.getTransaction(options)

***Special Note: Any and all amounts/fees will already be in HUMAN readable units. DO NOT DIVIDE AMOUNTS AGAIN unless you've specified ```decimalDivisor``` as ```1``` in the options. You have been warned.***

#### Parameters

```options.transactionHash```: The hash of the transaction - *required*

#### Example Data

```javascript
{
  "transaction": {
    "amount": 10,
    "blockIndex": 469419,
    "extra": "014fa15a893c92e040fc97c8bda6d811685a269309b37ad444755099cbed6d8438",
    "fee": 0.1,
    "isBase": false,
    "paymentId": "",
    "state": 0,
    "timestamp": 1526876765,
    "transactionHash": "d01e448f7b631cebd989e3a150258b0da59c66f96adecec392bbf61814310751",
    "transfers": [
      {
        "address": "TRTLv2MXbzaPYVYqtdNwYpKY7azcVjBjsETN188BpKwi2q83NibqJWtFYL9CHxpWph2wCPZcJ6tkPfUxVZcUN8xmYsSDJYpcE3D",
        "amount": 10,
        "type": 0
      },
      {
        "address": "",
        "amount": -20,
        "type": 0
      },
      {
        "address": "",
        "amount": 9.9,
        "type": 0
      }
    ],
    "unlockTime": 0
  }
}
```

### wallet.api.newTransfer(address, amount)

This method creates a transfer object designed to be used with *wallet.api.sendTransaction*

***Special Note: Any and all amounts/fees will already be in HUMAN readable units. DO NOT SUPPLY NATIVE CURRENCY AMOUNTS unless you've specified ```decimalDivisor``` as ```1``` in the options. You have been warned.***

### wallet.api.sendTransaction(options)

***Special Note: Any and all amounts/fees will already be in HUMAN readable units. DO NOT SUPPLY NATIVE CURRENCY AMOUNTS unless you've specified ```decimalDivisor``` as ```1``` in the options. You have been warned.***

#### Parameters

```options.addresses```: Array of addresses to use for the *inputs* - *optional*

```options.transfers```: Array of transfer objects (see *wallet.api.newTransfer*) to send funds to - *required*

```options.fee```: Fee we are willing to pay for the transaction. Ex: 0.1 - *optional*

```options.unlockTime```: Blockheight to unlock the transaction at, the UTC timestamp, or ```0``` for now. - *optional*

```options.mixin```: Mixins to use - *optional*

```options.extra```: Extra data to put in the transaction - *optional*

```options.paymentId```: The payment ID for the transaction - *optional*

```options.changeAddress```: Where to send any change from the transaction to. If not specified, the first address in the wallet container is used. - *optional*

#### Example Data

```javascript
{
  "transactionHash": "93faedc8b8a80a084a02dfeffd163934746c2163f23a1b6022b32423ec9ae08f"
}
```

### wallet.api.createDelayedTransaction(options)

***Special Note: Any and all amounts/fees will already be in HUMAN readable units. DO NOT SUPPLY NATIVE CURRENCY AMOUNTS unless you've specified ```decimalDivisor``` as ```1``` in the options. You have been warned.***

#### Parameters

```options.addresses```: Array of addresses to use for the *inputs* - *optional*

```options.transfers```: Array of transfer objects (see *wallet.api.newTransfer*) to send funds to - *required*

```options.fee```: Fee we are willing to pay for the transaction. Ex: 0.1 - *optional*

```options.unlockTime```: Blockheight to unlock the transaction at, the UTC timestamp, or ```0``` for now. - *optional*

```options.mixin```: Mixins to use - *optional*

```options.extra```: Extra data to put in the transaction - *optional*

```options.paymentId```: The payment ID for the transaction - *optional*

```options.changeAddress```: Where to send any change from the transaction to. If not specified, the first address in the wallet container is used. - *optional*

#### Example Data

```javascript
{
  "transactionHash": "93faedc8b8a80a084a02dfeffd163934746c2163f23a1b6022b32423ec9ae08f"
}
```

### wallet.api.getDelayedTransactionHashes()

#### Example Data

```javascript
{
  "transactionHashes": [
    "957dcbf54f327846ea0c7a16b2ae8c24ba3fa8305cc3bbc6424e85e7d358b44b",
    "25bb751814dd39bf46c972bd760e7516e34200f5e5dd02fda696671e11201f78"
  ]
}
```

### wallet.api.deleteDelayedTransaction(options)

#### Parameters

```options.transactionHash```: The hash of the transaction - *required*

### wallet.api.sendDelayedTransaction()

#### Parameters

```options.transactionHash```: The hash of the transaction - *required*

### wallet.api.sendFusionTransaction(options)

#### Parameters

```options.threshold```: The minimum fusion threshold amount - *optional*

```options.mixin```: Mixins to use - *optional*

```options.addresses```: Array of addresses to use for the *inputs* - *optional*

```options.destinationAddress```: The address to send the fusion transaction to - *optional/required*

***Note:*** If the container has only one address or ```addressess``` consists of one address, then ```destinationAddress``` need not be supplied. Otherwise, ```destinationAddress``` is required.

#### Example Data

```javascript
{
  "transactionHash": "93faedc8b8a80a084a02dfeffd163934746c2163f23a1b6022b32423ec9ae08f"
}
```

### wallet.api.estimateFusion(options)

#### Parameters

```options.threshold```: The minimum fusion threshold amount - *optional*

```options.addresses```: Array of addresses to use for the *inputs* - *optional*

#### Example Data

```javascript
{
  "fusionReadyCount": 0,
  "totalOutputCount": 19
}
```

## WebSocket Connections

A WebSocket [socket.io](https://socket.io/) server is initialized if ```enableWebSocket``` is true in the initialization of the module. The server is created on the ```bindPort``` specified + ```1```.

This server requires that you the client authenticates otherwise you will **not** receive any of the below events aside from the *challenge* event. Authentication must occur within 5 seconds or the socket will be disconnected.

### Client Initiated Events

|Event|Payload|
|---|---|
|challenge|*string* sha256 hash of password|

**Note:** Passing an invalid password will disconnect the socket.

### Server Initiated Events

|Event|Authentication Required|Payload|
|---|---|---|
|challenge|No|*boolean* Always **true**|
|auth|No|*boolean* Responds to a client initiated *challenge* event. If **true** the password was correct. If **false** the password was incorrect.
|close|Yes|*integer* Walletd exitcode upon close|
|data|Yes|*string* Console output of walletd|
|down|Yes||
|error|Yes|*string* Error message|
|info|Yes|*string* Information message|
|save|Yes||
|scan|Yes|Ex. ```{fromBlock: 1, toBlock: 10}```|
|status|Yes|See. [status event](#event---status)
|synced|Yes||
|transaction|Yes|See. [transaction event](#event---transaction)|
|warning|Yes|*string* Warning message|
