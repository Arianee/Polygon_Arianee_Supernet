# Arianee RPC NODE

**Requirements:**

1. Genesis file from the existing deployment, standard genesis file will be provided on the repo, This needs to be modified by updating the boot node details with non-validator details
2. JSON RPC URL for Ethereum from service providers such as Alchemy
3. A ec2 instance(minimum a c6i.large type) hosted outside form the supernet internal network on which the RPC node is going to be deployed

**Steps for setting up edge services and OS dependencies:-**

EC2 instance( minimum a c6i.large type ) with Ubuntu 22.04 LST is recommended for setting up a node. The setup procedure mentioned is for Ubuntu OS. Use the provided

```
polygon-edge
```

Binary for deployment

```
sudo apt update
sudo apt upgrade
```

Create the folder for the node files

```
cd ~
git clone https://github.com/Arianee/Polygon_Arianee_Supernet
cd Polygon_Arianee_Supernet
sudo mkdir /var/lib/edge
sudo chown $user:$user /var/lib/edge
sudo cp polygon-edge /usr/local/bin/
```

Create edge.service file on

```
/etc/systemd/system/
```

Using below command

```
sudo cp edge_non_validator.service /etc/systemd/system/edge.service
```

Below is the edge_non_validator.service for your reference. you can modify the parameters such as JSON rpc port if needed

```
[Unit]
Description=Polygon Edge Client
Documentation=https://github.com/Arianee/Polygon_Arianee_Supernet

# Bring this up after the network is online
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=polygon-edge server --data-dir /var/lib/edge \
    --chain /var/lib/edge/genesis.json \
    --grpc-address 0.0.0.0:10000 \
    --json-rpc-batch-request-limit 999 \
    --libp2p 0.0.0.0:10001 \
    --jsonrpc 0.0.0.0:10002 \
    --prometheus 0.0.0.0:9091 \
    --max-slots 276480 \
    --max-enqueued 276480 \
    --log-level DEBUG \
    --block-gas-target 10000000 \
    --num-block-confirmations 2

MemoryHigh=2690M
MemoryMax=3074M
MemorySwapMax=0

Restart=on-failure
RestartSec=5s

Type=simple

User=edge
Group=edge-group

TimeoutStartSec=infinity
TimeoutStopSec=600

RuntimeDirectory=edge
RuntimeDirectoryMode=0700

ConfigurationDirectory=edge
ConfigurationDirectoryMode=0700

StateDirectory=edge
StateDirectoryMode=0750

# Hardening measures
# https://www.linuxjournal.com/content/systemd-service-strengthening
# sudo systemd-analyze security
# systemd-analyze syscall-filter
####################

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full

# Deny access to /home, /root and /run/user
ProtectHome=true

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

# Deny any ability to create namespaces. Should not be needed
RestrictNamespaces=true

# Restrict any kind of special capabilities
CapabilityBoundingSet=

# Allow minimal system calls for IO (filesystem network) and basic systemctl operations
SystemCallFilter=@signal @network-io @ipc @file-system @chown @system-service

# Access to  /sys/fs/cgroup/ should not be needed
ProtectControlGroups=true

# We don't need access to special file systems or extra kernel modules to work
ProtectKernelModules=true

# Access to proc/sys/, /sys/, /proc/sysrq-trigger, /proc/latency_stats, /proc/acpi, /proc/timer_stats, /proc/fs and /proc/irq is not needed
ProtectKernelTunables=true

# From the docsk "As the SUID/SGID bits are mechanisms to elevate privileges, and allow users to acquire the identity of other users, it is recommended to restrict creation of SUID/SGID files to the few programs that actually require them"
RestrictSUIDSGID=true

[Install]
WantedBy=multi-user.target
```

**Steps for deployment of the RPC node:-**

Generate the node file with below command

```
polygon-edge polybft-secrets --insecure --data-dir rpc-node
```

Copy all the generated contents of the folder rpc-node to

```
/var/lib/edge/
```

In the genesis file(from existing deployment), the entire bootnodes object needs to be modified as below and save the genesis.json file to this location `/var/lib/edge/`

**Note**:

“non-validator-node-id” is from an internal non-validator that is exposed to the internet

```
sudo nano genesis.json
```

```
// look for "bootnodes" and update the below contents of the genesis.json
...
"bootnodes": [
"/ip4/bootnode1.polygonsupernet.arianee.net/tcp/10001/p2p/16Uiu2HAm2ZGC6fkLwmY4pJDBJEqFouTsQ6izStEaYRom7uGzGFTE"
]
...

// update the "jsonRPCEndpoint" with URL from a Ethereum Mainnet RPC 

...
 "jsonRPCEndpoint": "https:....."
...
```

