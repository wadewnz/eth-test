<template>
  <div class="center">
    <div style="color:blue;font-weight:600;font-size:28px">Ethereum</div>
    <div v-if="!hasWallet" style="color:red;font-weight:600;font-size:22px">No Supported Wallets</div>
    <div v-if="verified" style="color:green;font-weight:600">Wallet Address Verified</div>
    <button type="button" v-if="hasWallet && !(account || accounts.length)" @click="connectClicked()">Connect to Wallet</button>
    <button type="button" v-if="account && !verified && allowSign" @click="signClicked()">Sign Message</button>
    <div v-if="wallets.length == 1">Wallet: {{ selWallet }}</div>
    <label for="selWallet" v-if="wallets.length > 1">Select Wallet</label>
    <select id="selWallet" v-if="wallets.length > 1" v-model="selWallet">
      <option v-for="(i, index) in wallets" :key="index" :value="i.info.name" :text="i.info.name"></option>
    </select>
    <label for="selAccount" v-if="accounts.length">Select Account</label>
    <select id="selAccount" v-if="accounts.length" v-model="selAccount">
      <option v-for="(i, index) in accounts" :key="index" :value="i.value" :text="i.text"></option>
    </select>
    <!-- <button type="button" v-if="accounts.length && !account"  @click="selectClicked()">Select Address</button> -->
    <div v-if="account" >Account: {{account}}</div>
    <div v-if="balance" >Balance: {{balance}} ETH</div>
    <div v-if="signature"  style="overflow-wrap: break-word">Signature: {{signature}}</div>
    <div v-if="recoveredAcccount" >Recovered Account: {{recoveredAcccount}}</div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onUnmounted, onMounted } from 'vue'
import { recover } from 'web3-eth-accounts'
//import { MetaMaskSDK, MetaMaskSDKOptions } from '@metamask/sdk'

interface AccountData {
  value: string
  text: string
}

interface EIP1193Provider {
  request: (payload: {
    method: string;
    params?: unknown[] | object
  }) => Promise<unknown>
  enable(): void
  on(eventName: string, callback: (...args: any[]) => void): void
  removeListener(eventName: string, callback: (...args: any[]) => void): void
}
/*
type EIP1193Provider = {
    isMetaMask?: boolean;
    isStatus?: boolean;
    host?: string;
    path?: string;
    sendAsync?: (request: { method: string, params?: Array<any> }, callback: (error: any, response: any) => void) => void
    send?: (request: { method: string, params?: Array<any> }, callback: (error: any, response: any) => void) => void
    request?: (request: { method: string, params?: Array<any> }) => Promise<any>
}
*/

interface EIP6963ProviderInfo {
  uuid: string
  name: string
  icon: string
  rdns?: string
}

interface EIP6963ProviderDetail {
  info: EIP6963ProviderInfo
  provider: EIP1193Provider
}

interface EIP6963AnnounceProviderEvent extends Event {
  detail: EIP6963ProviderDetail
}

const hasWallet = ref(false)
const hasConnected = ref(false)
const account = ref('')
const balance = ref(0)
const verified = ref(false)
const recoveredAcccount = ref('')
const signature = ref<string | undefined>('')
const selAccount = ref('')
const accounts = ref<AccountData[]>([])
const allowSign = ref(true)
const wallets = ref<EIP6963ProviderDetail[]>([])
const selWallet = ref('')

//const options = { preferDesktop: true }
//const MMSDK = new MetaMaskSDK(options as MetaMaskSDKOptions)
//const ethereum = MMSDK.getProvider() //(window as any).ethereum
let ethereum: EIP1193Provider|undefined = (window as any).ethereum

hasWallet.value = !!ethereum
console.log('Has Ethereum: ' + hasWallet.value)

