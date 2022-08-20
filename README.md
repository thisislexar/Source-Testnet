<h1 align="center">Source Protocol Testnet Kurulumu

## Merhabalar, Source Protocol için testnet kurulumunu gerçekleştireceğiz. Sorularınız olursa: [LossNode Chat](https://t.me/LossNode)

![image](https://user-images.githubusercontent.com/101462877/185739528-e73647e2-9ccb-4c80-a068-cec3e04fcac7.png)

## Minimum sistem gereksinimleri:
NODE TİPİ | CPU     | RAM      | SSD     |
| ------------- | ------------- | ------------- | -------- |
| Testnet | 4          | 8         | 160  |


## Source Protocol için önemli linkler:
- [Website](https://www.sourceprotocol.io/)
- [Explorer](https://explorer.testnet.sourceprotocol.io/source)
- [Twitter](https://twitter.com/SourceProtocol_/)
- [Discord](https://discord.gg/wUeGfeUt5X)

# Başlayalım. Öncelikle sunucumuza gerekli güncellemeleri ve kurulumları yapıyoruz.

```
sudo apt update && sudo apt upgrade -y && \
sudo apt install curl tar wget clang pkg-config libssl-dev libleveldb-dev jq build-essential bsdmainutils git make ncdu htop screen unzip bc fail2ban htop -y
```

# Go yükleyelim.
```
ver="1.18.3" && \
cd $HOME && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

# Binary'yi indirelim.
```
git clone -b testnet https://github.com/Source-Protocol-Cosmos/source.git
cd ~/source
make install
```

# Moniker initialisation'ını yapalım.
```
sourced init <MONIKERADI> --chain-id=sourcechain-testnet
```

Bu kısımda `<MONIKERADI>` yerine kendi validator isminizi yazın.

# Cüzdan oluşturalım.
```
sourced keys add <CUZDANIADI>
```
Bu kısımda `<CUZDANIADI>` yerine kendi cüzdan isminizi yazın.

# [Discord](https://discord.gg/wUeGfeUt5X)'a gidip faucet alalım.
![image](https://user-images.githubusercontent.com/101462877/185739228-cdb992c5-e3c3-4238-a370-407c5865bbb2.png)


# Genesis dosyasını indirelim.
```
curl -s  https://raw.githubusercontent.com/Source-Protocol-Cosmos/testnets/master/sourcechain-testnet/genesis.json > ~/.source/config/genesis.json
```

# Seeds ve peers düzenleyelim.
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0usource\"/;" ~/.source/config/app.toml
sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.source/config/config.toml
external_address=$(wget -qO- eth0.me) 
sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.source/config/config.toml

peers="6ca675f9d949d5c9afc8849adf7b39bc7fccf74f@164.92.98.17:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.source/config/config.toml

seeds=""
sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.source/config/config.toml

sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 100/g' $HOME/.source/config/config.toml
sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 100/g' $HOME/.source/config/config.toml
```

# Addrbook dosyasını indirelim.
```
wget -O $HOME/.source/config/addrbook.json "https://raw.githubusercontent.com/obajay/nodes-Guides/main/Source/addrbook.json"
```


# Servis dosyası oluşturalım.
```
sudo tee /etc/systemd/system/sourced.service > /dev/null <<EOF
[Unit]
Description=source
After=network-online.target

[Service]
User=$USER
ExecStart=$(which sourced) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
# Snapshot kullanalım. Kullanmazsak çok ileride olduğu için güncel bloğu yakalamanız uzun sürecektir. Ayrıca node fazla yer kaplayacaktır.
```
sudo systemctl stop sourced
rm -rf $HOME/.source/data/
mkdir $HOME/.source/data/
cd $HOME
wget http://116.202.236.115:8000/sourcedata.tar.gz
tar -C $HOME/ -zxvf sourcedata.tar.gz --strip-components 1
wget -O $HOME/.source/data/priv_validator_state.json "https://raw.githubusercontent.com/obajay/StateSync-snapshots/main/priv_validator_state.json"
cd && cat .source/data/priv_validator_state.json
{
  "height": "0",
  "round": 0,
  "step": 0
}
cd $HOME
rm sourcedata.tar.gz
sudo systemctl daemon-reload && \
sudo systemctl enable sourced && \
systemctl restart systemd-journald.service && \
sudo systemctl restart sourced && \
sudo journalctl -u sourced -f -o cat
```

# Bir süre geçtikten sonra güncel bloğu yakalayıp yakalamadığımıza bakalım.
```
sourced status 2>&1 | jq .SyncInfo
```
![image](https://user-images.githubusercontent.com/101462877/185738944-d4228906-1580-4a1c-a1d6-db09fa371912.png)

Eğer yakaladıysanız, görseldeki yer `false` olacaktır. Hala `true` ise yakalamasını bekleyin. 

# Güncel bloğu yakaladıktan sonra validatorümüzü oluşturalım.
```
sourced tx staking create-validator \
--amount=990000usource \
--pubkey=$(sourced tendermint show-validator) \
--moniker=<MONIKERADI> \
--chain-id sourcechain-testnet \
--commission-max-change-rate "0.01" \
--commission-max-rate "0.2" \
--commission-rate "0.07" \
--min-self-delegation "1" \
--fees=500usource \
--from=<CÜZDANADI> \
--website="http://linktr.ee/LossNode" \
--details="Testing the Source" \
-y
```
Bu kısımda;
- `<MONIKERADI>` yerine kendi validator isminizi yazın.
- `<CUZDANIADI>` yerine kendi cüzdan isminizi yazın.

# [Discord](https://discord.gg/wUeGfeUt5X)'a tekrar gidip `validator-role-request` kanalına [Explorer](https://explorer.testnet.sourceprotocol.io/source) linkimizi atalım ve validator rolü alalım.
![image](https://user-images.githubusercontent.com/101462877/185739328-6e91d56c-550a-474d-b568-ca55140455f0.png)


# Kurulum bu kadardı, validatör olarak kullanabileceğiniz bazı komutları aşağıya bırakıyorum.

# Validatörünüze token delege etmek için kod:

```
sourced tx staking delegate <SOURCE_VALOPER_ADRESI> <MIKTAR>usource --from=<CUZDANIADI> --chain-id=sourcechain-testnet --fees=500usource -y
```

# Logları kontrol etmek için kod:

```
journalctl -u sourced -f -o cat
```

# Senkronize durumunu kontrol etmek için kod:

```
sourced status 2>&1 | jq .SyncInfo
```