```
sudo cp genesis.json /var/lib/edge/
sudo chown $user:$user /var/lib/edge/genesis.json
sudo cp -r rpc-node/* /var/lib/edge/
sudo chown $user:$user -R /var/lib/edge/*
```

Start the edge service to start the rpc-node using

```
sudo systemctl start edge.service
```

Enable the edge service

```
sudo systemctl enable edge.service
```

Check the edge.service logs using

```
sudo journalctl -u edge -f
```

Once you run the above command, you should be able to see the blocks getting synced you might need to wait for a while to complete the syncing process, it is based on the age of the chain. after syncing RPC node will be fully functional

Json RPC URL for this node will be available on  `port 10002`

**Validation check:**

Do a curl call to the new RPC endpoint and check the block height whether it is syncing with the supernet present block height, It might take a while for a new RPC node to fully sync to the current block height.

```
curl —location ‘http://localhost:10002' \
--header 'Content-Type: application/json' \
--data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest", true],"id":1}'
```

On result check value of the “number“ object it should be similar to

```
"number":"0x00778"
```

The value in Hexadecimal format. you need to compare this value with the current block height that can be obtained from a block explorer.




# Arianee VALIDATOR NODE

**Requirements**:-

1. one of the non-validator on the internal network has to be exposed to the Internet (need to be connected to a domain)
2. from the non-validator we need
    1. Domain name of the exposed non-validator
    2. Node-id of the above non-validator
3. Genesis file from the existing deployment, standard genesis file will be provided on the repo, This needs to be modified by updating the bootnode details with non-validator details
4. Json RPC URL for Ethereum from service providers such as Alchemy
5. a ec2 ( minimum a c6i.large type ) instance hosted outside form supernet internal network on which RPC node to be deployed


**Steps for setting up edge services and OS dependencies:-**
EC2 instance(minimum a c6i.large type) with Ubuntu 22.04 LST is recommended for setting up the validator node. The setup procedure below, mentioned is for Ubuntu OS. Use the provided `polygon-edge` binary for deployment

```
sudo apt update
sudo apt upgrade
```

create the folder for the node files

```
cd ~
git clone https://github.com/Arianee/Polygon_Arianee_Supernet
cd Polygon_Arianee_Supernet
sudo mkdir /var/lib/edge
sudo chown $user:$user /var/lib/edge
sudo cp polygon-edge /usr/local/bin/
```

Create edge.service file on `/etc/systemd/system/` using below command

```
sudo cp edge_validator.service /etc/systemd/system/edge.service
```

below is the edge_validator.service for your reference. you can modify the parameters such as json rpc port if needed. Also username and group name needs to be change according to current user name. Here we have used **ubuntu** for username and group

```
[Unit]
Description=Polygon Edge Client
Documentation=https://github.com/Arianee/Polygon_Arianee_Supernet

# Bring this up after the network is online
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=polygon-edge server --data-dir /var/lib/edge \
    --chain /var/lib/edge/genesis.json \
    --grpc-address 127.0.0.1:10000 \
    --json-rpc-batch-request-limit 999 \
    --libp2p 0.0.0.0:10001 \
    --jsonrpc 127.0.0.1:10002 \
    --prometheus 127.0.0.1:9091 \
    --max-slots 276480 \
    --max-enqueued 276480 \
    --log-level INFO \
    --block-gas-target 10000000 \
    --num-block-confirmations 2 \
    --seal

MemoryHigh=2690M
MemoryMax=3074M
MemorySwapMax=0

Restart=on-failure
RestartSec=5s

Type=simple

User=ubuntu
Group=ubuntu

TimeoutStartSec=infinity
TimeoutStopSec=600

RuntimeDirectory=edge
RuntimeDirectoryMode=0700

ConfigurationDirectory=edge
ConfigurationDirectoryMode=0700

StateDirectory=edge
StateDirectoryMode=0750

# Hardening measures
# https://www.linuxjournal.com/content/systemd-service-strengthening
# sudo systemd-analyze security
# systemd-analyze syscall-filter
####################

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full

# Deny access to /home, /root and /run/user
ProtectHome=true

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

# Deny any ability to create namespaces. Should not be needed
RestrictNamespaces=true

# Restrict any kind of special capabilities
CapabilityBoundingSet=

# Allow minimal system calls for IO (filesystem network) and basic systemctl operations
SystemCallFilter=@signal @network-io @ipc @file-system @chown @system-service

# Access to  /sys/fs/cgroup/ should not be needed
ProtectControlGroups=true

# We don't need access to special file systems or extra kernel modules to work
ProtectKernelModules=true

# Access to proc/sys/, /sys/, /proc/sysrq-trigger, /proc/latency_stats, /proc/acpi, /proc/timer_stats, /proc/fs and /proc/irq is not needed
ProtectKernelTunables=true

# From the docsk "As the SUID/SGID bits are mechanisms to elevate privileges, and allow users to acquire the identity of other users, it is recommended to restrict creation of SUID/SGID files to the few programs that actually require them"
RestrictSUIDSGID=true

[Install]
WantedBy=multi-user.target
```


