# Arx-Node-Setup
Everything you need to know about how I set up my Arcium testnet Node.
All information here is gotten from the official [Arcium docs](https://docs.arcium.com/developers/node-setup), make sure to check it out.

## PREREQUISITIES 
First, you’ll prepare your environment by installing the necessary tools and generating security keys. 
Then you’ll get your node registered onchain and configure it to run. 
Finally, you’ll connect to other nodes in a cluster and start doing computations.

Rust
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Solana CLI
```
curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash
```

A successful installation will return output like the following:
<pre>
Installed Versions:
Rust: rustc 1.86.0 (05f9846f8 2025-03-31)
Solana CLI: solana-cli 2.2.12 (src:0315eb6a; feat:1522022101, client:Agave)
Anchor CLI: anchor-cli 0.31.1
Node.js: v23.11.0
Yarn: 1.22.1
</pre>

Docker Compose & git
```
sudo apt-get update
sudo apt install git
sudo apt-get install ./docker-desktop-amd64.deb
```
openSSL is usually installed by default on Linux

## SETTING-UP YOUR WORKSPACE

We need to create a dedidcated folder for our node
```
mkdir arcium-node-setup
cd arcium-node-setup
```

## INSTALLING ARCIUM TOOLING 

The Arcium tooling suite includes the CLI and ARX node software. Install it using the automated installer:
```
curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ | bash
```

This script will:
+ Check for all required dependencies
+ Install arcup (Arcium’s version manager)
+ Install the latest Arcium CLI
+ Install the ARX node software

verify the installation
```
arcium --version
arcup --version
```

## GENERATE REQUIRED KEYS 

Security is crucial for node operations, so we’ll create three separate keypairs that each handle different aspects of your node’s functionality.
Your ARX node needs three different keypairs for secure operation. Create these in your `arcium-node-setup` directory: 

+ Node Authority Keypair
  
This Solana keypair identifies your node and handles onchain operations:

```
solana-keygen new --outfile node-keypair.json --no-bip39-passphrase
```

Note: The --no-bip39-passphrase flag creates a keypair without a passphrase for easier automation.
​


+ Callback Authority Keypair
  
This Solana keypair signs callback computations and must be different from your node keypair for security separation:

```
solana-keygen new --outfile callback-kp.json --no-bip39-passphrase
```



+ Identity Keypair
  
This keypair handles node-to-node communication and must be in PKCS#8 format:

```
openssl genpkey -algorithm Ed25519 -out identity.pem
```


Keep these keypairs safe and private - they’re like the master keys to your node. Store them securely and don’t share them with anyone.

​
## FUND YOUR ACCOUNTS

Currently as of writing, funding using the solana CLI isn't working so we'd fund our generated `Node Authority` and `Callback Authority` wallets using the [Solana Faucet](https://faucet.solana.com/).
Proceed to fund both accounts with some devent SOL (2.5 SOL recommended).


## INITIALIZE NODE ACCOUNTS

Now we’ll register our node with the Arcium network by creating its onchain accounts. This step tells the blockchain about your node and its capabilities.

Set your Solana CLI to Devnet once so you can avoid passing `--rpc-url <rpc-url>` repeatedly: 
```
solana config set --url https://api.devnet.solana.com
```

Get your Public IP address
```
curl https://ipecho.net/plain ; echo
```

#### Get your RPC URL.
Its recommended to get a private rpc url to avoid performance issues or down time issues that arise from using public rpcs at peak times. 
I got my private url from [Helius](https://www.helius.dev/) and their free plan works perfectly fine for this node. 

Proceed to setup your helius account and create a project to get your RPC and WSS urls. Remember to switch from mainnet to devnet befor copying your RPC and WSS urls. 


<img width="1809" height="685" alt="image" src="https://github.com/user-attachments/assets/3be4b74b-f26c-4d7c-9796-4743671be667" />

Use the init-arx-accs command to initialize all required onchain accounts for your node:

```
arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset <your-node-offset> \
  --ip-address <your-node-ip> \
  --rpc-url https://devnet.helius-rpc.com/?api-key=1111111112221
```
​
Proceed to edit the above using the Required Parameters:

<pre>
--node-offset: Think of this as your node’s unique ID number on the network. 
  Choose a large random number (like 4-6 digits) to make sure it doesn’t conflict with other nodes.
  If you get an error during setup saying your number is already taken, just pick a different one and try again.
--ip-address: Your node’s public IP address
--rpc-url: Solana Devnet RPC endpoint (Your RPC gotten form Helius)
</pre>

It should look something like this when done (Example)

<pre>
  arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset 111111 \
  --ip-address 112.122.12.11 \
  --rpc-url [https://api.devnet.solana.com](https://devnet.helius-rpc.com/?api-key=11111111122111)
</pre>


If successful, you’ll see confirmation that your node accounts have been initialized onchain.

## CONFIGURING YOUR NODE

The configuration file is like your node’s instruction manual - it specifies which network to connect to, how to communicate with other nodes, and various operational settings.

Create a `node-config.toml` file in your `arcium-node-setup` directory:

```
[node]
offset = <your-node-offset>  # 
hardware_claim = 0  # Currently not required to specify, just use 0
starting_epoch = 0
ending_epoch = 9223372036854775807

[network]
address = "0.0.0.0" # Bind to all interfaces for reliability behind NAT/firewalls

[solana]
endpoint_rpc = "<your-rpc-provider-url-here>"  # Replace with your RPC provider URL or use default https://api.devnet.solana.com
endpoint_wss = "<your-rpc-websocket-url-here>"   # Replace with your RPC provider WebSocket URL or use default wss://api.devnet.solana.com
cluster = "Devnet"
commitment.commitment = "confirmed"  # or "processed" or "finalized"

```

Due to the complexities of using `nano` to edit files in the CLI, I used Virtual Studio Code to create and edit my `node-config.toml`

Link to video will be [here](https://x.com/ZennnRetired/status/1981358797252702640)

## DEPLOY YOUR NODE

Now we’ll get the node up and running using Docker, which packages everything your node needs into a clean, isolated environment.

+ Prepare Log Directory
Before running Docker, create a local directory and log file for the container to write to:

```
mkdir -p arx-node-logs && touch arx-node-logs/arx.log
```

This ensures Docker can mount and write logs without permission issues.

Start the Container
Before running Docker, verify you’re in the correct directory and have all required files:

```
pwd  # Should show: /path/to/arcium-node-setup
ls   # Should show: node-keypair.json, callback-kp.json, identity.pem, node-config.toml, arx-node-logs/
```

Now start the container:

```
docker run -d \
  --name arx-node \
  -e NODE_IDENTITY_FILE=/usr/arx-node/node-keys/node_identity.pem \
  -e NODE_KEYPAIR_FILE=/usr/arx-node/node-keys/node_keypair.json \
  -e OPERATOR_KEYPAIR_FILE=/usr/arx-node/node-keys/operator_keypair.json \
  -e CALLBACK_AUTHORITY_KEYPAIR_FILE=/usr/arx-node/node-keys/callback_authority_keypair.json \
  -e NODE_CONFIG_PATH=/usr/arx-node/arx/node_config.toml \
  -v "$(pwd)/node-config.toml:/usr/arx-node/arx/node_config.toml" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/node_keypair.json:ro" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/operator_keypair.json:ro" \
  -v "$(pwd)/callback-kp.json:/usr/arx-node/node-keys/callback_authority_keypair.json:ro" \
  -v "$(pwd)/identity.pem:/usr/arx-node/node-keys/node_identity.pem:ro" \
  -v "$(pwd)/arx-node-logs:/usr/arx-node/logs" \
  -p 8080:8080 \
  arcium/arx-node
```

Save the same command as a reusable script for convenience


```
#!/bin/bash

# Start the ARX node using Docker
docker run -d \
  --name arx-node \
  -e NODE_IDENTITY_FILE=/usr/arx-node/node-keys/node_identity.pem \
  -e NODE_KEYPAIR_FILE=/usr/arx-node/node-keys/node_keypair.json \
  -e OPERATOR_KEYPAIR_FILE=/usr/arx-node/node-keys/operator_keypair.json \
  -e CALLBACK_AUTHORITY_KEYPAIR_FILE=/usr/arx-node/node-keys/callback_authority_keypair.json \
  -e NODE_CONFIG_PATH=/usr/arx-node/arx/node_config.toml \
  -v "$(pwd)/node-config.toml:/usr/arx-node/arx/node_config.toml" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/node_keypair.json:ro" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/operator_keypair.json:ro" \
  -v "$(pwd)/callback-kp.json:/usr/arx-node/node-keys/callback_authority_keypair.json:ro" \
  -v "$(pwd)/identity.pem:/usr/arx-node/node-keys/node_identity.pem:ro" \
  -v "$(pwd)/arx-node-logs:/usr/arx-node/logs" \
  -p 8080:8080 \
  arcium/arx-node
```
then run

```
chmod +x start.sh
./start.sh
```

# VERIFY NODE OPERATION
Check that your node is running correctly:

Check Node Status

```
arcium arx-info <your-node-offset> --rpc-url https://api.devnet.solana.com

```

Check if Node is Active

```
arcium arx-active <your-node-offset> --rpc-url https://api.devnet.solana.com
```
​
# Cluster Operations

Clusters are groups of nodes that work together on computations during stress testing. You can either start your own cluster and invite others to join, or join an existing cluster that someone else created.
Clusters are groups of nodes that collaborate on MPC computations during stress testing. You have two options for cluster participation. For concepts and roles, see the Clusters Overview.
​
Option A: Create Your Own Cluster

If you want to create a new cluster and invite other nodes:

```
arcium init-cluster \
  --keypair-path node-keypair.json \
  --offset <cluster-offset> \
  --max-nodes <max-nodes> \
  --rpc-url https://api.devnet.solana.com
```

<pre>
Parameters:
--offset: Unique identifier for your cluster (different from your node offset)
--max-nodes: Maximum number of nodes in the cluster
</pre>

Example:

```
arcium init-cluster \
  --keypair-path node-keypair.json \
  --offset <cluster-offset> \
  --max-nodes 10 \
  --rpc-url https://api.devnet.solana.com
```
​
Option B: Join an Existing Cluster (MY CLUSTER)

Send me a message on [Twitter (X)](x.com/ZennnRetired) or comment your node offset to join my cluster, 
I'll send you an invite asap

To join an existing cluster, you need to be invited by the cluster authority. Once invited, accept the invitation:

```
arcium join-cluster true \
  --keypair-path node-keypair.json \
  --node-offset <your-node-offset> \
  --cluster-offset 80778 \
  --rpc-url https://api.devnet.solana.com
```

<pre>
Parameters:

true: Accept the join request (use false to reject)
--rpc-url your rpc url
--cluster-offset: The cluster’s offset you’re joining (already set to actual value 80778)
</pre>


# SENDING OUT CLUSTER INVITES TO NODES

To send out invites to other nodes to join your cluster,

you need to run:

```
arcium propose-join-cluster \
  --cluster-offset <YOUR_CLUSTER_OFFSET> \
  --node-offset <NODE_OFFSET_TO_INVITE> \
  --keypair-path <YOUR_CLUSTER_AUTHORITY_KEYPAIR> \
  --rpc-url https://api.devnet.solana.com
```

 After you propose, the node operator can accept by running:

```
arcium join-cluster true \
  --keypair-path <THEIR_NODE_KEYPAIR> \
  --node-offset <THEIR_NODE_OFFSET> \
  --cluster-offset <YOUR_CLUSTER_OFFSET> \
  --rpc-url https://api.devnet.solana.com/
```


























