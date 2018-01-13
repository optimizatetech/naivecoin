# Naivecoin - una implementación de una criptomoneda en menos de 1500 líneas de código 

### Motivación  
Las criptomonedas y los contratos inteligentes que funcionan sobre las cadenas de bloques no son los conceptos más intuitivos. Conceptos como carteras (wallets), direcciones (addresses), prueba de trabajo (prove-of-work), transacciones (transactions) y sus firmas (signatures) son conceptos que deben entenderse en su contexto. Este proyecto está inspirado por el repositorio llamado [naivechain](https://github.com/lhartikk/naivechain). Este proyecto es un intento de implementar de la forma más sencilla una criptomoneda.   

### ¿Qué es una criptomoneda? 
[De Wikipedia](https://es.wikipedia.org/wiki/Criptomoneda) : una criptomoneda es un valor digital diseñado para ser utilizado como medio de intercambio. La criptografía se utiliza para proporcionar seguridad a las transacciones y para controlar la creación de unidades de la moneda. 

### Conceptos clave de Naivecoin 
* Componentes: 
    * Servidor HTTP
    * Nodo
    * Blockchain (cadena de bloques)
    * Operador
    * Minero
* Interfaz para controlar todo por una API HTTP
* Sincronización de la cadena de bloques y transacciones
* Prueba de trabajo sencilla (La dificultad incrementa cada 5 bloques)
* Creación de direcciones usando un enfoque determinista [EdDSA](https://en.wikipedia.org/wiki/EdDSA)
* Persistencia de datos en un directorio

> Naivechain utiliza comunicación con websocket en p2p, pero se desistió su implementación para simplificar el ejemplo de este caso, para entender mejor la transacción de mensajes. Entonces, en este repositorio se utilizará comunicación REST únicamente.

#### Comunicación de los componentes
```
               +---------------+
               |               |
     +------+--+ Servidor HTTP +---------+
     |      |  |               |         |
     |      |  +-------+-------+         |
     |      |          |                 |
+----v----+ |  +-------v------+    +-----v------+
|         | |  |              |    |            |
| Minero  +---->  Blockchain  <----+  Operador  |
|         | |  |              |    |            |
+---------+ |  +-------^------+    +------------+
            |          |
            |     +----+---+
            |     |        |
            +----->  Nodo  |
                  |        |
                  +--------+
```

No todos los componentes de este ejemplo siguen una implementación que se pueda considerar como segura y escalable. En el código fuente se han colocado comentarios con INFO en donde se informa de cómo se debería de hacer para solucionar los problemas correspondientes.

#### Servidor HTTP 
Propociona una API para gestionar la cadena de bloques, carteras, direcciones, creación de transacciones, solicitud de minados y conectividad entre nodos.  

Es el punto de entrada para interactuar con la moneda y cada nodo proporciona una API (swagger) para facilitar la interacción). Los puntos disponibles serían:  

##### Cadena de bloques (Blockchain)

|Method|URL|Description|
|------|---|-----------|
|GET|/blockchain/blocks|Get all blocks|
|GET|/blockchain/blocks/{index}|Get block by index|
|GET|/blockchain/blocks/{hash}|Get block by hash|
|GET|/blockchain/blocks/latest|Get the latest block|
|PUT|/blockchain/blocks/latest|Update the latest block|
|GET|/blockchain/transactions|Get all transactions|
|POST|/blockchain/transactions|Create a transaction|
|GET|/blockchain/blocks/transactions/{transactionId}|Get a transaction from some block|
|GET|/blockchain/transactions/unspent|Get unspent transactions|

##### Operator

|Method|URL|Description|
|------|---|-----------|
|GET|/operator/wallets|Get all wallets|
|POST|/operator/wallets|Create a wallet from a password|
|GET|/operator/wallets/{walletId}|Get wallet by id|
|POST|/operator/wallets/{walletId}/transactions|Create a new transaction|
|GET|/operator/wallets/{walletId}/addresses|Get all addresses of a wallet|
|POST|/operator/wallets/{walletId}/addresses|Create a new address|
|GET|/operator/wallets/{walletId}/addresses/{addressId}/balance|Get the balance of a given address and wallet|

##### Node

|Method|URL|Description|
|------|---|-----------|
|GET|/node/peers|Get all peers connected to node|
|POST|/node/peers|Connects a new peer to node|
|GET|/node/transactions/{transactionId}/confirmations|Get how many confirmations a block has|

##### Miner

|Method|URL|Description|
|------|---|-----------|
|POST|/miner/mine|Mine a new block|

#### Cadena de bloques - Blockchain

La cadena de bloques guarda los trozos de información, una lista de bloques y una lista de transacciones (un hash map). 

Es la responsable de:
* Verificar los bloques que llegan;
* Verificación de las transacciones que llegan;
* Sincronización de la lista de transacciones;
* Sincronización de la lista de bloques;

La cadena de bloques es una lista enlazada en donde el hash del siguiente bloque está calculado en función del hash del bloque previo añadiéndole los datos de ese bloque último.

```
+-----------+                +-----------+                +-----------+
|           |  Hashprevio    |           |  Hashprevio    |           |
|  Bloque 0 <----------------+  Bloque 1 <----------------+  Bloque N |
|           |                |           |                |           |
+-----------+                +-----------+                +-----------+
```
Un bloque se añade a la cadena de bloques sí y solo sí: 
1. Si el bloque es el último (índice previo + 1);
2. Si el bloque previo es correcto (si el has previo == block.previousHash);
3. El hash es correcto (el hash calculado del bloque == block.hash); 
4. El nivel de dificultad del reto prueba de trabajo es correcto (la dificultad en el índice de la cadena N < la dificultad del bloque que se quiere añadir)
5. Todas las transacciones en ese bloque son válidas;
6. La suma de la salida de las transacciones es igual a la suma de las transacciones de entrada + 50 monedas representando la recompensa por haber minado el bloque;
7. Si existe solo 1 comisión de la transacción y 1 recompensa de transacción. 

Una transacción de un bloque es válida si y solo si: 
1. Si el hash de la transacción es correcto (hash calculado de la transacción == transaction.hash);
2. La firma de todas las transacciones de entrada son correctas (los datos de la transacción están firmados por la clave pública de la dirección); 
3. Lasuma de las transacciones de entrada es mayor que la suma de las transacciones de salida, incluyendo que tiene que dejar algo de hueco o espacio para cubrir con las comisiones (fees);
4. Si la transacción no existe ya en la blockchain
5. Si todas las transacciones de entrada están sin gastar en la cadena de bloques.

Puedes leer este [post](https://medium.com/@lhartikk/a-blockchain-in-200-lines-of-code-963cc1cc0e54#.dttbm9afr5) from [naivechain](https://github.com/lhartikk/naivechain) para conocer más detalles de cómo funciona una cadena de bloques.

Existe una lista de transacciones pendientes. No existe nada especial al respecto. En esta implementación, la lista de transacciones contiene únicamente las transacciones pendientes. En cuanto una transacción está confirmada, la cadena de bloques eliminará la transacción de la lista. 


```
[
    transaction 1,
    transaction 2,
    transaction 3
]
```

A transaction is added to the transaction list:
1. If it's not already in the transaction list;
2. If the transaction hash is correct (calculated transaction hash == transaction.hash);
3. The signature of all input transactions are correct (transaction data is signed by the public key of the address);
4. The sum of input transactions are greater than output transactions, it needs to leave some room for the transaction fee;
5. If the transaction isn't already in the blockchain
6. If all input transactions are unspent in the blockchain;

##### Block structure

A block represents a group of transactions and contains information that links it to the previous block.

```javascript
{ // Block
    "index": 0, // (first block: 0)
    "previousHash": "0", // (hash of previous block, first block is 0) (64 bytes)
    "timestamp": 1465154705, // number of seconds since January 1, 1970
    "nonce": 0, // nonce used to identify the proof-of-work step.
    "transactions": [ // list of transactions inside the block
        { // transaction 0
            "id": "63ec3ac02f...8d5ebc6dba", // random id (64 bytes)
            "hash": "563b8aa350...3eecfbd26b", // hash taken from the contents of the transaction: sha256 (id + data) (64 bytes)
            "type": "regular", // transaction type (regular, fee, reward)
            "data": {
                "inputs": [], // list of input transactions
                "outputs": [] // list of output transactions
            }
        }
    ],
    "hash": "c4e0b8df46...199754d1ed" // hash taken from the contents of the block: sha256 (index + previousHash + timestamp + nonce + transactions) (64 bytes)
}
```

The details about the nonce and the proof-of-work algorithm used to generate the block will be described somewhere ahead.

##### Transaction structure

A transaction contains a list of inputs and outputs representing a transfer of coins between the coin owner and an address. The input list contains a list of existing unspent output transactions and it is signed by the address owner. The output list contains amounts to other addresses, including or not a change to the owner address.

```javascript
{ // Transaction
    "id": "84286bba8d...7477efdae1", // random id (64 bytes)
    "hash": "f697d4ae63...c1e85f0ac3", // hash taken from the contents of the transaction: sha256 (id + data) (64 bytes)
    "type": "regular", // transaction type (regular, fee, reward)
    "data": {
        "inputs": [ // Transaction inputs
            {
                "transaction": "9e765ad30c...e908b32f0c", // transaction hash taken from a previous unspent transaction output (64 bytes)
                "index": "0", // index of the transaction taken from a previous unspent transaction output
                "amount": 5000000000, // amount of satoshis
                "address": "dda3ce5aa5...b409bf3fdc", // from address (64 bytes)
                "signature": "27d911cac0...6486adbf05" // transaction input hash: sha256 (transaction + index + amount + address) signed with owner address's secret key (128 bytes)
            }
        ],
        "outputs": [ // Transaction outputs
            {
                "amount": 10000, // amount of satoshis
                "address": "4f8293356d...b53e8c5b25" // to address (64 bytes)
            },
            {
                "amount": 4999989999, // amount of satoshis
                "address": "dda3ce5aa5...b409bf3fdc" // change address (64 bytes)
            }
        ]
    }
}
```

#### Operator

The operator handles wallets and addresses as well the transaction creation. Most of its operation are CRUD related. Each operator has its list of wallets and addresses, meaning that they aren't synchronized between nodes.

##### Wallet structure

A wallet contains a random id number, the password hash and the secret generated from that password. It contains a list of key pairs each one representing an address.

```javascript
[
    { // Wallet
        "id": "884d3e0407...f29af094fd", // random id (64 bytes)
        "passwordHash": "5ba9151d1c...1424be8e2c", // hash taken from password: sha256 (password) (64 bytes)
        "secret": "6acb83e364...c1a04b6ee6", // pbkdf2 secret taken from password hash: sha512 (salt + passwordHash + random factor)
        "keyPairs": [
            {
                "index": 1,
                "secretKey": "6acb83e364...ee6bcdbc73", // EdDSA secret key generated from the secret (1024 bytes)
                "publicKey": "dda3ce5aa5...b409bf3fdc" // EdDSA public key generated from the secret (64 bytes) (also known as address)
            },
            {
                "index": 2,
                "secretKey": "072ab010ed...246ed16d26", // EdDSA secret key generated from pbkdf2 (sha512 (salt + passwordHash + random factor)) over last address secret key (1024 bytes)
                "publicKey": "4f8293356d...b53e8c5b25"  // EdDSA public key generated from the secret (64 bytes) (also known as address)
            }     
        ]
    }
]
```

##### Address structure

The address is created in a deterministic way, meaning that for a given password, the next address is created based on the previous address (or the password secret if it's the first address).

It uses the EdDSA algorithm to generate a secret public key pair using a seed that can come from a random generated value from the password hash (also in a deterministic way) or from the last secret key.

```javascript
{ // Address
    "index": 1,
    "secretKey": "6acb83e364...ee6bcdbc73", // EdDSA secret key generated from the secret (1024 bytes)
    "publicKey": "dda3ce5aa5...b409bf3fdc" // EdDSA public key generated from the secret (64 bytes) (also known as address)
},
```

Only the public key is exposed as the user's address.

#### Miner

The Miner gets the list of pending transactions and creates a new block containing the transactions. By configuration, every block has at most 2 transactions in it.

Assembling a new block:
1. Get the last two transactions from the list of pending transactions;
2. Add a new transaction containing the fee value to the miner's address, 1 satoshi per transaction;
3. Add a reward transaction containing 50 coins to the miner's address;
4. Prove work for this block;

##### Proof-of-work

The proof-of-work is done by calculating the 14 first hex values for a given transaction hash and increases the nonce until it reaches the minimal difficulty level required. The difficulty increases by an exponential value (power of 5) every 5 blocks created. Around the 70th block created it starts to spend around 50 seconds to generate a new block with this configuration. All these values can be tweaked.

```javascript
const difficulty = this.blockchain.getDifficulty();
do {
    block.timestamp = new Date().getTime() / 1000;
    block.nonce++;
    block.hash = block.toHash();
    blockDifficulty = block.getDifficulty();
} while (blockDifficulty >= difficulty);
```

The `this.blockchain.getDifficulty()` returns the hex value of the current blockchain's index difficulty. This value is calculated by powering the initial difficulty by 5 every 5 blocks.

The `block.getDifficulty()` returns the hex value of the first 14 bytes of block's hash and compares it to the currently accepted difficulty. 

When the hash generated reaches the desired difficulty level, it returns the block as it is.

#### Node

The node contains a list of connected peers and does all the data exchange between nodes, including:
1. Receive new peers and check what to do with it
1. Receive new blocks and check what to do with it
2. Receive new transactions and check what to do with it

The node rebroadcasts every information it receives unless it doesn't do anything with it, for example, if it already has the peer/transaction/blockchain.

An extra responsibility is to get a number of confirmations for a given transaction. It does that by asking every node if it has that transaction in its blockchain.

### Quick start

```sh
# Run a node
$ node bin/naivecoin.js

# Run two nodes
$ node bin/naivecoin.js -p 3001 --name 1
$ node bin/naivecoin.js -p 3002 --name 2 --peers http://localhost:3001

# Access the swagger API
http://localhost:3001/api-docs/
```

#### Example (wallet, address, transaction and mining)
```sh
# Create a wallet using password 't t t t t' (5 words)
$ curl -X POST --header 'Content-Type: application/json' -d '{ "password": "t t t t t" }' 'http://localhost:3001/operator/wallets'
{"id":"a2fb4d3f93ea3d4624243c03f507295c0c7cb5b78291a651e5575dcd03dfeeeb","addresses":[]}

# Create two addresses for the wallet created (replace walletId)
$ curl -X POST --header 'Content-Type: application/json' --header 'password: t t t t t' 'http://localhost:3001/operator/wallets/a2fb4d3f93ea3d4624243c03f507295c0c7cb5b78291a651e5575dcd03dfeeeb/addresses'
{"address":"e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"}

$ curl -X POST --header 'Content-Type: application/json' --header 'password: t t t t t' 'http://localhost:3001/operator/wallets/a2fb4d3f93ea3d4624243c03f507295c0c7cb5b78291a651e5575dcd03dfeeeb/addresses'
{"address":"c3c96504e432e35caa94c30034e70994663988ab80f94e4b526829c99958afa8"}

# Mine a block to the address 1 so we can have some coins
$ curl -X POST --header 'Content-Type: application/json' -d '{ "rewardAddress": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c" }' 'http://localhost:3001/miner/mine'
{
    "index": 1,
    "nonce": 1,
    "previousHash": "c4e0b8df46ce5cb2bcb0379ab0840228536cf4cd489783532a7c9d199754d1ed",
    "timestamp": 1493475731.692,
    "transactions": [
        {
            "id": "ab872b412afe62a087f3a8c354a27377f5fda33d7c98a1db3b1b0985801a6784",
            "hash": "423bae0bd2f4782f34c770df5be21f856b468a45bf88bb146da8ec2fe0fd3d21",
            "type": "reward",
            "data": {
                "inputs": [],
                "outputs": [
                    {
                        "amount": 5000000000,
                        "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
                    }
                ]
            }
        }
    ],
    "hash": "0311a3a89198ccf888c76337cc190e2db238b67a7db0d5062aac97d14fb679b4"
}

# Create a transaction that transfer 1000000000 satoshis from address 1 to address 2
$ curl -X POST --header 'Content-Type: application/json' --header 'password: t t t t t' -d '{ "fromAddress": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c", "toAddress": "c3c96504e432e35caa94c30034e70994663988ab80f94e4b526829c99958afa8", "amount": 1000000000, "changeAddress": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c" }' 'http://localhost:3001/operator/wallets/a2fb4d3f93ea3d4624243c03f507295c0c7cb5b78291a651e5575dcd03dfeeeb/transactions'
 {
  "id": "c3c1e6fbff949042b065dc9e22d065a54ab826595fd8877d2be8ddb8cbb0e27f",
  "hash": "3b5bbf698031e437787fe7b31f098e214a1eeff01fee9b95c22bccf20146982c",
  "type": "regular",
  "data": {
    "inputs": [
      {
        "transaction": "ab872b412afe62a087f3a8c354a27377f5fda33d7c98a1db3b1b0985801a6784",
        "index": "0",
        "amount": 5000000000,
        "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c",
        "signature": "4500f432d6b400811d83364224ce62bccd042ad92299118c0672bc5bc1390ffdfdbef135f36927d8bd77843f3a0b868d9ed3a5346dcbeda6c06f33876cfae00d"
      }
    ],
    "outputs": [
      {
        "amount": 1000000000,
        "address": "c3c96504e432e35caa94c30034e70994663988ab80f94e4b526829c99958afa8"
      },
      {
        "amount": 3999999999,
        "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
      }
    ]
  }
}

# Mine a new block containing that transaction
$ curl -X POST --header 'Content-Type: application/json' -d '{ "rewardAddress": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c" }' 'http://localhost:3001/miner/mine'
{
  "index": 2,
  "nonce": 6,
  "previousHash": "0311a3a89198ccf888c76337cc190e2db238b67a7db0d5062aac97d14fb679b4",
  "timestamp": 1493475953.226,
  "transactions": [
    {
      "id": "c3c1e6fbff949042b065dc9e22d065a54ab826595fd8877d2be8ddb8cbb0e27f",
      "hash": "3b5bbf698031e437787fe7b31f098e214a1eeff01fee9b95c22bccf20146982c",
      "type": "regular",
      "data": {
        "inputs": [
          {
            "transaction": "ab872b412afe62a087f3a8c354a27377f5fda33d7c98a1db3b1b0985801a6784",
            "index": "0",
            "amount": 5000000000,
            "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c",
            "signature": "4500f432d6b400811d83364224ce62bccd042ad92299118c0672bc5bc1390ffdfdbef135f36927d8bd77843f3a0b868d9ed3a5346dcbeda6c06f33876cfae00d"
          }
        ],
        "outputs": [
          {
            "amount": 1000000000,
            "address": "c3c96504e432e35caa94c30034e70994663988ab80f94e4b526829c99958afa8"
          },
          {
            "amount": 3999999999,
            "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
          }
        ]
      }
    },
    {
      "id": "6b55b1e85369743f360edd5bedc3467eba81b35c2b88490686eee90946231dd6",
      "hash": "86f5b4a40c027e1ef7e060dd9b9ab7ae48258f3a43dfc19d9d8111c396463b8c",
      "type": "fee",
      "data": {
        "inputs": [],
        "outputs": [
          {
            "amount": 1,
            "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
          }
        ]
      }
    },
    {
      "id": "0f6f6c04602ac1bea15157a1a86978d46488a7865fa3db3bfc581a1407950599",
      "hash": "f9fa281fbf9ffd3d63dd0c3503588fe3010dd6740a4a960b98d1be4aa1fa7a05",
      "type": "reward",
      "data": {
        "inputs": [],
        "outputs": [
          {
            "amount": 5000000000,
            "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
          }
        ]
      }
    }
  ],
  "hash": "08861fc4864ba0bf7a899db9ffaaa39376ad3857b1115951db074e3d06f93a5f"
}

# Check how many confirmations that transaction has.
$ curl -X GET 'http://localhost:3001/node/transactions/c3c1e6fbff949042b065dc9e22d065a54ab826595fd8877d2be8ddb8cbb0e27f/confirmations'
{"confirmations":1}

# Get address 1 balance
$ curl -X GET 'http://localhost:3001/operator/wallets/a2fb4d3f93ea3d4624243c03f507295c0c7cb5b78291a651e5575dcd03dfeeeb/addresses/e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c/balance'
{"balance":9000000000}

# Get address 2 balance
$ curl -X GET 'http://localhost:3001/operator/wallets/a2fb4d3f93ea3d4624243c03f507295c0c7cb5b78291a651e5575dcd03dfeeeb/addresses/c574de33acfd82f2146d2f45f37ce95b7bdca133b8ad310adbd46938c75992c8/balance'
{"balance":1000000000}

# Get unspent transactions for address 1
$ curl -X GET 'http://localhost:3001/blockchain/transactions/unspent?address=e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c'
[
  {
    "transaction": "c3c1e6fbff949042b065dc9e22d065a54ab826595fd8877d2be8ddb8cbb0e27f",
    "index": "1",
    "amount": 3999999999,
    "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
  },
  {
    "transaction": "6b55b1e85369743f360edd5bedc3467eba81b35c2b88490686eee90946231dd6",
    "index": "0",
    "amount": 1,
    "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
  },
  {
    "transaction": "0f6f6c04602ac1bea15157a1a86978d46488a7865fa3db3bfc581a1407950599",
    "index": "0",
    "amount": 5000000000,
    "address": "e155df3a1bac05f88321b73931b48b54ea4300be9d1225e0b62638f537e5544c"
  }
]
```

#### Docker

```sh
# Build the image
$ docker build . -t naivecoin

# Run naivecoin in a docker
$ ./dockerExec.sh

# Run naivecoin in a docker using port 3002
$ ./dockerExec.sh -p 3002

# Run naivecoin in a docker options
$ ./dockerExec.sh -h
Usage: ./dockerExec.sh -a HOST -p PORT -l LOG_LEVEL -e PEERS -n NAME

# Run docker-compose with 3 nodes
$ docker-compose up
```

### Client

```sh
# Command-line options
$ node bin/naivecoin.js -h
Usage: bin\naivecoin.js [options]

Options:
  -a, --host       Host address. (localhost by default)
  -p, --port       HTTP port. (3001 by default)
  -l, --log-level  Log level (7=dir, debug, time and trace, 6=log and info,
                   4=warn, 3=error, assert, 6 by default).
  --peers          Peers list.                                           [array]
  --name           Node name/identifier.
  -h, --help       Show help                                           [boolean]
```

### Development

```sh
# Clonning repository
$ git clone git@github.com:conradoqg/naivecoin.git
$ cd naivecoin
$ npm install

# Testing
$ npm test
```

### Contribution and License Agreement

If this implementation does something wrong, please feel free to contribute by opening an issue or sending a PR. The main goal of this project is not to create a full-featured cryptocurrency, but a good example of how it works.

If you contribute code to this project, you are implicitly allowing your code
to be distributed under the Apache 2.0 license. You are also implicitly verifying that
all code is your original work.

[![Twitter](https://img.shields.io/twitter/url/https/github.com/conradoqg/naivecoin.svg?style=social)](https://twitter.com/intent/tweet?text=Check%20it%20out%3A%20Naivecoin%20-%20a%20cryptocurrency%20implementation%20in%20less%20than%201500%20lines%20of%20code&url=%5Bobject%20Object%5D)

[![GitHub license](https://img.shields.io/badge/license-Apache%202-blue.svg)](https://raw.githubusercontent.com/conradoqg/naivecoin/master/LICENSE)
