# Instructions Aptos Testnet Test2
Download Wallet "Petra" for Chrome.

https://github.com/aptos-labs/aptos-core/releases?q=wallet&expanded=true

Open chrome and go to chrome://extensions

Enable Developer mode at the top right of the Extensions page.

Click on Load unpacked at the top left, and point it to the folder where you just unzipped the downloaded Wallet release.****

Open Petra and create a new wallet. Backup the keys.

In settings bottom left click credentials copy the ADDRESS The ADDRESS is your OWNER WALLET. The bottom one.

Paste it to the googledocs spreadsheet from Sherry

In the Petra wallet, go to Settings -> Network -> Add -> Node URL: https://sherryx.aptosdev.com

Once connected to sherry testnet, you should see the token I airdropped to your wallet address from the spreadsheet you filled in. (marked light green)

The rest of this document assumes you have coins in your wallet.

Requirements for vps or dedicated server 8 core, 16 threads, 32GB ram 2.8ghz or faster 300GB storage or more.

Update & upgrade the machine.
```bash
sudo apt update && sudo apt upgrade
```
Install requirements.
```bash
sudo apt install git curl unzip
```
Clone the Aptos repo.
```bash
git clone https://github.com/aptos-labs/aptos-core.git
```
cd into aptos-core directory.
```bash
cd aptos-core
```
Run the setup script to download more requirements.
```bash
echo y | ./scripts/dev_setup.sh
```
Update shell environment for rust.
```bash
source ~/.cargo/env
```
Check out the testnet branch.
```bash
git checkout --track origin/testnet
```
Build aptos "Cli"
```bash
cargo build -p aptos --release
```
Build aptos-node
```bash
cargo build -p aptos-node --release
```
Move aptos-node to /usr/local/bin/
```bash
mv  ~/aptos-core/target/release/aptos-node /usr/local/bin
```
Move aptos to /usr/local/bin/
```bash
mv  ~/aptos-core/target/release/aptos /usr/local/bin
```
Set variables and create a directory for your Aptos files, "Change testnet and alice"
```bash
export WORKSPACE=testnet
export USERNAME=alice
mkdir ~/$WORKSPACE
```
Change directory.
```bash
cd ~/$WORKSPACE
```
Generate keys. 
```bash
aptos genesis generate-keys --output-dir ~/$WORKSPACE/keys
```
Configure validator information add your ipaddress below<br>
This creates a new folder from your username above with 2 files owner.yaml and operator.yaml
```bash
aptos genesis set-validator-configuration \
    --local-repository-dir ~/$WORKSPACE \
    --username $USERNAME \
    --owner-public-identity-file ~/$WORKSPACE/keys/public-keys.yaml \
    --validator-host IPADDRESSHERE:6180 \
    --stake-amount 100000000000000
```
Create the layout template.
```bash
aptos genesis generate-layout-template --output-file ~/$WORKSPACE/layout.yaml
```
Make it look exactly the same as below the only thing you need to change is users. The root key and chain_id needs to be the one in this example. 
```bash
nano ~/$WORKSPACE/layout.yaml
```
```bash
root_key: "D04470F43AB6AEAA4EB616B72128881EEF77346F2075FFE68E14BA7DEBD8095E"
users: ["<username you specified from previous step>"]
chain_id: 43
allow_new_validators: false
epoch_duration_secs: 7200
is_test: true
min_stake: 100000000000000
min_voting_threshold: 100000000000000
max_stake: 100000000000000000
recurring_lockup_duration_secs: 86400
required_proposer_stake: 100000000000000
rewards_apy_percentage: 10
voting_duration_secs: 43200
voting_power_increase_limit: 20
```
Download genesis.blob and waypoint.txt
```bash
wget https://sherryx.aptosdev.com/genesis.blob 
wget https://sherryx.aptosdev.com/waypoint.txt 
```
Build and copy the AptosFramework Move package into the ~/$WORKSPACE directory as framework.mrb
```bash
cd ~/aptos-core
cargo run --package framework -- release
cp head.mrb ~/$WORKSPACE/framework.mrb
```
Create a directory to hold your data
```bash
mkdir ~/$WORKSPACE/data
```
Copy validator.yaml 
```bash
mkdir ~/$WORKSPACE/config
cp docker/compose/aptos-node/validator.yaml ~/$WORKSPACE/config/validator.yaml
```
Edit the file so it looks similar to below changing testnet to what you set in the command "export WORKSPACE=testnet" and username in this example its alice from the command "export USERNAME=alice"
```bash
nano ~/$WORKSPACE/config/validator.yaml
```
```bash
base:
  role: "validator"
  data_dir: "~/testnet/data"
  waypoint:
    from_file: "~/testnet/waypoint.txt"

consensus:
  safety_rules:
    service:
      type: "local"
    backend:
      type: "on_disk_storage"
      path: ~/testnet/data/secure-data.json
      namespace: ~
    initial_safety_rules_config:
      from_file:
        waypoint:
          from_file: ~/testnet/waypoint.txt
        identity_blob_path: ~/testnet/alice/validator-identity.yaml
  quorum_store_poll_count: 1

execution:
  genesis_file_location: "~/testnet/genesis.blob"
  concurrency_level: 4

validator_network:
  discovery_method: "onchain"
  mutual_authentication: true
  identity:
    type: "from_file"
    path: ~/testnet/alice/validator-identity.yaml

full_node_networks:
- network_id:
    private: "vfn"
  listen_address: "/ip4/0.0.0.0/tcp/6181"
  identity:
    type: "from_config"
    key: "b0f405a3e75516763c43a2ae1d70423699f34cd68fa9f8c6bb2d67aa87d0af69"
    peer_id: "00000000000000000000000000000000d58bc7bb154b38039bc9096ce04e1237"

api:
  enabled: true
  address: "0.0.0.0:8080"
```
Make a systemd service
```bash
sudo nano /etc/systemd/system/aptosd.service
```
Copy and paste below change testnet to what you set in the command "export WORKSPACE=testnet" that is the only thing to change here.

