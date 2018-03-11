<template>
  <div>
    <div class="btn-wrapper">
      <Button class="button" :class="{'active':method=='transfer'}" @click="changMethod('transfer')">转账</Button>
      <Button class="button" :class="{'active':method=='receive'}" @click="changMethod('receive')">收入</Button>
    </div>
    <div class="filter-wrapper">
    </div>
    <div class="content-wrapper" v-if="method=='receive'">
      <div class="receive-wallet-wrapper">
        <div class="receive-wallet-selector">
        当前钱包：<i-select v-model="current_wallet.address" class="wallet-source" placeholder="选择钱包" @on-change="changeReceiveWallet">
          <Option v-for="item in wallet_list" :value="item.address" :key="item.address">{{ item.address }}</Option>
        </i-select>
        </div>
        <div class="receive-wallet-qrcode">
          二维码：<div class="qrcode" id="qrcode"></div>
        </div>
      </div>
    </div>
    <div class="content-wrapper" v-if="method=='transfer'">
      <div class="form-item">
        从<i-select v-model="current_wallet.address" class="wallet-source" placeholder="选择钱包" @on-change="changeTransferWallet">
            <Option v-for="item in wallet_list" :value="item.address" :key="item.address">{{ item.address }}</Option>
        </i-select>
      </div>
      <div class="form-item">
        转入<i-input v-model="target_address" placeholder="转入地址" class="wallet-target"></i-input>
      </div>
      <div class="gap clearfix"></div>
      <div class="form-item">
        数量
        <div class="transfer">
          <i-input-number v-model="transfer_token" class="transfer-token" :max="max" :min="min" :step="step">
          </i-input-number>
            <i-select v-model="token_address" slot="append" class="transfer-token-selector" placeholder="币种">
                <Option v-for="item in token_list" :value="item.value" :key="item.value">{{ item.label }}</Option>
            </i-select>
        </div>
      </div>
      <div class="form-item" v-show="token_address && current_wallet">
        持有
        <div class="have-token-group">
            <span class="have-token" v-text="current_wallet[token_address]"></span><span class="token" v-text="token"></span>
        </div>
      </div>
      <div class="result-wrapper">
        <div class="form-item">
          共计<span class="transfer-amount" v-text="transfer_token"  v-if="token"></span><span class="token"  v-text="token"></span>
          <Button class="button ready-to-transfer" size="large" @click="transfer">确定</Button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import _ from "lodash";
import async from "async";
import BigNumber from "bignumber.js";
import reportUtils from "../reportUtils";
import web3Utils from "../web3Utils";
import kjua from 'kjua'

