# Guide Initia Node

## Testnet Setup Instructions

### Install dependencies

Update system and install build tools
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget build-essential jq make lz4 gcc unzip -y
```

### Install GO
```
cd $HOME && \
ver="1.21.5" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

### Install binary
```
git clone https://github.com/initia-labs/initia
cd initia
git checkout v0.2.10
make install
```

### Init Moniker
```
initiad init YOUR-MONIKER --chain-id initiation-1
```

### Set up configuration
Prepare the genesis file And set some mandatory configuration options
```
rm ~/.initia/config/genesis.json
wget https://initia.s3.ap-southeast-1.amazonaws.com/initiation-1/genesis.json
cp genesis.json ~/.initia/config/genesis.json
```

This will prevent continuous reconnection try. (default P2P_PORT is 26656)
```
sed -i -e 's/external_address = \"\"/external_address = \"'$(curl httpbin.org/ip | jq -r .origin)':26656\"/g' ~/.initia/config/config.toml
```

### Set minimum gas price & peers
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0025uinit"|g' $HOME/.initia/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.initia/config/config.toml
```
```
PEERS="093e1b89a498b6a8760ad2188fbda30a05e4f300@35.240.207.217:26656" && \
sed -i -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.initia/config/config.toml
```

### Set pruning
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.initia/config/app.toml
```

### Create service file
```
sudo tee /etc/systemd/system/initiad.service > /dev/null <<EOF
[Unit]
Description=initiad
After=network-online.target

[Service]
User=$USER
ExecStart=$(which initiad) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### Enable and start service
```
sudo systemctl daemon-reload
sudo systemctl enable initiad  
sudo systemctl restart initiad  && sudo journalctl -fu initiad  -o cat
```

## Key management

### Add new wallet
```
initiad keys add wallet
```

### Restore wallet
```
initiad keys add wallet --recover
```

### Check wallet balance 
```
initiad q bank balances $(initiad keys show wallet -a)
```

### Create validator
```
initiad tx mstaking create-validator \
    --amount=5000000uinit \
    --pubkey=$(initiad tendermint show-validator) \
    --moniker="<your_moniker>" \
    --chain-id=initiation-1 \
    --from=wallet \
    --commission-rate="0.10" \
    --commission-max-rate="0.20" \
    --commission-max-change-rate="0.01" \
    --identity=<keybase_identity>
```

## Additional commands

### Validator info
```
initiad q staking validator $(initiad keys show $WALLET --bech val -a)
```

### Node info
```
initiad status 2>&1 | jq
```

### Delegate to your validator
```
initiad tx staking delegate $(initiad keys show wallet --bech val -a) 1000000uinit --from wallet -y
```

### Unjail Validator 
```
initiad tx slashing unjail --from wallet --fees=0.025uinit -y
```

### Stop Node
```
sudo systemctl stop initiad
```

### Restart Node
```
sudo systemctl restart initiad
```

### Check Log
```
sudo journalctl -u initiad -f -o cat
```

### Disable Indexing
```
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.initia/config/config.toml
```

### Delete Node
```
sudo systemctl stop initiad.service
sudo systemctl disable initiad.service
sudo rm /etc/systemd/system/initiad.service
rm -rf $HOME/.inita $HOME/initia
```
