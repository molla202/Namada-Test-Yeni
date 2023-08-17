# Namada Testnet Node Guide
![opengraph](https://user-images.githubusercontent.com/82613690/225137742-6d599592-9773-45c0-b9b5-09240b082d40.jpg)
 * [Topluluk kanalımız](https://t.me/corenodechat)<br>
 * [Topluluk Twitter](https://twitter.com/corenodeHQ)<br>
 * [Namada Website](https://www.namada.net/)<br>
 * [Namada Explorer](https://namada.explorers.guru/)<br>
 * [Discord](https://discord.gg/RrQ2S9zg)<br>
 * [Twitter](https://twitter.com/namada)<br>
 * [Resmi Dokuman](https://docs.namada.net/introduction/testnets)<br>
## NOT: ödül bilgisi var var ben öle diola :D hadi yine iyisiniz. değiştir yazmamışssak elleşmeyin.

##  Gereksinimler
| Bileşenler | Minimum Gereksinimler | 
| ------------ | ------------ |
| CPU |	4 |
| RAM	| 8 GB |
| Storage	| 250 GB SSD |

##  Sunucu Hazırlığı

```
sudo apt update && sudo apt upgrade -y
```
```
sudo apt install curl tar wget clang pkg-config git make libssl-dev libclang-dev libclang-12-dev -y
```
```
sudo apt install jq build-essential bsdmainutils ncdu gcc git-core chrony liblz4-tool -y
```
```
sudo apt install uidmap dbus-user-session protobuf-compiler unzip -y
```
```
apt install nodejs
```
##  Rust & Go & Protobuf yükleme

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
Not: > böle dicek 1 diceniz kurcanız
```
source $HOME/.cargo/env
```

```
sudo rm -rvf /usr/local/go/
wget https://golang.org/dl/go1.19.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.19.4.linux-amd64.tar.gz
rm go1.19.4.linux-amd64.tar.gz
echo "export PATH=\$PATH:/usr/local/go/bin" >>~/.profile
echo "export PATH=\$PATH:\$(go env GOPATH)/bin" >>~/.profile
source ~/.profile
go version
```

```
cd $HOME && rustup update
PROTOC_ZIP=protoc-23.3-linux-x86_64.zip
curl -OL https://github.com/protocolbuffers/protobuf/releases/download/v23.3/$PROTOC_ZIP
sudo unzip -o $PROTOC_ZIP -d /usr/local bin/protoc
sudo unzip -o $PROTOC_ZIP -d /usr/local 'include/*'
rm -f $PROTOC_ZIP

protoc --version
```

##  Ayarlar
```
sed -i '/public-testnet/d' "$HOME/.bash_profile"
sed -i '/NAMADA_TAG/d' "$HOME/.bash_profile"
sed -i '/WALLET_ADDRESS/d' "$HOME/.bash_profile"
sed -i '/CBFT/d' "$HOME/.bash_profile"
```
```
echo "export NAMADA_TAG=v0.21.1" >> ~/.bash_profile
echo "export CBFT=v0.37.2" >> ~/.bash_profile
echo "export CHAIN_ID=public-testnet-12.fedec12f3428" >> ~/.bash_profile
```

##  bir kullanıcı hesabı oluşturun
```
echo "export WALLET=wallet" >> ~/.bash_profile
```
👉Not: VALIDATORADI YAZILAN KISIMI DEĞİŞTİR!!
```
echo "export BASE_DIR=$HOME/.local/share/namada" >> ~/.bash_profile
echo "export VALIDATOR_ALIAS=VALIDATORADI" >> ~/.bash_profile

source ~/.bash_profile
```

##  NAMADA Yükle
```
cd $HOME && git clone https://github.com/anoma/namada && cd namada && git checkout $NAMADA_TAG
make build-release
```
```
cd $HOME && git clone https://github.com/cometbft/cometbft.git && cd cometbft && git checkout $CBFT
make install
```


```
cometbft version
namada --version
```
## Servis oluşturuyoruz.
```
sudo tee /etc/systemd/system/namadad.service > /dev/null <<EOF
[Unit]
Description=namada
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.local/share/namada
Environment=TM_LOG_LEVEL=p2p:none,pex:error
Environment=NAMADA_CMT_STDOUT=true
ExecStart=/usr/local/bin/namada node ledger run 
StandardOutput=syslog
StandardError=syslog
Restart=always
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
## Servis başlatıyoruz
```
sudo systemctl daemon-reload
sudo systemctl enable namadad
```
##  Ağa katılıp çalıştırıyoruz
```
cd $HOME && namada client utils join-network --chain-id $CHAIN_ID
sudo systemctl start namadad && sudo journalctl -u namadad -f -o cat 
```

##  Senkronize olana kadar bekleyin
```
curl -s localhost:26657/status | jq
```
👉false çıltısı aldıysanız devam edin.
##  Doğrulayıcı Hesabı Başlatın
```
cd $HOME
namada wallet address gen --alias $WALLET --unsafe-dont-encrypt
```
```
cd $HOME
namada client transfer \
  --source faucet \
  --target $WALLET \
  --token NAM \
  --amount 1000 \
  --gas-payer $WALLET \
  --signing-keys $WALLET
   
 ```

```
cd $HOME
namada client init-validator \
--alias $VALIDATOR_ALIAS \
--commission-rate 0.05 \
--max-commission-rate-change 0.01 \
--signing-keys $WALLET \
--gas-amount 100000000 \
--gas-token NAM \
--account-keys $WALLET \
--unsafe-dont-encrypt
```

##  Faucet alalım

```
cd $HOME
namada client transfer \
  --source faucet \
  --target $WALLET \
  --token NAM \
  --amount 1000 \
  --gas-payer $WALLET \
  --signing-keys $WALLET
   
 ```
 👉Not: İşlem başına musluktan maksimum 1000 NAM alınabilir, bu nedenle daha fazlasını elde etmek için bunu birden çok kez çalıştırın
 
 ##  Bakiyeni kontrol et
 ```
 namada client balance --owner $VALIDATOR_ALIAS --token NAM
 ```
## Epoch numarasını kontrol edin.
```
namada client epoch
```
👉Not: hangi epoch dayız bakalım.  2 3 epoch geçtikten sonra kendinize delege edin. altaki komutla.

## 12. Stake işlemi
 👉Not: miktar giriniz.--amount 1888 yazan kısım. bakiye kontol ederek fazla bi rakamda girebilrisiniz yada olduğu gibide bırakabilirsiniz faucetten 3 5 kere alın önceden.
 ```
namada client bond \
--validator $VALIDATOR_ALIAS \
--amount 1888 \
--signing-keys $WALLET \
--source $VALIDATOR_ALIAS
  ```

## Validator adreslerini kaydedin
👉Not: yukarıda ki işlemler bittikten 2 3 epoc sonra altaki işlemi yapın.
```
RAW_ADDRESS=`cat "$HOME/.local/share/namada/$CHAIN_ID/wallet.toml" | grep "address ="`
WALLET_ADDRESS=$(echo -e $RAW_ADDRESS | sed 's|.*=||' | sed -e 's/^ "//' | sed -e 's/"$//')
echo "export WALLET_ADDRESS=$WALLET_ADDRESS" >> ~/.bash_profile
source ~/.bash_profile
echo -e "\033[32m YOUR WALLET ADDRESS: \033[35m $WALLET_ADDRESS"
```
```
namada client epoch
namada client bonded-stake --validator $VALIDATOR_ALIAS
namada client bonds --validator $VALIDATOR_ALIAS
```
## Node Silme
```
cd $HOME && mkdir $HOME/namada_backup
cp -r $HOME/.local/share/namada/pre-genesis $HOME/namada_backup
cp -r .namada/pre-genesis $HOME/namada_backup
systemctl stop namadad && systemctl disable namadad
rm /etc/systemd/system/namada* -rf
rm $(which namada) -rf
rm /usr/local/bin/namada /usr/local/bin/namadac /usr/local/bin/namadan /usr/local/bin/namadaw /usr/local/bin/tendermint /usr/local/bin/cometbft -rf
rm $HOME/.namada* -rf
rm $HOME/.local/share/namada -rf
rm $HOME/namada -rf
rm $HOME/tendermint -rf
rm $HOME/cometbft -rf
```

