<template>
  <div class="center">
    <div style="color:blue;font-weight:600;font-size:28px">Ethereum</div>
    <div v-if="!hasWallet" style="color:red;font-weight:600;font-size:22px">No Supported Wallets</div>
    <div v-if="verified" style="color:green;font-weight:600">Wallet Address Verified</div>
    <button type="button" v-if="hasWallet && !(account || accounts.length)"   @click="connectClicked()">Connect to Wallet</button>
    <button type="button" v-if="account && !verified && allowSign" @click="signClicked()">Sign Message</button>
    <select v-if="accounts.length" v-model="selAccount">
      <option v-for="(i, index) in accounts" :key="index" :value="i.value" :text="i.text"></option>
    </select>
    <button type="button" v-if="accounts.length && !account"  @click="selectClicked()">Select</button>
    <div v-if="account" >Account: {{account}}</div>
    <div v-if="balance" >Balance: {{balance}} ETH</div>
    <div v-if="signature"  style="overflow-wrap: break-word">Signature: {{signature}}</div>
    <div v-if="recoveredAcccount" >Recovered Account: {{recoveredAcccount}}</div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { recover } from 'web3-eth-accounts'

interface AccountData {
  value: string
  text: string
}

const hasWallet = ref(false)
const hasConnected = ref(false)
const account = ref('')
const balance = ref(0)
const verified = ref(false)
const recoveredAcccount = ref('')
const signature = ref('')
const selAccount = ref('')
const accounts = ref<AccountData[]>([])
const allowSign = ref(true)

const ethereum = (window as any).ethereum

hasWallet.value = !!ethereum
console.log('Has Ethereum: ' + hasWallet.value)
    
async function connectClicked(): Promise<void> {
  console.log(`connectClicked: hasWallet=${hasWallet.value}`)
  if (!hasWallet) return
  try {
    await ethereum.enable()
    console.log('Connected')

    hasConnected.value = true
    if (account.value) {
      return
    }
    const data = await ethereum.request({ method: 'eth_requestAccounts' })
    console.log(data)
    if (data.length === 0) {
      return;
    }
    if (data.length == 1) {
      account.value = data[0]
      return selectClicked()
    }
    selAccount.value = data[0];
    accounts.value = data.map((a: string) => { return {value: a, text: a} })
  } catch (error) {
    console.error(error)
  }
}
async function selectClicked(): Promise<void> {
  try {
    if (!account.value) {
      account.value = selAccount.value
    }
        
    console.log("Account: " + account.value);
    if (!account.value) {
      return;
    }
    let balanceData = await ethereum.request({ method: 'eth_getBalance', params: [account.value, 'latest']})
    //web3.eth.getBalance(account.value)
    console.log("Raw Balance: ", balanceData)
    const strBalance = '000000000000000000' + BigInt(balanceData).toString(10)
    const ethBalance = Number(strBalance.slice(0, -18) + '.' + strBalance.slice(-18))
    console.log(`Balance: ${ethBalance} ETH`)
    balance.value = ethBalance
  } catch (error) {
    console.error(error)
  }
}

async function signClicked(): Promise<void> {
  console.log(`hasWallet=${hasWallet} Account=${account}`)
  if (!hasWallet || !account) return
  try {
    const message = 'Test message for signing'
    console.log("hasConnected=" + hasConnected.value + " message=" + message)
    const promise = hasConnected ? Promise.resolve() : ethereum.enable()
    await promise
    console.log('Connected')
    if (!hasConnected.value) {
      hasConnected.value = true
    }
    //signature.value = await web3.eth.personal.sign(message, account.value, '')
    signature.value = await ethereum.request({ method: 'personal_sign', params: [message, account.value]}) // web3.eth.personal.sign(message, account.value, '')
    console.log('Signature: ' + signature.value)

    const recovered = recover(message, signature.value)
    console.log('Recovered Address: ' + recovered)
    recoveredAcccount.value = recovered
    verified.value = recovered.toLowerCase() === account.value.toLowerCase()
  }
  catch (error) {
    console.error("Signing", error)
  }
}
</script>

<style scoped>

select {
  min-width: 200px;
  height: 32px;
}

.center {
  display: flex;
  flex-direction: column;
  align-items: center;
  align-content: center;
  column-count: 1;
  row-gap: 8px;
}
</style>
