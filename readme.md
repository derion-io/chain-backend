# Chain Backend

* A library to retrieve and strore data from Binance Smart Chain network.
* [Quick Start](#quick-start)
* [APIs](#apis)
* [Types](#types)
* [Development](#development)

## Example

```js
const { ethers } = require('ethers')
const {JsonRpcProvider} = require('@ethersproject/providers')
const {Mongoose} = require('mongoose')
const {
    startWorker, 
    createAccumulatorConsumer,
    createChainlogConfig
} = require('../index')
const { ZERO_HASH } = require('../helpers/constants').hexes
const contractABI = require('../ABIs/SFarm.json').abi

async function _createMongoose() {
    let mongoose = new Mongoose()
    let endpoint = 'mongodb://localhost/sfarm'

    await mongoose.connect(endpoint, { 
        useNewUrlParser: true, 
        useUnifiedTopology: true
    })

    return mongoose
}

function createConsumer(config) {
    let sfarmContract = new ethers.Contract(
        '0x8141AA6e0f40602550b14bDDF1B28B2a0b4D9Ac6', 
        contractABI
    )

    return createAccumulatorConsumer({
        key: 'consumer_1',
        filter: sfarmContract.filters.AuthorizeAdmin(null, null),
        genesis: 8967359,
        mongoose: config.mongoose,
        applyLogs: (value, logs) => {
            value = {...value}
            logs.forEach(log => {
                const address = ethers.utils.getAddress(
                    '0x'+log.topics[1].slice(26)
                )

                if (log.data != ZERO_HASH) {
                    value[address] = true
                } else {
                    delete value[address]
                }
            })

            return value
        }
    })
}

async function main() {
    let mongoose = await _createMongoose()
    let ethersProvider = new JsonRpcProvider(
        'https://bsc-dataseed.binance.org'
    )
    let headProcessorConfig = createChainlogConfig({
        type: 'HEAD',
        config: {
            provider: ethersProvider,
            size: 6,
            concurrency: 1,
        },
        hardCap: 4000,
        target: 500,
    })
    let pastProcessorConfig = createChainlogConfig({
        type: 'PAST',
        config: {
            provider: ethersProvider,
            size: 4000,
            concurrency: 10,
        },
        hardCap: 4000,
        target: 500,
    })

    await startWorker({
        consumerConstructors: [
            createConsumer
        ],
        mongoose: mongoose,
        ethersProvider: ethersProvider,
        headProcessorConfig: headProcessorConfig,
        pastProcessorConfig: pastProcessorConfig
    })
}

main().catch(console.error)
```

## APIs

```js
const {
    startWorker, 
    createAccumulatorConsumer,
    createSyncConsumer,
    createUpdateConsumer,
    createChainlogConfig
} = require('chain-backend')

// Description
//  * Start a worker that retrieve and store data from Binance Smart Chain
//    network.
//
// Input
//  * config {WorkerConfiguration}
async function startWorker(config) {}

// Description
// ?
//
// Input
//  * config {AccumulatorConsumerConfig}
//
// Output {Consumer}
function createAccumulatorConsumer(config) {}

function createSyncConsumer() {}
function createUpdateConsumer() {}
function createChainlogConfig() {}
```

## Types

```js
// Type WorkerConfiguration {Object}
//  * consumerConstructors {Array<ConsumerConstructor>}
//  * mongoose {mongoose.Mongoose}
//  * ethersProvider {ethers.providers.JsonRpcProvider}
//  * pastProcessorConfig {ProcessorConfig}
//  * headProcessorConfig {ProcessorConfig}

// Type ConsumerConstructor {function(config)}
//
// Input
//  * config.mongoose {mongoose.Mongoose}
//  
// Output {Consumer}


// Type ProcessorConfig {Object}
//  * getLogs {function ?}
//  * getConcurrency {function ?}
//  * getSize {function ?}


// Type AccumulatorConsumerConfig
//  * key {String}
//  * filter {ethers.Contract.Filter}
//  * genesis {String}
//  * mongo {MongoService}
//  * applyLogs {function ?}

// Type Consumer {Object}
//  * key {String}
//  * getRequests {function(?)}
```

## Development

```bash
npm install
npm test
```