const handleAnnounceProvider = ((evt: EIP6963AnnounceProviderEvent) => {
    console.log(`ProviderName:${evt.detail.info.name} Uuid:${evt.detail.info.uuid}`)
    //  if (evt.detail.info.name !== 'MetaMask') return
    wallets.value.push(evt.detail)
    if (!selWallet.value || evt.detail.info.name === 'MetaMask') {
      selWallet.value = evt.detail.info.name
    }
    hasWallet.value = true
  }) as EventListener


onMounted(() => {
  console.log('onMounted')
  // The DApp listens to announced providers
  window.addEventListener('eip6963:announceProvider', handleAnnounceProvider)
  // The DApp dispatches a request event which will be heard by 
  // Wallets' code that had run earlier
  window.dispatchEvent(new Event("eip6963:requestProvider"))
})

onUnmounted(() => {
  console.log('onUnmounted')
  window.removeEventListener('eip6963:announceProvider', handleAnnounceProvider)
  wallets.value = []
  if (ethereum) {
    ethereum.removeListener('accountsChanged', handleAccountsChanged)
  }
})
    
async function connectClicked(): Promise<void> {
  console.log(`connectClicked: hasWallet=${hasWallet.value}`)
  if (!hasWallet.value) return
  try {
    // if (!ethereum.isConnected()) {
      // await ethereum.enable()
    // }

    console.log('Connected')    

    hasConnected.value = true
    if (account.value || !ethereum) {
      return
    }
    ethereum.on('accountsChanged', handleAccountsChanged)
    const current_accounts = await ethereum.request({ method: 'eth_accounts'})
    handleAccountsChanged(current_accounts as Array<string>)

  } catch (error) {
    console.error(error)
  }
}

async function handleAccountsChanged(current_accounts: Array<string>) {
  try {
    console.log('Accounts:', current_accounts)
    if (!ethereum) return
    const data = current_accounts?.length ? current_accounts : await ethereum.request({ method: 'eth_requestAccounts' }) as any
    if (!(current_accounts as any)?.length) {
      console.log('requestAccounts', data)
    }
    // const permissions = await ethereum.request({ method: 'wallet_getPermissions'})
    // console.log('Permissions:', permissions)
    if (data.length === 0) {
      return;
    }
    if (data.length == 1) {
      account.value = data[0]
      return selectClicked()
    }
    selAccount.value = data[0]
    accounts.value = data.map((a: string) => { return {value: a, text: a} })
  } catch (error) {
    console.error(error)
  }
}

async function selectClicked(): Promise<void> {
  try {
    if (!ethereum) return
    if (!account.value) {
      account.value = selAccount.value
    }
        
    console.log("Account: " + account.value);
    if (!account.value) {
      return;
    }
    let balanceData:any = await ethereum.request({ method: 'eth_getBalance', params: [account.value, 'latest']})
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
  console.log(`hasWallet=${hasWallet.value} Account=${account.value}`)
  if (!ethereum || !hasWallet.value || !account.value) return
  try {
    const message = 'Test message for signing'
    console.log("hasConnected=" + hasConnected.value + " message=" + message)
    const promise = hasConnected ? Promise.resolve() : ethereum.enable()
    await promise
    console.log('Connected')
    if (!hasConnected.value) {
      hasConnected.value = true
    }
    (signature.value as any) = await ethereum.request({ method: 'personal_sign', params: [message, account.value]})
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

watch(selWallet, name => {
  console.log(`Selected Wallet Changed - Name:${name}`)
  if (ethereum) {
    ethereum.removeListener('accountsChanged', handleAccountsChanged)
  }
  ethereum = wallets.value.find(w => w.info.name === name)?.provider
  if (!ethereum) {
    console.log('  Not found')
  }
  selAccount.value = ''
  account.value = ''
  accounts.value = []  
  //hasConnected.value = false
  recoveredAcccount.value = ''
  signature.value = ''
  verified.value = false
  balance.value = 0
  if (hasConnected.value) {
    connectClicked()
  }
})

watch(selAccount, a => {
  console.log(`Selected Account Changed - Account:${a}`)
  account.value = a
  if (account.value) {
    selectClicked()
  }
})

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
