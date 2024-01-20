## Tutorial for running a cosmoshub node and configuring cometBft for storing data in PostgreSQL table

CometBft provides the capability to index blocks and transactions for subsequent querying. One can query blocks and transactions data using node's JSON-RPC or by running our own node and query the data from the database.We can see all the available rpc and their specification [here](https://github.com/cometbft/cometbft/blob/main/spec/rpc/README.md).

There are three options to configure cometbft for indexing - null, kv and psql.

1. null - Node will not index the data 
2. kv - kv is used by default for indexing, node store data in key-value pairs(using levelDB)
3. psql - Node can store the blocks and transactions data in postgres table.

We are going to start a blockchain node, configure it for using psql indexing type to store the data in the postgres table and query the table to get all the transactions from a single block height.

Checkout [this](https://hub.cosmos.network/main/hub-tutorials/join-mainnet.html#hardware) for hardware requirements for running a node in your system, we will be running a pruned node(pruned node only store data for some past blocks only, it is configurable).

**Steps**

1. Install Gaia binary - Follow [this guide](https://hub.cosmos.network/main/getting-started/installation.html) to install gaia binary in your system.

2. Initialize chain - Initialize chain using below command and provide human readable name as custom-moniker. It will create `~/.gaia` directory with subfolders config and data.

    ```go
    gaiad init <custom-moniker>
    ```

3. Download genesis - Download genesis file and move it in the `~/.gaia/config` directory. Below command will download the genesis file, unzip it and move it in `~/.gaia/config` directory.

    ```
    wget https://raw.githubusercontent.com/cosmos/mainnet/master/genesis/genesis.cosmoshub-4.json.gz
    gzip -d genesis.cosmoshub-4.json.gz
    mv genesis.cosmoshub-4.json ~/.gaia/config/genesis.json
    ```

4. **Configure config.toml for adding seed and peers** 

    Upon initialisation the node will need to connect to the peer to be part of the chain. We can find active peers information on [polkachu](https://polkachu.com/live_peers/cosmos).
    Update `~/.gaia/config/config.toml` toml file with seed and persistent peer information from the polkachu.

    ```toml
    seeds = "1fa4f7c1d32aa695a0d6c83b5960421e6b2bc981@65.21.33.236:26656, .....,c843429ef1b4a551dab64901991c41af2bc4454f@65.108.130.171:26656

    persistent_peers = "f4ae01e628935aeb72c52018bd9f9e9e4c8706ee@49.12.165.122:26104,914f2b9c7ee4a6a806ff24cd58379418aed0c121@3.144.11.250:26656,97da81404be96002eb999572ed51f96f460c0b3a@51.91.74.8:26656,67685d93f2256caa7a2d53e3a104f9e437c3d247@95.216.114.244:26656,8220e8029929413afff48dccc6a263e9ac0c3e5e@204.16.247.237:26656"
    ``` 

5. **Run a postgres instance and create tables for storing block and transaction data**

    Before starting the node we will have to setup postgreSql database server where block and transaction data will be stored. Postgres is available in all the ubuntu versions. If you are using other os then follow [this guide](https://www.postgresql.org/download/) to install postgres.

    We also have to create postgres table in which cometBft will store the data. The database schema is provided in [internal/state/indexer/sink/psql/schema.sql](https://github.com/cometbft/cometbft/blob/main/internal/state/indexer/sink/psql/schema.sql#L21-L22) directory. Download this file locally.
    
    Now connect to the database and execute the `schema.sql` script which will create the table.

    ```
    # psql -U <username> -W -d <database-name> -p <port> -h <host> -f <schema.sql-directory>
    psql -U postgres -W -d cosmoshub -p 5432 -h localhost -f ./schema.sql
    ```
    It will prompt you to enter the password, enter password and the press enter. It has now created database named cosmoshub and created tables specified in the `schema.sql`.

6. **Configure config.toml for indexing, using psql** 

    Open the `~/.gaia/config/config.toml` file and modify the indexer within the `[tx-index]` section to use `psql`. Additionally, update the `psql-conn` with the PostgreSQL configuration. The format for the PostgreSQL connection is `postgresql://<user>:<password>@<host>:<port>/<db>?<opts>`. In our scenario, the PostgreSQL configuration will be `postgresql://postgres:<your-password>@127.0.0.1:5432/cosmoshub`.

    ```toml
    [tx_index]
    indexer = "psql"

    # The PostgreSQL connection configuration, the connection format:
    #  postgresql://<user>:<password>@<host>:<port>/<db>?<opts>
    psql-conn = "postgresql://postgres:<your-password>@127.0.0.1:5432/cosmoshub"
    ```

7. **Start the node and sync blocks using pruned snapshot**

    We are going to use the pruned snapshot to sync the blocks. Download the pruned snapshot of cosmoshub from [here](https://quicksync.io/networks/cosmos.html).
    ```
    wget https://dl2.quicksync.io/cosmoshub-4-pruned.20240118.0310.tar.lz4
    ```  
    `lz4` is required to extract the snapshot. To install `lz4` - 

    ```
    sudo apt update
    sudo apt install snapd -y
    sudo snap install lz4
    ```

    Follow the below commands to extract the snapshot file in `~/.gaia` directory.

    ```
    lz4 -c -d cosmoshub-4-pruned.20240118.0310.tar.lz4  | tar -x -C $HOME/.gaia
    ```

    At this point, we have successfully installed the Gaia binary, initialized the chain, updated peers and indexing configuration, updated the genesis file with cosmoshub genesis file and saved a snapshot file of the CosmosHub for syncing to the chain. It is now time to start the node and to be a part of the chain.

8. **Start the node**

    All the configurations are ready now. Now start the node -

    ```
    gaiad start
    ```

    It will start syncing blocks and you can see your postgres table populating.

    Now, it is time to retrieve all the transaction data from a specific height. It is advisable to wait for the blocks to synchronize. To determine the synchronization status, check the latest block height on [Mintscan](https://www.mintscan.io/cosmos/block) and compare it with the block height indicated in the terminal logs.

9. **Query all the transactions data for a particular blockheight**

    The below query will fetch all the transaction result(TxResult) for blockheight 18761861(update the height).
    Open a terminal, connect to the postgreSql database using psql and run the below query.

    ```
    postgres=# SELECT 
        tx_results.rowid AS rowid,
        tx_results.block_id AS block_id,
        tx_results.index AS index,
        tx_results.created_at AS created_at,
        tx_results.tx_hash,
        tx_results.tx_result
        FROM blocks
        JOIN tx_results ON blocks.rowid = tx_results.block_id
        WHERE blocks.height=18761861;
    ```
    You can see the all the transactions inforamtion from the blockheight 18761861.
    
    ```
    rowid   | block_id | index | created_at                    | tx_hash    |       tx_result
    1280743 |    49676 |     0 | 2024-01-17 15:30:20.035306+00 | 268C..04F0 | \x088591f9...163635f736571
    ```
