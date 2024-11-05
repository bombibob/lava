Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```
**Clone project repository**
```
cd && rm -rf lava
git clone https://github.com/lavanet/lava
cd lava
git checkout v3.1.0
```

**Build binary**
```
export LAVA_BINARY=lavad
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.lava/cosmovisor/genesis/bin
sudo ln -s $HOME/.lava/cosmovisor/genesis $HOME/.lava/cosmovisor/current -f
sudo ln -s $HOME/.lava/cosmovisor/current/bin/lavad /usr/local/bin/lavad -f
```

**Move binary to cosmovisor directory**
```
mv $(which lavad) $HOME/.lava/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
lavad config chain-id lava-testnet-2
lavad config keyring-backend test
lavad config node tcp://localhost:19957
```

**Initialize the node**
```
lavad init "Your Node Name" --chain-id lava-testnet-2
```

**Download genesis and addrbook files**
```
curl -L https://snapshots-testnet.nodejumper.io/lava/genesis.json > $HOME/.lava/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/lava/addrbook.json > $HOME/.lava/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "3a445bfdbe2d0c8ee82461633aa3af31bc2b4dc0@prod-pnet-seed-node.lavanet.xyz:26656,e593c7a9ca61f5616119d6beb5bd8ef5dd28d62d@prod-pnet-seed-node2.lavanet.xyz:26656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@testnet-seeds.polkachu.com:19956,eb7832932626c1c636d16e0beb49e0e4498fbd5e@lava-testnet-seed.itrocket.net:20656"|' $HOME/.lava/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.000001ulava"|' $HOME/.lava/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
$HOME/.lava/config/app.toml
```
**Change ports**
```
sed -i -e "s%:1317%:19917%; s%:8080%:19980%; s%:9090%:19990%; s%:9091%:19991%; s%:8545%:19945%; s%:8546%:19946%; s%:6065%:19965%" $HOME/.lava/config/app.toml
sed -i -e "s%:26658%:19958%; s%:26657%:19957%; s%:6060%:19960%; s%:26656%:19956%; s%:26660%:19961%" $HOME/.lava/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots-testnet.nodejumper.io/lava/lava-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.lava"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

**Create a service**
```
sudo tee /etc/systemd/system/lava.service > /dev/null << EOF
[Unit]
Description=Lava node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.lava
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.lava"
Environment="DAEMON_NAME=lavad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable lava.service
```

**Start the service and check the logs**
```
sudo systemctl start lava.service
sudo journalctl -u lava.service -f --no-hostname -o cat
Create Validator
```

**create wallet**
```
lavad keys add wallet
```

**console output**
```
- name: wallet
  type: local
  address: lava@1us3tv59r3wz57ydjafkzgpy0pccae2a2e4k5en
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"Auq9WzVEs5pCoZgr2WctjI7fU+lJCH0I3r6GC1oa0tc0"}'
  mnemonic: ""
```

**!!! SAVE SEED PHRASE (example)**
```
kite upset hip dirt pet winter thunder slice parent flag sand express suffer chest custom pencil mother bargain remember patient other curve cancel sweet
```

**SAVE PRIVATE VALIDATOR KEY**
cat $HOME/.lava/config/priv_validator_key.json

**wait util the node is synced, should return FALSE**
```
lavad status 2>&1 | jq .SyncInfo.catching_up
```
**faucet some tokens with the command below or ask in discord, if the command doesn't work**
```
curl -X POST -d '{"address": "YOUR_WALLET_ADDRESS", "coins": ["10000000ulava"]}' https://faucet-api.lavanet.xyz/faucet/
```

**verify the balance**
```
lavad q bank balances $(lavad keys show wallet -a)
```

**console output**
```
balances:
 amount: "10000000"
denom: ulava
```

**create validator**
```
lavad tx staking create-validator \
--amount=9000000ulava \
--pubkey=$(lavad tendermint show-validator) \
--moniker="$NODE_MONIKER" \
--chain-id=lava-testnet-2 \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.05 \
--min-self-delegation=1 \
--fees=10000ulava \
--from=wallet \
-y
```

**make sure you see the validator details**
```
lavad q staking validator $(lavad keys show wallet --bech val -a)
Secure Server Setup (Optional)
```

**generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE**
```
ssh-keygen -t rsa
```

**save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY**
```
cat ~/.ssh/id_rsa.pub
```

**upgrade system packages**
```
sudo apt update
sudo apt upgrade -y
```

**add new admin user**
```
sudo adduser admin --disabled-password -q
```

**upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above**
```
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
