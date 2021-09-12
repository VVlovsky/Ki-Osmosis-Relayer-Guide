## Used relayer client
Name: relayer \
Repository: https://github.com/cosmos/relayer \
Version: v0.9.3 \
Description: a Golang implementation of a Cosmos IBC relayer
## Installation instruction
1. First of all we need to install another network client. In my case it was Osmosis:
```shell
git clone https://github.com/osmosis-labs/osmosis
cd osmosis
git checkout v1.0.1
make install
osmosisd keys add vvosmosis --recover
```
where `vvosmosis` is my name for the wallet.
2. Now we need to install the relayer:
```shell
git clone https://github.com/cosmos/relayer.git
cd relayer
make install
```
## Configurations
```shell
cd
rly config init
mkdir rly_config
cd rly_config
sudo tee kichain-t-4_config.json > /dev/null <<EOF
{
  "chain-id": "kichain-t-4",
  "rpc-addr": "http://127.0.0.1:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
EOF
sudo tee cygnusx-osmo-1_config.json > /dev/null <<EOF
{
  "chain-id": "cygnusx-osmo-1",
  "rpc-addr": "http://54.166.148.90:26657",
  "account-prefix": "osmo",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025uosmox",
  "trusting-period": "48h"
}
EOF
rly chains add -f kichain-t-4_config.json
rly chains add -f cygnusx-osmo-1_config.json
cd
rly keys add kichain-t-4 kidrly
rly keys add cygnusx-osmo-1 osmorly
rly chains edit kichain-t-4 key kidrly
rly chains edit cygnusx-osmo-1 key osmorly
sed 's/timeout: 10s/timeout: 3m' ~/.relayer/config/config.yaml
```
After this block of commands we need to save our new addresses.
## Channels
Let's add some fund to pur relayer's addresses:
```shell
kid tx bank send $KICHAIN_WALLET_NAME $RELAYER_KI_WALLET_ADDRESS 10000000utki --home $PATH_TO_NODE --chain-id kichain-t-4
osmosisd tx bank send $OSMOSIS_WALLET_NAME $RELAYER_OSMOSIS_WALLET_ADDRESS 10000000uosmox --node http://$OSMOSIS_NODE_IP:26657/ --chain-id cygnusx-osmo-1 --fees 5000uosmox
```
where 
1. $KICHAIN_WALLET_NAME is kichain wallet name 
2. $RELAYER_KI_WALLET_ADDRESS is ki wallet which you save in the previous section 
3. $PATH_TO_NODE is the path to ki node 
4. $OSMOSIS_WALLET_NAME is osmosis wallet name 
5. $OSMOSIS_NODE_IP is osmosis node IPv4
6. $RELAYER_OSMOSIS_WALLET_ADDRESS is ki wallet which you save in the previous section 


Wait until the tokens come to the wallets and init the clients:
```shell
rly light init kichain-t-4 -f
rly light init cygnusx-osmo-1 -f
rly paths generate kichain-t-4 cygnusx-osmo-1 transfer --port=transfer
rly tx link transfer
rly paths list -d
```
## Instructions to send a cross chain transaction
Firstly let's create the service and start the relayer:
```shell
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=relayer client
After=network-online.target, kichaind.service[Service]
User=$USER
ExecStart=$(which rly) start transfer
Restart=always
RestartSec=3
LimitNOFILE=65535[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd
```
Now we can perform transactions:
1. Using the relayer
```shell
rly tx transfer kichain-t-4 cygnusx-osmo-1 1000000utki $OSMOSIS_ADDRESS --path transfer
```
2. Or using the clients:
```shell
kid tx ibc-transfer transfer transfer $CHANNEL_ID $OSMOSIS_ADDRESS 1000000utki --from KI_WALLET_NAME --home PATH_TO_NODE --chain-id kichain-t-4
```
to get the $CHANNEL_ID execute 
```shell
rly paths show transfer --yaml | grep channel-id
```
and save the first value.


Note that it can be done in the another direction. Example:
```shell
rly tx transfer cygnusx-osmo-1 kichain-t-4 100000uosmox tki1ypgejmj27mg533ufcrfsat346y7q2gnv0rxyz6 --path transfer
```
## hash of the performed Transactions
```shell
["D5A1D503068928B4BD1CA0886FA797E3F9CCEEFAFC447A469092E61C2F979E50", "B41F189DF46FA3E4C47529EA659E686070AE7D2581B3E9D38A83F330A498DF9E", "D95184A53F73BE96BD034D7390E6B88C7E53788768D153B7412C2A02E7759AFF", "0BDB35FEA47D33424325FDB7BEEB488A10C3479BC52AF2324372C9964CC73D90", "FCC20195379DD22AF58FC0FD7704439DF11A08E48A622C895151B760DB5926BA"]
```