**Steps for deployment of the validator node:-**

1. generate the node file with the below command

```
polygon-edge polybft-secrets --insecure --data-dir ext-validator
```

copy all the generated contents of the folder ext-validator to `/var/lib/edge/` . Save the generated logs we will be using them in further process.


1. In the genesis file(from existing deployment), the entire bootnodes object needs to be modified as below and save the genesis.json file to this location `/var/lib/edge/`
   **Note**:- “non-validator-node-id” is from an internal non-validator that is exposed to the internet

```
sudo nano genesis.json 
```

```
// look for "bootnodes" and update the below contents of the genesis.json
...
"bootnodes": [
"/ip4/bootnode1.polygonsupernet.arianee.net/tcp/10001/p2p/16Uiu2HAm2ZGC6fkLwmY4pJDBJEqFouTsQ6izStEaYRom7uGzGFTE"
]
...

// look for "jsonRPCEndpoint" and update with URL from Alchemy 

...
 "jsonRPCEndpoint": "https:....."
...
```

1. from the logs you will find some thing like public key, which will be your validator’s address. please fund the wallet address with some ETH for paying gas and ARIA20 token (10000 tokens) that needs to be staked

```
...
 Public key (address) = 0x1a6e68b42B41a158F8095f4D3675E01F8eC0E0dC
...
```

1. After Funding the wallet white-list the validator address, this needs to be done by Arianee, communicate your validator address to Arianee for the whitelisting of the validator.

```
polygon-edge polybft whitelist-validators \
  --private-key <deployer_wallet_key> \
  --addresses <validator_address> \
  --supernet-manager <customSupernetManagerAddr_from_genesis_file> \
  --jsonrpc <json_rpc_url_root_chian_from_alchemy> 
```

1. Register validator

```
polygon-edge polybft register-validator --data-dir ./ext-validator \
--jsonrpc <json_rpc_url_root_chian> \
--supernet-manager <customSupernetManagerAddr_from_genesis_file>
```

1. Stake the token from the validator

```
polygon-edge polybft stake --data-dir ./ext-validator \
--supernet-id <supernetID_from_genesis_file> \
--amount 10000000000000000000000 \
--jsonrpc <json_rpc_url_root_chian_from_alchemy> \
--stake-manager <stakeManagerAddr_from_genesis_file> \
--stake-token <stakeTokenAddr_from_genesis_file> 
```

1. start the edge service to start the validator using

```
sudo systemctl start edge.service 
```

1. Enable the edge service

```
sudo systemctl enable edge.service
```

1. check the edge.service logs using

```
sudo journalctl -u edge -f
```

once you run the above command, you should be able to see the blocks getting synced you might need to wait for a while to complete the syncing process, it is based on the age of the chain. The validator will be stating consensus only after passing an epoch or a checkpoint. Please check the time when the next epoch or a checkpoint is going to happen, so that you can know when the validator will start its consensus process.

1. Json RPC URL for this node will be available on   ` port 10002 `

**Validation check:-**

1.  Do a curl call to the new RPC endpoint and check the block height whether it is syncing with the supernet present block height, It might take a while for a new validator node to fully sync to the current block height.

```
curl —location ‘[http://localhost:10002](http://localhost:10001/)' \
--header 'Content-Type: application/json' \
--data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest", true],"id":1}'
```

On  result check value of “number“ object it should be similar to  `"``number":"0x00778"`  the value in Hexadecimal format. you need to compare this value with the current block height that can be obtained from a block explorer.


1. check their logs after the validator is fully synced and an epoch or checkpointing is passed,

```
sudo journalctl -u edge -f
```

you should be able to see these lines on the logs, and then you can make sure the validator started the consensus processes, check below screencap for reference

```
...
polygon.server.polybft: we are the proposer ...
polygon.server.polybft.consensus_runtime: Proposer calculated: ...
```


**Note:-**
After deploying the validator, the validator has to fully sync and an epoch or check-point has to pass, after this, the validator will start the consensus process. 
