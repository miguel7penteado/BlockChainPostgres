# BlockChainPostgres

ferramenta
```
wget https://storage.googleapis.com/provendb-prd/proofable-cli/proofable-cli_linux_amd64.tar.gz
```

autenticar
```bash
proofable-cli auth
```

```bash
npm i proofable
``` 

```SQL
CREATE TABLE ProofableAnchors(
  trieId      VARCHAR(256) PRIMARY KEY,
  trie        BYTEA NOT NULL,
  tableName   VARCHAR(1000) NOT NULL,
  keyName     VARCHAR(1000) NOT NULL,
  minKey      BIGINT NOT NULL,
  maxKey      BIGINT NOT NULL,
  create_date TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT current_timestamp
);
``` 
`trie` é a "chave forasteira" entre o postgres e o blockchain
PG N<->1 Blockchain

```javascript
    // obter dados não ancorados
    const data = await getDBData(tableName, keyName, await getMaxKey(tableName));
    if (data.keyValues.length === 0)
    {
      console.log('Nenhum dado para ancorar no momento');
    } 
    else
    { 
      // obter a ancora do blockchain e selecionar blockchain
      // Ethereum
      // Bitcoin
      // Elastos
      // GoChain 
      const anchoredTrie = await anchorData(data, 'HEDERA');
      // salvar ancora na tabela
      await saveTrieToDB(anchoredTrie.getTrieId(), tableName, keyName, data.minKey, data.maxKey);
    }
``` 

```javascript
    const trieId = await getLastTrie(tableName);
    const trieDB = await getTrieFromDB(trieId);
    const dbData = await getDBData(trieDB.tableName, trieDB.keyName, trieDB.minKey, trieDB.maxKey);
    const validatedTrie = await validateTrie(dbData, trieDB.trie);
    console.log('validatedTrie', validatedTrie);
``` 

trabalhar exemplo acima com python.

