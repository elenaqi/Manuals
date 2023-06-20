## Uptick Node Install
```
# Update & upgrade
sudo apt update && sudo apt upgrade -y

# Install the required packages
sudo apt-get install nano mc git gcc g++ make curl build-essential tmux chrony wget jq yarn -y

# Install GO
wget -O go1.18.3.linux-amd64.tar.gz https://go.dev/dl/go1.18.3.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.3.linux-amd64.tar.gz && rm go1.18.3.linux-amd64.tar.gz
cat <<'EOF' >> $HOME/.bash_profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
. $HOME/.bash_profile
cp /usr/local/go/bin/go /usr/bin

# version
go version
```
### Install the [latest version](https://github.com/UptickNetwork/uptick/blob/main/release) of Uptick
```
mkdir /root/uptick-bin
cd /root/uptick-bin
wget https://github.com/UptickNetwork/uptick/blob/main/release/uptick-v0.2.7.tar.gz?raw=true
tar -zxvf uptick-v0.2.7.tar.gz?raw=true
chmod +x uptick-v0.2.7/linux/uptickd
/root/uptick-bin/uptickd version --long | head
cp /root/uptick-bin/uptick-v0.2.7/linux/uptickd /usr/local/bin
uptickd version
```
### Add variables
- `<MONIKER>` - Your node name
- `<WALLET>` - Your wallet name
```
UP_NODENAME=<MONIKER>
UP_WALLET=<WALLET>
UP_CHAIN=uptick_7000-1
echo 'export UP_NODENAME='\"${UP_NODENAME}\" >> $HOME/.bash_profile
echo 'export UP_WALLET='\"${UP_WALLET}\" >> $HOME/.bash_profile
echo 'export UP_CHAIN='\"${UP_CHAIN}\" >> $HOME/.bash_profile
source $HOME/.bash_profile
echo $UP_NODENAME $UP_WALLET $UP_CHAIN
```
### Init your config files
```
uptickd init $UP_NODENAME --chain-id $UP_CHAIN
```
### Download `genesis.json` and `addrbook.json`
```
curl -s  https://raw.githubusercontent.com/UptickNetwork/uptick-testnet/main/uptick_7000-1/genesis.json > ~/.uptickd/config/genesis.json
wget -qO $HOME/.uptickd/config/addrbook.json "https://raw.githubusercontent.com/AlexToTheSun/Validator_Activity/main/Testnet-guides/Uptick/addrbook.json"
```
#### Config your node
 Seeds and peers
```
peers="a9bb3d5c36cf62a280c13f3e37c93a4b17707eab@142.132.196.251:46656,eecdfb17919e59f36e5ae6cec2c98eeeac05c0f2@peer0.testnet.uptick.network:26656,178727600b61c055d9b594995e845ee9af08aa72@peer1.testnet.uptick.network:26656"
seeds="61f9e5839cd2c56610af3edd8c3e769502a3a439@seed0.testnet.uptick.network:26656"
sed -i.bak -e "s/^seeds *=.*/seeds = \"$seeds\"/; s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.uptickd/config/config.toml
```
 Minimum gas prices
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0auptick\"/" $HOME/.uptickd/config/app.toml
```


### Create a service file
```
sudo tee <<EOF >/dev/null /etc/systemd/system/uptickd.service
[Unit]
Description=Uptick Cosmos daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$(which uptickd) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
### Start synchronization from scratch
```
uptickd tendermint unsafe-reset-all --home $HOME/.uptickd
sudo systemctl daemon-reload
sudo systemctl enable uptickd
sudo systemctl restart uptickd

# Logs and status
sudo journalctl -u uptickd -f -o cat
curl localhost:26657/status | jq
uptickd status 2>&1 | jq .SyncInfo
```
Wait until full synchronization.

### Upgrade your node to a Validator
Before you Fund your wallet

```
uptickd tx staking create-validator \
--chain-id $UP_CHAIN \
--commission-rate=0.07 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.1 \
--min-self-delegation="1" \
--amount=4900000000000000000auptick \
--pubkey $(uptickd tendermint show-validator) \
--moniker $UP_NODENAME \
--identity=""
--details=""
--website=""
--from=$UP_WALLET \
--gas="auto"
```
Explorer: https://explorer.testnet.uptick.network/uptick-network-testnet/staking

### Useful commands
```
# Logs and status
sudo journalctl -u uptickd -f -o cat
curl localhost:26657/status | jq
uptickd status 2>&1 | jq .SyncInfo

# Find out your wallet:
uptickd keys show $UP_WALLET -a

# Your validator:
uptickd keys show $UP_WALLET --bech val -a

# Information about your validator:
uptickd query staking validator $(uptickd keys show $UP_WALLET --bech val -a)

# For service file
sudo systemctl daemon-reload
sudo systemctl enable uptickd
sudo systemctl restart uptickd
sudo systemctl status uptickd
sudo systemctl stop uptickd
sudo systemctl disable uptickd
```
