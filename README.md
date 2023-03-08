# Nibiru
### Nibiru Node Installation Instructions </br>
## [Official documentation](https://nibiru.fi/docs/run-nodes/testnet/) </br>

### Minimum hardware requirements. </br>
4 CPU </br>
16 GB RAM </br>
500 GB of disk space (SSD) </br>

#### Update the system:
```
sudo apt update && sudo apt upgrade --yes
```
#### Install nibid:
```
curl -s https://get.nibiru.fi/@v0.19.2! | bash
```
```
nibid version #v0.19.2
```
#### Init the chain:
```
nibid init <moniker-name> --chain-id=nibiru-itn-1 --home $HOME/.nibid
```
#### Create a local key pair:
```
nibid keys add <key-name>
```
#### Copy the genesis file to the $HOME/.nibid/config folder.</br>
#### You can get genesis from our networks endpoint with:
```
NETWORK=nibiru-itn-1
curl -s https://networks.itn.nibiru.fi/$NETWORK/genesis > $HOME/.nibid/config/genesis.json
```
#### Verify Genesis File Checksum:
```
shasum -a 256 $HOME/.nibid/config/genesis.json #e162ace87f5cbc624aa2a4882006312ef8762a8a549cf4a22ae35bba12482c72 $HOME/.nibid/config/genesis.json
```
#### Update seeds list in the configuration file $HOME/.nibid/config/config.toml:
```
NETWORK=nibiru-itn-1
sed -i 's|seeds =.*|seeds = "'$(curl -s https://networks.itn.nibiru.fi/$NETWORK/seeds)'"|g' $HOME/.nibid/config/config.toml
```
#### Set minimum gas prices:
```
sed -i 's/minimum-gas-prices =.*/minimum-gas-prices = "0.025unibi"/g' $HOME/.nibid/config/app.toml
```
#### Request tokens from the Web Faucet for nibiru-itn-1 if required: _Faucet can be used once every 6 hours_
```
FAUCET_URL="https://faucet.itn-1.nibiru.fi/"
ADDR="..." # your address 
curl -X POST -d '{"address": "'"$ADDR"'", "coins": ["11000000unibi","100000000unusd","100000000uusdt"]}' $FAUCET_URL
```
#### Create a service file:
```
sudo tee /etc/systemd/system/nibiru.service<<EOF
[Unit]
Description=Nibiru Node
After=network-online.target
[Service]
User=$USER
ExecStart=$(which nibid) start
Restart=on-failure
RestartSec=10
LimitNOFILE=10000
[Install]
WantedBy=multi-user.target
EOF
```
#### Enable and Start the service:
```
sudo systemctl daemon-reload
sudo systemctl enable nibiru.service
sudo systemctl start nibiru.service
```
#### State Sync (Thanks guys at nodejumper):
```
sudo apt install lz4 -y

sudo systemctl stop nibiru.service

cp $HOME/.nibid/data/priv_validator_state.json $HOME/.nibid/priv_validator_state.json.backup
nibid tendermint unsafe-reset-all --home $HOME/.nibid --keep-addr-book

SNAP_RPC="https://nibiru-testnet.nodejumper.io:443"

LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

PEERS="a1b96d1437fb82d3d77823ecbd565c6268f06e34@nibiru-testnet.nodejumper.io:27656"
sed -i 's|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.nibid/config/config.toml

sed -i 's|^enable *=.*|enable = true|' $HOME/.nibid/config/config.toml
sed -i 's|^rpc_servers *=.*|rpc_servers = "'$SNAP_RPC,$SNAP_RPC'"|' $HOME/.nibid/config/config.toml
sed -i 's|^trust_height *=.*|trust_height = '$BLOCK_HEIGHT'|' $HOME/.nibid/config/config.toml
sed -i 's|^trust_hash *=.*|trust_hash = "'$TRUST_HASH'"|' $HOME/.nibid/config/config.toml

mv $HOME/.nibid/priv_validator_state.json.backup $HOME/.nibid/data/priv_validator_state.json

curl -s https://snapshots2-testnet.nodejumper.io/nibiru-testnet/wasm.lz4 | lz4 -dc - | tar -xf - -C $HOME/.nibid/data
```
#### Restart the service and check the logs:
```
sudo systemctl restart nibiru.service
sudo journalctl -u nibiru.service -f
```
#### Create validator:
```
nibid tx staking create-validator \
--amount 10000000unibi \
--commission-max-change-rate "0.1" \
--commission-max-rate "0.20" \
--commission-rate "0.1" \
--min-self-delegation "1" \
--details "put your validator description there" \
--pubkey=$(nibid tendermint show-validator) \
--moniker <your_moniker> \
--chain-id nibiru-itn-1 \
--gas-prices 0.025unibi \
--from <key-name>
```
#### Find out valoper address:
```
nibid q staking validator $(nibid keys show <wallet_name> --bech val -a)
```
#### Checking the balance:
```
nibid q bank balances <you_address>
```
#### Delegate to our validator:
```
nibid tx staking delegate YOUR_VALOPER_ADDRESS 10000000unibi --from <wallet_name> --chain-id nibiru-itn-1 --gas="auto" --gas-adjustment=1.5
```
[Nibiru explorer](https://itn-1.nibiru.fi/)