To save press control x then press y
```bash
[Unit]
Description=Aptos
After=network.target

[Service]
User=$USER
Type=simple
ExecStart=/usr/local/bin/aptos-node -f $HOME/testnet/config/validator.yaml
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Update your account_address in validator-identity.yaml to your owner wallet address. This is the ADDRESS from Petra.
```bash
nano ~/$WORKSPACE/keys/validator-identity.yaml
```
Change directory
```bash
cd ~/$WORKSPACE
```
Initialize CLI with your wallet private key from Petra, you can get in from Settings -> Credentials
```bash
aptos init --profile ait3-owner \
  --rest-url https://sherryx.aptosdev.com \
  --private-key <owner_private_key>
```
Send some coins 5000 from petra to your operator address for gas.
Operator address and voter address can be the same they are found in the keys directory public-keys.yaml and its the top line the account_address. Once you have sent gas run the command below important not to change the initial stake amount or you will have problems later.
```bash
aptos stake initialize-stake-owner \
  --initial-stake-amount 100000000000000 \
  --operator-address <operator-address> \
  --voter-address <voter-address> \
  --profile ait3-owner
```
Start the node reload systemd enable service start on boot. 
```bash
sudo systemctl restart systemd-journald
sudo systemctl daemon-reload
sudo systemctl enable aptosd
sudo systemctl restart aptosd
```
If you need to check logs; 
```bash
journalctl -eu aptosd
```
Setup the validator node using operator account, and join the validator set. We created before. account_private_key for operator can be found in the private-keys.yaml file under ~/$WORKSPACE/keys folder.
```bash
aptos init --profile ait3-operator \
--private-key <operator_account_private_key> \
--rest-url https://sherryx.aptosdev.com/ \
--skip-faucet
```
Update validator network addresses on chain owner address is the one from Petra.
```bash
aptos node update-validator-network-addresses  \
  --pool-address <owner-address> \
  --operator-config-file ~/$WORKSPACE/$USERNAME/operator.yaml \
  --profile ait3-operator
```
Update validator consensus key on chain. Owner address is the one from Petra.
```bash
aptos node update-consensus-key  \
  --pool-address <owner-address> \
  --operator-config-file ~/$WORKSPACE/$USERNAME/operator.yaml \
  --profile ait3-operator
```

Join validator set.
```bash
aptos node join-validator-set \
  --pool-address <owner-address> \
  --profile ait3-operator
```

ValidatorSet will be updated at every epoch change, which is once every 2 hours. You will only see your node joining the validator set in next epoch. Both Validator and fullnode will start syncing once your validator is in the validator set.

After 2 hours check if your node is syncing
```bash
curl 127.0.0.1:9101/metrics 2> /dev/null | grep "aptos_state_sync_version"
```