```javascript

/* eslint no-console:off */
/* eslint no-use-before-define:off */
/* eslint no-trailing-spaces:off */
/* eslint padded-blocks:off */

const fs = require('fs');
const tmp = require('tmp');

const assert = require('assert');
const crypto = require('crypto');
const stringify = require('json-stringify-safe');

const proofable = require('proofable');
const { Client } = require('pg');

const proofableClient = proofable.newAPIClient('api.dev.proofable.io:443'); 
const pgClient = new Client();

async function main() {
  try {

    const tableName = 'contracts';
    const keyName = 'contractid';
    const blockchainType = 'HEDERA_MAINNET'; // or 'ETH', 'HEDERA', 'GOCHAIN', 'BTC', etc
    await pgClient.connect();
    
    // await performanceTest(1000000, tableName, keyName, blockchainType);

    addRows(1000);
    // Get latest data from database and anchor to Blockchain
    const maxExistingKey = Number(await getMaxKey(tableName));
    const data = await getDBData(tableName, keyName, maxExistingKey + 1);
    if (Object.keys(data.keyValues).length === 0) {
      console.log('No new data to anchor');
    } else {
      console.log(data);
      const anchoredTrie = await anchorData(data, blockchainType);
      // Save this trie to the database
      await saveTrieToDB(anchoredTrie.getTrieId(), tableName, keyName, data.minKey, data.maxKey);
    }
 
    // Validate the trie we just created 
    let trieId = await getLastTrie(tableName);
    let trieDB = await getTrieFromDB(trieId);
    let dbData = await getDBData(trieDB.tableName, trieDB.keyName, 
      trieDB.minKey, trieDB.maxKey);
    let validatedTrie = await validateTrie(dbData, trieDB.trie);
    console.log('validatedTrie', validatedTrie);
  
    await tamperData(tableName, keyName);
    trieId = await getLastTrie(tableName);
    trieDB = await getTrieFromDB(trieId);
    dbData = await getDBData(trieDB.tableName, trieDB.keyName, 
      trieDB.minKey, trieDB.maxKey);
    validatedTrie = await validateTrie(dbData, trieDB.trie);
    console.log('validatedTrie', validatedTrie);

  } catch (error) {
    console.log(error.stack);
  }
  process.exit(0);
}

async function performanceTest(testRowCount, tableName, keyName, blockchainType) {
  addRows(testRowCount);
  const start = new Date();
  const maxExistingKey = Number(await getMaxKey(tableName));
  const data = await getDBData(tableName, keyName, maxExistingKey + 1);
  const anchoredTrie = await anchorData(data, blockchainType);
  const elapsedTime = (new Date()) - start;
  console.log(testRowCount, 'rows anchored in', elapsedTime, 'ms');
  console.log(testRowCount * 1000 / elapsedTime, 'rows/sec');
  return { maxExistingKey, data };
}

// Anchor data to a blockchain - create a trie and anchor that trie
async function anchorData(data, anchorChainType) {
  console.log('--> Anchoring data to ', anchorChainType);
 
  const inputData = await proofable.dataToKeyValues(data.keyValues);
  const trie = await proofableClient.createTrieFromKeyValues(inputData);
  await new Promise((resolve) => setTimeout(resolve, 2000));
  const anchoredTrie = await proofableClient.anchorTrie(trie, 
    proofable.Anchor.Type[anchorChainType]);
  console.log('anchoredTrie->', anchoredTrie);

  return anchoredTrie;
}

// Validate a trie against some input data
async function validateTrie(dbData, trie) {
  console.log('--> Validating trie ');
  await new Promise((resolve) => setTimeout(resolve, 1000));
  const newSortedData = await proofable.sortKeyValues(proofable.dataToKeyValues(dbData.keyValues));
  const validatedProof = await proofableClient.verifyTrieWithSortedKeyValues(trie,
    newSortedData);
  return validatedProof;
}

// Save a trie to the ProofableAnchors table (see the postresTables.sql file for table definition)
async function saveTrieToDB(trieId, tableName, keyName, minKey, maxKey) {
  console.log(`--> saving trie ${trieId} to db`);
  const tmpFile = tmp.fileSync();
  const trieFileName = tmpFile.name; // `${trieId}.tmp1`;
  // const trieFileName = `${Math.random() * 1000}.save.trie`;
  await proofableClient.exportTrie(trieId, trieFileName);
  // await new Promise((resolve, reject) => { setTimeout(resolve, 1000); }); // Workaround
  const trieData = fs.readFileSync(trieFileName, 'base64');

  const insOut = await pgClient.query(
    `INSERT INTO ProofableAnchors (trieId, trie, tableName, keyName, minKey,maxKey ) 
     VALUES($1,$2 , $3, $4, $5, $6)`,
    [trieId, trieData, tableName, keyName, minKey, maxKey],
  );
  console.log(`${insOut.rowCount} rows inserted`);
}

// Retrieve a trie from the database
async function getTrieFromDB(trieId) {
  console.log(`--> Getting ${trieId} from DB`);

  const data = await pgClient.query(`select * 
                                       from ProofableAnchors 
                                      where trieid=$1`, [trieId]);
  const trieDataOut = data.rows[0].trie;
  const tableName = data.rows[0].tablename;
  const minKey = data.rows[0].minkey;
  const maxKey = data.rows[0].maxkey;
  const keyName = data.rows[0].keyname;

  const trie = await parseTrie(trieDataOut);
  return ({
    trie,
    tableName,
    keyName,
    minKey,
    maxKey,
  });
}

// Get a proof for a specific row in the input data
async function getRowProof(tableName, keyValue) {
  const tmpFile = tmp.fileSync().name;
  const trie = await getTrieForKey(tableName, keyValue);
  const proofableKey = proofable.Key.from(keyValue.toString());
  const proofableKeyValuesFilter = proofable.KeyValuesFilter.from([proofableKey]);
  await proofableClient.createKeyValuesProof(trie.getId(), '',
    proofableKeyValuesFilter, tmpFile);

  // await new Promise((resolve) => setTimeout(resolve, 1000));
  const docProof = fs.readFileSync(tmpFile);
  // verifyKeyValuesProof();
  return (docProof);
 
}

// retrieve a trie corresponding to a particular table primary key value
async function getTrieForKey(tableName, keyValue) {
  console.log('--> Getting trie for table', tableName, 'key', keyValue);
  const data = await pgClient.query(
    `SELECT  * FROM ProofableAnchors
     WHERE $1 >= minKey AND $1 <=maxKey AND tableName=$2;`,
    [keyValue, tableName],
  );
  console.log(data.rows.length, 'rows returned');
  const trieDataOut = data.rows[0].trie;
  return (parseTrie(trieDataOut));
}

// Take raw trie data and import it into a proper trie
async function parseTrie(trieData) {
  const tmpFile = tmp.fileSync();
  const trieFileName = tmpFile.name;
  fs.writeFileSync(trieFileName, trieData.toString(), {
    encoding: 'base64',
  });
  const trie = await proofableClient.importTrie('', trieFileName);
  return trie;
}

async function getLastTrie(tableName) {
  console.log('--> Getting last trie for ', tableName);
  const data = await pgClient.query(
    `SELECT trieid FROM proofableAnchors
             WHERE tableName=$1
             ORDER BY create_date DESC
            LIMIT 1`, [tableName],
  );
  const trieId = data.rows[0].trieid;
  return trieId;
}

async function getMaxKey(tableName) {
  let maxKey = 0;
  const data = await pgClient.query(`select max(maxKey)
                                       from ProofableAnchors 
                                      where tableName=$1`, [tableName]);
  if (data.rows[0].max != null) {
    maxKey = data.rows[0].max;
  }
  return (maxKey);
}

async function addRows(nRows) {
  await pgClient.query('call populateContracts($1)', [nRows]);
}

async function tamperData(tableName, keyName) {
  const query = `UPDATE ${tableName} 
            SET metadata='{}' 
         WHERE ${keyName}=(SELECT MAX(${keyName}) FROM ${tableName})`;
  const data = await pgClient.query(query);
  console.log(data.rowCount, ' row updated');
}

async function getDBData(tableName, keyName, minValue, maxValue = 2 ** 32) {
  // For this example, we assume the key is a monotonically increasing number
  // More complex logic required otherwise
  const debug = false;
  console.log('Getting data for ', tableName, keyName, minValue, maxValue);
  const keyValues = {};
  let maxKey;
  let minKey;
  const query = `SELECT * FROM ${tableName} 
                  WHERE ${keyName} >= $1 
                    AND ${keyName} <= $2`;

  const data = await pgClient.query(query, [minValue, maxValue]);
  if (debug) console.log(data.rows.length, 'rows returned');

  data.rows.forEach((row) => {

    assert.equal((keyName in row), true, 'Table row does not contain key');

    const key = Number(row[keyName]); // Will fail if non-numeric
    if (maxKey == null || maxKey < key) maxKey = key;
    if (minKey == null || minKey > key) minKey = key;
    const hash = crypto.createHash('sha256').update(stringify(row)).digest('base64');
    keyValues[key] = hash;
  });
  return ({
    minKey,
    maxKey,
    keyValues,
  });
}

main();

``` 
