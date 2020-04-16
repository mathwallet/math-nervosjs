<template>
  <v-app>
    <v-app-bar
      app
      color="primary"
      dark
    >
      <v-spacer></v-spacer>
      <div class="d-flex align-center">
         <h3> Nervos Samples </h3>
      </div>
      <v-spacer></v-spacer>
    </v-app-bar>

    <v-content>
      <v-card
        class="mx-auto mt-6"
        max-width="600"
        outlined
      >
        <v-list-item three-line>
          <v-list-item-content>
            <v-list-item-title class="title mb-1">使用 mathwallet 插件登录</v-list-item-title>
            <v-list-item-subtitle>{{account?`${account.name}[${account.address}]`:""}}</v-list-item-subtitle>
            <v-list-item-subtitle>{{account?account.root_pub:""}}</v-list-item-subtitle>
          </v-list-item-content>
        </v-list-item>

        <v-card-actions>
          <v-btn color="primary" @click="login" v-if="!account">登录</v-btn>
          <v-btn color="error" @click="logout" v-else>注销</v-btn>
        </v-card-actions>
      </v-card>
      <v-card
        class="mx-auto mt-6"
        max-width="600"
        outlined
      >
        <v-list-item three-line>
          <v-list-item-content>
            <v-text-field placeholder="请输入收款地址" :value="toAddress"></v-text-field>
          </v-list-item-content>
        </v-list-item>

        <v-card-actions>
          <v-btn color="error" @click="transfer">转账</v-btn>
        </v-card-actions>
      </v-card>
    </v-content>
  </v-app>
</template>

<script>
import CKB from "@nervosnetwork/ckb-sdk-core";
import { pubkeyToAddress, privateKeyToAddress, privateKeyToPublicKey, hexToBytes, bytesToHex, blake160, scriptToHash, JSBI} from "@nervosnetwork/ckb-sdk-utils";
import * as bip32 from "bip32";

export default {
  name: 'App',
  data() {
    return {
      account:null,
      toAddress:"ckb1qyqrkrum6vhye7lnldczzweasgv3y5p46tys89y8jr"
    }
  },
  methods: {
    /* 
      扩展公钥生成子地址或找零地址方法
      // 子地址 1
      const publicKey1 = bip32.fromBase58(xpub).derivePath("0/0").publicKey;
      console.log(pubkeyToAddress(publicKey1, { prefix:"ckb" }));
      // 找零地址 1
      const changePublicKey1 = bip32.fromBase58(xpub).derivePath("1/0").publicKey;
      console.log(pubkeyToAddress(changePublicKey1, { prefix:"ckb" }));
    */
    publicKeyByPath(xpub,path){
      return bytesToHex(bip32.fromBase58(xpub).derivePath(path).publicKey);
    },
    login(){
      if (!window.ckb) {
        alert("Please install MathWallet first!");
        return;
      }
      /* 
      // account 结构 => 
      {
        name:"test1",
        address:"ckb1qyqwguzpkxr5rjrtwhdkrjhthkqr4rfg609s4qv70z",
        xpub:"xpub6Cm44fNM6B9UeFh2Mrs3SvLXrDLCbKMsvLNR7XSZZeW4ujKhAiUdE91eNNneHvhberLchpz47Q5ZdND3cV5GX3X7vkjbLrnRDCf6mPEjFsT
      }
      */
      window.ckb.getAccount().then(account => {
        console.log(account);
        
        this.account = account;
      });
    },
    logout(){
      window.ckb.forgetAccount().then(res => {
        this.account = null;
      });
    },
    async transfer(){
      if (!this.account) {
        alert("Please log in first!");
        return;
      }

      // 当前账户的第一个子地址作为转出账户
      const derive_paths = ["0/0"];

      const publicKey = this.publicKeyByPath(this.account.xpub,derive_paths[0]);
      const address = pubkeyToAddress(publicKey, { prefix:"ckb" });

      // 初始化 RPC 节点
      const http = require('http')
      const httpAgent = new http.Agent({ keepAlive: true })

      const nodeUrl = "http://localhost:8114";
      const ckb = new CKB(nodeUrl);
      ckb.rpc.setNode({ httpAgent })

      const secp256k1Dep = await ckb.loadSecp256k1Dep();

      // 转出账户 lockScript
      const publicKeyHash1 = `0x${blake160(publicKey, 'hex')}`
      const lockScript = {
          hashType: secp256k1Dep.hashType,
          codeHash: secp256k1Dep.codeHash,
          args: publicKeyHash1,
      };

      // Nervos-JS-SDK中获取unspentCells的测试方法，如下
      // const lockHash = scriptToHash(lockScript);
      // const unspentCells = await ckb.loadCells({lockHash});
      
      // 手动组装 Cells（UTXO）测试使用
      const capacity = JSBI.BigInt("11899998290");
      const unspentCells = [
        {
          blockHash:
            "0x5246612578f3d2c484c586fde900c023b231223793ebc2b756053891d73d443a",   // 区块对应的hash
          lock: lockScript,
          outPoint: {
            txHash:
              "0x456b9b9073ba2f51d8e300713d454cf5f66a1ccbfae555ec1146dc8bf941f4bf",  //交易hash
            index: "0x0",
          },
          capacity: `0x${capacity.toString(16)}`,

          dataHash:
            "0x0000000000000000000000000000000000000000000000000000000000000000",
          status: "live",
          type: null,
        }
      ];
      
      const fee = JSBI.BigInt("1100");
      
      const rawTx = ckb.generateRawTransaction({
        fromAddress: address,
        toAddress: this.toAddress,
        capacity: `0x${JSBI.subtract(capacity,fee).toString(16)}`,
        fee: `0x${fee.toString(16)}`,
        safeMode: true,
        cells: unspentCells,
        deps: ckb.config.secp256k1Dep,
        changeThreshold: `0x${JSBI.BigInt("0").toString(16)}`
      })
      rawTx.witnesses = rawTx.inputs.map(() => '0x')
      rawTx.witnesses[0] = {
        lock: '',
        inputType: '',
        outputType: ''
      }
      // ckb-sdk-js 签名方法
      // const signedTx = ckb.signTransaction(privateKey)(rawTx)

      // 替换签名方法，使用麦子插件签名
      // derive_paths 签名账户对应的子地址或找零地址路径(eg 0/0,1/0 ...)

      const signedTx = await window.ckb.signTransaction(derive_paths,rawTx);
      // const signedTx = await window.ckb.signTransaction(derive_paths,rawTx,unspentCells);
      const realTxHash = await ckb.rpc.sendTransaction(signedTx)
      console.log(`real tx hash: ${realTxHash}`)


      // 更多 ckb-sdk-js 示例
      // https://github.com/nervosnetwork/ckb-sdk-js/blob/develop/packages/ckb-sdk-core/examples
    }
  },
};
</script>
