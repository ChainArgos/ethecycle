# Ethecycle - Architecture & System Documentation

Ethecycle is a blockchain wallet tagging and transaction graph database system. It aggregates wallet address metadata from 19+ external sources, loads blockchain transaction data into a Neo4j graph database, and enables forensic analysis, address clustering, and transaction flow tracing across 14+ blockchains.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Data Models](#data-models)
4. [Chain Address Database](#chain-address-database)
5. [Address Importers](#address-importers)
6. [Blockchain Support](#blockchain-support)
7. [Transaction Loading Pipeline](#transaction-loading-pipeline)
8. [Neo4j Graph Schema](#neo4j-graph-schema)
9. [Docker Architecture](#docker-architecture)
10. [CLI Reference](#cli-reference)
11. [Cypher Query Examples](#cypher-query-examples)
12. [Relationship to ChainArgos Ecosystem](#relationship-to-chainargos-ecosystem)

---

## System Overview

Ethecycle serves as the **graph intelligence layer** in the ChainArgos ecosystem. While `node-etl` extracts raw blockchain data and `airflow_blockchain_etl` orchestrates ETL workflows and wallet tagging at scale, Ethecycle provides:

- **Graph-based transaction analysis** via Neo4j for forensic investigations
- **Wallet metadata aggregation** from 19+ open-source and proprietary sources
- **Multi-chain address normalization** with blockchain-specific validation
- **Visual transaction flow tracing** via Neo4j Browser and Cypher queries

### Key Statistics

- **14+ blockchains** supported (Bitcoin, Ethereum, Tron, Polygon, Arbitrum, etc.)
- **19 address importers** pulling from GitHub repos, Google Sheets, APIs
- **30+ wallet categories** with color-coded terminal output
- **Neo4j bulk import** for fast data loading (not LOAD CSV)

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                      EXTERNAL DATA SOURCES                          │
├─────────┬──────────┬───────────┬──────────┬────────────┬────────────┤
│ GitHub  │ Google   │ Etherscan │ Dune     │ Trust      │ Manual     │
│ Repos   │ Sheets   │ Labels    │ Analytics│ Wallet     │ Curation   │
│ (7)     │ (1)      │ (3)       │ (2)      │ (1)        │ (3)        │
└────┬────┴────┬─────┴─────┬────┴────┬─────┴─────┬──────┴─────┬──────┘
     │         │           │         │           │            │
     ▼         ▼           ▼         ▼           ▼            ▼
┌──────────────────────────────────────────────────────────────────────┐
│              ADDRESS IMPORTERS (19 importers)                        │
│  CoinMarketCap | DeFi Llama | Etherscan Labels | CryptoScamDB |    │
│  Ethereum Lists | Trust Wallet | MyEtherWallet | OKX POR |         │
│  FTX Partners | Google Sheets | Hand Collated | Hardcoded | ...    │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│              chain_addresses.db (SQLite)                             │
│  Tables: wallets, tokens, data_sources                              │
│  UNIQUE(address, blockchain, data_source_id)                        │
│  Collision detection + data coalescing from multiple sources        │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               │  (lookup during transaction loading)
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│              TRANSACTION LOADING                                     │
│                                                                      │
│  Input: Pipe-delimited CSVs (ethereumetl format)                    │
│  token_address | from_address | to_address | value |                │
│  transaction_hash | log_index | block_number                        │
│                                                                      │
│  Processing:                                                         │
│  1. Parse CSV → Txn objects                                         │
│  2. Decimal adjustment by token                                     │
│  3. Wallet extraction (unique addresses)                            │
│  4. chain_addresses.db lookup for labels/categories                 │
│  5. Generate Neo4j-compatible CSVs                                  │
│  6. neo4j-admin database import (bulk load)                         │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│              NEO4J GRAPH DATABASE                                     │
│                                                                      │
│  Nodes:  (wallet) — address, blockchain, name, category             │
│  Edges:  [:transaction] — tx_id, symbol, num_tokens, block_number   │
│                                                                      │
│  Queries via: Neo4j Browser (port 7474) | Python driver (port 7687) │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Data Models

### Wallet (`ethecycle/models/wallet.py`)

Represents a blockchain address (node in graph):

| Property | Type | Description |
|---|---|---|
| `address` | str | Blockchain address (lowercase for EVM, preserved case for BTC/TRON) |
| `blockchain` | str | Chain identifier (`ethereum`, `bitcoin`, `tron`, etc.) |
| `name` | str | Human-readable label (max 250 chars) |
| `category` | str | Classification (CEX, DeFi, hackers, etc.) |
| `extracted_at` | datetime | When this wallet was processed |

**Wallet Categories** (30+):

| Category | Color | Description |
|---|---|---|
| `cex` | Dark Orange | Centralized exchanges |
| `cefi` | Dark Orange | Centralized finance (BlockFi, Celsius, Nexo) |
| `defi` | Green | Decentralized finance protocols |
| `dex` | Green | Decentralized exchanges |
| `bridge` | Blue | Cross-chain bridges |
| `dao` | Blue | DAOs |
| `hackers` | Red | Exploits, stolen funds |
| `scam` | Red | Known scam addresses |
| `mixer` | Yellow | Privacy mixers (Tornado Cash) |
| `mev` | Magenta | MEV searcher bots |
| `token` | Cyan | Token contract addresses |
| `stablecoin` | Cyan | Stablecoin issuers |
| `nft` | Cyan | NFT platforms |
| `oracle` | Green | Oracle services |
| `miner` | Blue | Mining pools |
| `gambling` | Magenta | Gambling platforms |
| `custody` | Dark Orange | Custodial services |
| `vc` | Green | Venture capital |
| `otc` | Dark Orange | OTC desks |
| `market-maker` | Green | Market makers |
| `wallet-provider` | Blue | Wallet services |
| `multisig` | Blue | Multi-signature wallets |

### Transaction (`ethecycle/models/transaction.py`)

Represents a token transfer (edge in graph):

| Property | Type | Description |
|---|---|---|
| `transaction_hash` | str | On-chain transaction hash |
| `log_index` | int | Event log index within the transaction |
| `transaction_id` | str | Unique ID: `{hash}-{log_index}` |
| `token_address` | str | Contract address of the transferred token |
| `from_address` | str | Sender address |
| `to_address` | str | Recipient address |
| `value` | int | Raw transfer value |
| `num_tokens` | float | Decimal-adjusted token amount |
| `symbol` | str | Token symbol (USDT, ETH, etc.) |
| `block_number` | int | Block containing this transaction |

### Token (`ethecycle/models/token.py`)

Represents a token contract:

| Property | Type | Description |
|---|---|---|
| `symbol` | str | Token ticker symbol |
| `name` | str | Full token name |
| `decimals` | int | Decimal places (default 0 — data is pre-adjusted) |
| `token_type` | str | ERC20, TRC20, etc. |
| `is_active` | bool | Whether token is actively traded |
| `is_scam` | bool | Whether token is flagged as scam |

---

## Chain Address Database

The `chain_addresses.db` SQLite database (`ethecycle/chain_addresses/address_db.py`) is the central wallet metadata store.

### Schema

```sql
-- Wallets table
CREATE TABLE wallets (
    address      TEXT NOT NULL,
    blockchain   TEXT NOT NULL,
    label        TEXT,
    category     TEXT,
    data_source_id INTEGER NOT NULL,
    UNIQUE(address, blockchain, data_source_id)
);

-- Tokens table
CREATE TABLE tokens (
    address      TEXT NOT NULL,
    blockchain   TEXT NOT NULL,
    symbol       TEXT,
    name         TEXT,
    decimals     INTEGER DEFAULT 0,
    token_type   TEXT,
    is_active    BOOLEAN,
    is_scam      BOOLEAN,
    data_source_id INTEGER NOT NULL,
    UNIQUE(address, blockchain, data_source_id)
);

-- Data sources table
CREATE TABLE data_sources (
    id           INTEGER PRIMARY KEY,
    name         TEXT NOT NULL,
    version      TEXT
);
```

### Data Coalescing Logic

When multiple sources tag the same address:
1. **Hand collated** and **hardcoded** sources have highest priority
2. Collisions are detected and logged with warnings
3. The system coalesces data, preferring the most complete record (non-null label, category)
4. `data_source_id` provides ordering for conflict resolution

### Build Process

The database is built during Docker image creation:
```bash
./import_chain_addresses.py ALL  # Imports from all 19 sources
```

Or rebuilt incrementally:
```bash
./import_chain_addresses.py etherscan_labels  # Import specific source
./import_chain_addresses.py RESET_DB          # Drop and recreate empty
```

---

## Address Importers

All importers live in `ethecycle/chain_addresses/importers/` and produce `Wallet` or `Token` objects for the SQLite database.

### GitHub Repository Importers

| Importer | Repository | Data Produced |
|---|---|---|
| `coin_market_cap_repo_importer` | CoinMarketCap assets | Token contract addresses across chains |
| `ethereum_lists_repo_importer` | ethereum-lists | Multi-chain contract metadata |
| `my_ether_wallet_repo_importer` | MyEtherWallet | MEW address book |
| `trust_wallet_assets_importer` | Trust Wallet assets | Token and project addresses |
| `cryptoscamdb_addresses_importer` | CryptoScamDB blacklist | Scam/fraud addresses |
| `etherscrape_importer` | Etherscrape | Scraped contract data |
| `w_mcdonald_etherscan_repo_importer` | wmcdonald/etherscan-labels | Additional Etherscan labels |

### Etherscan Label Importers

| Importer | Chains Covered | Description |
|---|---|---|
| `etherscan_labels_importer` | Ethereum, BSC, Polygon, Arbitrum, Avalanche, Fantom, Optimism | Labels from `brianleect/etherscan-labels` repo |
| `etherscan_contract_crawler_importer` | Ethereum | Crawled contract verification data |

### External API / Data Importers

| Importer | Source | Description |
|---|---|---|
| `defi_llama_importer` | DeFi Llama | Protocol TVL and contract addresses |
| `okx_proof_of_reserves_importer` | OKX exchange | POR wallet addresses |
| `google_sheets_importer` | Public Google Sheets | Crowd-sourced wallet labels |
| `wallets_from_dune_importer` | Dune Analytics exports | CEX/DeFi/hacker labels |
| `dune_copy_paste_file_reader` | Manual Dune data | Hand-copied Dune query results |

### Manual / Static Importers

| Importer | Description |
|---|---|
| `hand_collated_address_importer` | Manually curated highest-priority addresses |
| `hardcoded_addresses_importer` | Built-in address mappings (known contracts) |
| `lost_forever_addresses_importer` | Addresses that burned/lost funds permanently |
| `ftx_major_partners_importer` | Major FTX trading counterparties |
| `m_ranger_data_importer` | M-Ranger wallet tag dataset |

### Label Category Mapping

The `etherscan.py` config maps raw labels to standardized categories:

```python
# Examples of label → category mapping
'binance'      → 'cex'
'tornado cash' → 'mixer'
'aave'         → 'defi'
'uniswap'      → 'dex'
'wormhole'     → 'bridge'
'flashbots'    → 'mev'
'celsius'      → 'cefi'
'circle'       → 'stablecoin'
```

### Organization Mapping

The `organizations.py` config groups related entities:

```python
# FTX/Alameda entity consolidation
'alameda research'  → 'FTX'
'ftx exchange'      → 'FTX'
'ftx us'            → 'FTX'
'blockfolio'        → 'FTX'
```

---

## Blockchain Support

All blockchain implementations extend `ChainInfo` (`ethecycle/blockchains/chain_info.py`).

### Supported Chains

| Chain | Module | Address Prefix | Address Length | Case Sensitive | Scanner |
|---|---|---|---|---|---|
| Ethereum | `ethereum.py` | `0x` | 42 | No | Etherscan |
| Bitcoin | `bitcoin.py` | `1`/`3`/`bc1` | Variable | Yes | Blockstream |
| Tron | `tron.py` | `T` | 34 | Yes | Tronscan |
| Polygon | `polygon.py` | `0x` | 42 | No | Polygonscan |
| Arbitrum | `arbitrum.py` | `0x` | 42 | No | Arbiscan |
| Optimism | `optimism.py` | `0x` | 42 | No | Optimistic Etherscan |
| Avalanche C | `avalanche.py` | `0x` | 42 | No | Snowtrace |
| BSC | `binance_smart_chain.py` | `0x` | 42 | No | BscScan |
| Fantom | `fantom.py` | `0x` | 42 | No | FTMScan |
| Solana | `solana.py` | Base58 | 32-44 | Yes | Solscan |
| Cardano | `cardano.py` | `addr1` | Variable | Yes | CardanoScan |
| Ripple | `ripple.py` | `r` | Variable | Yes | XRP Explorer |
| Litecoin | `litecoin.py` | `L`/`M`/`ltc1` | Variable | Yes | Blockchair |
| Bitcoin Cash | `bitcoin_cash.py` | Various | Variable | Yes | Blockchair |
| Ronin | `ronin.py` | `ronin:` | 42+ | No | Ronin Explorer |

### Address Guessing

The `blockchain.py` module provides `guess_blockchain_from_address()` which infers the chain from address format:

```python
guess_blockchain_from_address('0x742d...') → 'ethereum'
guess_blockchain_from_address('TN3W4H...') → 'tron'
guess_blockchain_from_address('1A1zP1...') → 'bitcoin'
```

---

## Transaction Loading Pipeline

### Input Format

Pipe-delimited CSV (ethereumetl-compatible), no headers:

```
token_address|from_address|to_address|value|transaction_hash|log_index|block_number
```

### Pipeline Steps

1. **Parse CSV** — `Txn.extract_from_csv()` reads each row into `Txn` objects
2. **Token Lookup** — Resolves `token_address` to symbol/decimals from `chain_addresses.db`
3. **Decimal Adjustment** — Divides raw `value` by `10^decimals` to get human-readable amount
4. **Wallet Extraction** — `Wallet.extract_wallets_from_transactions()` collects unique addresses
5. **Metadata Enrichment** — Looks up each wallet in `chain_addresses.db` for labels/categories
6. **CSV Generation** — Writes Neo4j-compatible wallet and relationship CSVs to `output/`
7. **Bulk Import** — Runs `neo4j-admin database import` for fast loading

### Output CSVs

| File | Contents | Key Columns |
|---|---|---|
| `Wallet_{timestamp}.csv` | Unique addresses | `address:ID`, `blockchain`, `name`, `category` |
| `Relationship_{timestamp}.csv` | Token transfers | `transaction_id`, `symbol`, `num_tokens:double`, `block_number:int` |

---

## Neo4j Graph Schema

### Nodes

**Label:** `wallet`

| Property | Type | Description |
|---|---|---|
| `address` | String (ID) | Blockchain address |
| `blockchain` | String | Chain identifier |
| `name` | String | Wallet label from chain_addresses.db |
| `category` | String | Wallet category |
| `extracted_at` | DateTime | Processing timestamp |

### Relationships

**Type:** `transaction`

| Property | Type | Description |
|---|---|---|
| `transaction_id` | String | `{hash}-{log_index}` |
| `blockchain` | String | Chain identifier |
| `token_address` | String | Token contract address |
| `symbol` | String | Token ticker |
| `num_tokens` | Double | Decimal-adjusted amount |
| `block_number` | Integer | Block number |
| `extracted_at` | DateTime | Processing timestamp |

### Indexes

```cypher
CREATE INDEX wallet_address FOR (w:wallet) ON (w.address);
CREATE INDEX wallet_blockchain FOR (w:wallet) ON (w.blockchain);
CREATE INDEX txn_symbol FOR ()-[t:transaction]-() ON (t.symbol);
CREATE INDEX txn_block FOR ()-[t:transaction]-() ON (t.block_number);
```

---

## Docker Architecture

### Services (docker-compose.yml)

| Service | Image | Ports | Purpose |
|---|---|---|---|
| `neo4j` | neo4j:5.1.0-community | 7474 (browser), 7687 (bolt) | Graph database |
| `python_etl` | Custom (Dockerfile) | — | ETL processing container |

### Environment Variables (.env)

| Variable | Description |
|---|---|
| `TXION_DATA_DIR` | Path to transaction CSV directory |
| `REBUILD_CHAIN_ADDRESS_DB` | `freshly_built_address_db` or `copy_prebuilt_address_db` |
| `NEO4J_DATA_DIR` | Neo4j data directory (default: `$PWD/data/neo4j/`) |
| `NEO4J_USER_AND_PASS` | Neo4j credentials (default: `neo4j/neo4j_password`) |

### Build Process

1. `container_file_setup.sh` generates SSH keys for inter-container communication
2. Docker build runs `import_chain_addresses.py ALL` to bake wallet DB into image
3. Neo4j container initialized with JVM settings from `generate_.neo4j.env_file.sh`

---

## CLI Reference

### load_transactions.py

```bash
# Load transactions (first run requires --drop)
./load_transactions.py /path/to/txns.csv --drop

# Load directory of CSVs
./load_transactions.py /path/to/csv_dir/ --drop

# Filter by token
./load_transactions.py /path/to/txns.csv --token USDT --drop

# Extract and transform only (no load)
./load_transactions.py /path/to/txns.csv --extract-only

# List available token symbols
./load_transactions.py --list-token-symbols
```

| Flag | Description |
|---|---|
| `-b, --blockchain` | Blockchain (default: `ethereum`) |
| `-t, --token` | Filter by token symbol |
| `-d, --drop` | Drop and recreate database |
| `-e, --extract-only` | Extract/transform without loading |
| `-p, --preserve-csvs` | Keep generated CSVs after load |
| `-D, --debug` | Debug output |

### import_chain_addresses.py

```bash
./import_chain_addresses.py ALL              # Rebuild from all sources
./import_chain_addresses.py RESET_DB         # Drop and recreate empty DB
./import_chain_addresses.py etherscan_labels # Import specific source
```

### Container Commands

| Command | Description |
|---|---|
| `show_chain_addresses` | Display wallet tags |
| `show_tokens` | Display known tokens |
| `chain_address_db` | Connect to SQLite DB |
| `dune_query` | Print Dune-compatible address query |
| `bpython` | Launch Python REPL with project loaded |

---

## Cypher Query Examples

### Find All Transactions for a Wallet

```cypher
MATCH (w:wallet {address: '0x742d35cc6634c0532925a3b844bc9e7595916da2'})
      -[t:transaction]-(other:wallet)
RETURN w.name, t.symbol, t.num_tokens, other.address, other.name
ORDER BY t.block_number
```

### Find Largest USDT Transfers

```cypher
MATCH (from:wallet)-[t:transaction {symbol: 'USDT'}]->(to:wallet)
WHERE t.num_tokens > 1000000
RETURN from.name, t.num_tokens, to.name, t.block_number
ORDER BY t.num_tokens DESC
LIMIT 50
```

### Trace Fund Flow (2 Hops)

```cypher
MATCH path = (source:wallet)-[:transaction*1..2]->(dest:wallet)
WHERE source.address = '0x...'
  AND dest.category = 'cex'
RETURN path
```

### Find Super Nodes (Highly Connected Wallets)

```cypher
MATCH (w:wallet)-[t:transaction]-()
WITH w, COUNT(t) AS tx_count
WHERE tx_count > 10000
RETURN w.address, w.name, w.category, tx_count
ORDER BY tx_count DESC
```

### Peeling Chain Detection

```cypher
MATCH path = (start:wallet)-[:transaction*3..10]->(end:wallet)
WHERE start.address = '0x...'
  AND ALL(r IN relationships(path) WHERE r.num_tokens > 100)
RETURN path
```

---

## Relationship to ChainArgos Ecosystem

Ethecycle is one of three interconnected repositories in the ChainArgos blockchain analytics platform:

```
┌─────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐
│     node-etl        │     │ airflow_blockchain_etl│     │    ethecycle         │
│                     │     │                      │     │                     │
│ Raw blockchain data │────>│ Orchestration layer  │────>│ Graph analysis      │
│ extraction from     │     │ DAGs, wallet tagging │     │ Neo4j transaction   │
│ RPC nodes & APIs    │     │ 90+ tag operators    │     │ graph database      │
│                     │     │ Redshift warehouse   │     │ Forensic queries    │
│ 25+ chains          │     │ Compliance reporting │     │ 14+ chains          │
└─────────────────────┘     └──────────────────────┘     └─────────────────────┘
```

- **node-etl** produces the raw transaction CSVs that ethecycle consumes
- **airflow_blockchain_etl** orchestrates the wallet tagging pipeline that enriches ethecycle's address database
- **ethecycle** provides graph-based forensic analysis capabilities on top of the enriched data
