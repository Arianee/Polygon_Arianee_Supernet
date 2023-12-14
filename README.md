# Arianee-External-Validator-Tool

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
"/ip4/15.188.145.41/tcp/10001/p2p/16Uiu2HAm2ZGC6fkLwmY4pJDBJEqFouTsQ6izStEaYRom7uGzGFTE"
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