export default {
  data() {
    return {
      method: "transfer",
      token_address: "",
      transfer_token: 0,
      token_list: [],
      wallet_list: [],
      current_wallet: {},
      target_address: "",
      qrcode:""
    };
  },
  computed: {
    token: function() {
      let _this = this,
        _token = _.find(this.token_list, { value: _this.token_address });
      return _token ? _token.label : "";
    },
    max: function() {
      return this.current_wallet
        ? Number.parseFloat(this.current_wallet[this.token_address])
        : 0;
    },
    min: function() {
      return 0;
    },
    step: function() {
      return 0.0001;
    }
  },
  mounted() {
    this.wallet_list = this.$root.globalData.wallet_list;

    if (this.$root.globalData.current_wallet) {
      this.current_wallet = this.$root.globalData.current_wallet;
    }
  },
  methods: {
    generateQRCode(text){
      let img_path = './assets/logo.png',
          _qrcode = document.querySelector("#qrcode");
      _qrcode.innerHTML = '';
      _qrcode.appendChild(kjua({text:text}));
    },
    changeReceiveWallet(address){
      let _this = this,
        current_wallet = _.cloneDeep(
          _.find(this.$root.globalData.wallet_list, ["address", address])
        );
      this.current_wallet = current_wallet;
      this.generateQRCode(address);
    },
    changeTransferWallet(address) {
      let _this = this,
        web3 = web3Utils.getWeb3(),
        current_wallet = _.cloneDeep(
          _.find(this.$root.globalData.wallet_list, ["address", address])
        );
      this.$Loading.start();
      web3Utils.setWebProvider(current_wallet.keystore);
      this.current_wallet = current_wallet;
      this.updateBalance(this.current_wallet);
      this.$Loading.finish();
    },
    updateBalance(wallet, resolve, reject) {
      var _this = this,
        web3 = web3Utils.getWeb3(),
        erc20tokens = web3Utils.getErc20Tokens(),
        _wallet = _.defaults({}, wallet),
        _token_list = [];

      _token_list.push({
        value: "ETH",
        label: "ETH"
      });
      _wallet["ETH"] = web3Utils.toRealAmount(
        web3.eth.getBalance(_wallet.address)
      );

      _.forEach(erc20tokens, token => {
        let balance = token.contract.balanceOf("" + _wallet.address);
        _token_list.push({
          value: token.address,
          label: token.symbol
        });
        _wallet[token.contract.address] = web3Utils.toRealAmount(
          balance,
          token.decimals
        );
      });
      _this.current_wallet = _.defaults(wallet, _wallet);
      _this.wallet_list[
        _.findIndex(_this.wallet_list, { address: wallet.address })
      ] = _.cloneDeep(_this.current_wallet);
      _this.token_list = _token_list;
    },
    changMethod(method) {
      this.method = method;
      this.current_wallet = {};
    },
    transfer() {
      var _this = this,
          text = [];
      if(!this.current_wallet || !this.current_wallet.address){
        text.push("未选择钱包")
      }
      if(!this.target_address){
        text.push("未填写转入地址")
      }
      if(!this.transfer_token){
        text.push("转入数量不可为0")
      }
      if(text.length){
        text.unshift("错误：")
        this.$Message.error(text.join(" "));
        return;
      }

      _this.$Loading.start();

      this.doTransfer()
        .then(txhash => {
          _this.$Loading.finish();
          _this.$Message.success(`提交成功：${txhash}`);
        })
        .catch(err => {
          _this.$Loading.error();
          _this.$Message.error("提交失败");
          console.log(err);
        });
    },
    doTransfer(resolve, reject) {
      var _this = this;
      return new Promise((resolve, reject) => {
        let web3 = web3Utils.getWeb3(),
          erc20tokens = web3Utils.getErc20Tokens(),
          fromAddr = this.current_wallet.address,
          toAddr = this.target_address,
          valueEth = this.transfer_token,
          value,
          gasPrice,
          gas;
        try {
          if (this.token_address === "ETH") {
            value = parseFloat(valueEth) * 1.0e18;
            gasPrice = 18000000000;
            gas = 50000;
            web3.eth.sendTransaction(
              {
                from: fromAddr,
                to: toAddr,
                value: value,
                gasPrice: gasPrice,
                gas: gas
              },
              function(err, txhash) {
                if (err) {
                  reject && reject(err);
                } else {
                  resolve && resolve(txhash);
                }
              }
            );
          } else {
            let erc20token = _.find(erc20tokens, {
                address: this.token_address
              }),
              contract = erc20token.contract,
              decimals = erc20token.decimals;

            value = new BigNumber(valueEth + "e+" + decimals);
            gasPrice = 21000000000;

            web3.eth.defaultAccount = fromAddr;

            gas = contract.transfer.estimateGas(toAddr, value.toString());

            contract.transfer(
              toAddr,
              value.toString(),
              {
                from: fromAddr,
                value: "0",
                gasPrice: gasPrice,
                gas: gas
              },
              function(err, result) {
                debugger;
                if (err) {
                  reject && reject(err);
                } else {
                  resolve && resolve(result);
                }
              }
            );
          }
        } catch (err) {
          reject && reject(err);
        }
      });
    },
    onWalletChange() {},
    onTokenChange(value) {
      this.token = value;
    }
  }
};
</script>

<style lang="less" scoped>
.content-wrapper {
  color: #fff;
  .form-item {
    display: inline-flex;
    align-items: flex-end;
    color: #ccc;
    font-size: 16px;
    width: 45%;
  }
  .wallet-source,
  .wallet-target,
  .transfer {
    font-size: 24px;
    color: #000;
  }
  .wallet-source,
  .wallet-target,
  .transfer,
  .token-amount {
    width: 80%;
  }
  .wallet-source {
    margin-left: 20px;
  }
  .wallet-target,
  .token-amount {
    margin-left: 5px;
  }
  .transfer {
    margin-left: 5px;
    .transfer-token {
      width: 248px;
    }
    .transfer-token-selector {
      width: 80px;
    }
  }
  .wallet-selector {
    font-size: 12px;
    text-decoration: underline;
    cursor: pointer;
  }
  .have-token-group {
    margin-left: 5px;
    color: #ccc;
    .have-token,
    .token {
      font-size: 16px;
      padding: 0 5px 0 0;
    }
  }
  .gap {
    clear: both;
    display: table;
    margin-top: 50px;
    width: 100%;
  }
  .result-wrapper {
    border-top: 1px solid rgba(110, 110, 110, 0.5);
    padding-top: 40px;
    margin-top: 50px;
    width: 100%;
    .ready-to-transfer {
      flex-grow: 0;
    }
    .form-item {
      width: 90%;
    }
    .transfer-amount,
    .amount-token {
      padding: 0 10px;
      font-size: 32px;
      color: #fff;
    }
    .token {
      flex-grow: 1;
    }
  }
  .receive-wallet-wrapper{
    font-size:16px;
    color:#ccc;
    .receive-wallet-qrcode{
      margin-top:30px;
      .qrcode{
        margin-left:100px;
      }        
    }
  }
}
</style>