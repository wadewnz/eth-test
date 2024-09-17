<template>
  <div class="center">
    <div style="color:blue;font-weight:600;font-size:28px">Ethereum</div>
    <div v-if="!hasWallet" style="color:red;font-weight:600;font-size:22px">No Supported Wallets</div>
    <div v-if="verified" style="color:green;font-weight:600">Wallet Address Verified</div>
    <button type="button" v-if="hasWallet && !(account || accounts.length)" @click="connectClicked">Connect to Wallet</button>
    <button type="button" v-if="account && allowSign" @click="signClicked">Sign Message</button>
    <button type="button" v-if="account && allowSign" @click="signTypedDataClicked">Sign Type Data</button>
    <button type="button" v-if="account && allowSign" @click="transferWithData">Transfer With Data</button>
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
import { recover  } from 'web3-eth-accounts'
import { recoverTypedSignature, SignTypedDataVersion } from '@metamask/eth-sig-util'
//import { MetaMaskSDK, MetaMaskSDKOptions } from '@metamask/sdk'
import { Buffer } from 'buffer';
import { ethers } from 'ethers';

// @ts-ignore
window.Buffer = Buffer;

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

async function signTypedDataClicked(): Promise<void> {
  console.log(`hasWallet=${hasWallet.value} Account=${account.value}`)
  if (!ethereum || !hasWallet.value || !account.value) return
  try {

    const chainId = 1;
    // More info on typed data: https://docs.metamask.io/guide/signing-data.html#sign-typed-data-v4
  const typedData = JSON.stringify({
    domain: {
      chainId,
      name: "Example App",
      verifyingContract: "0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC",
      version: "1",
    },

    message: {
      prompt: "Welcome! In order to authenticate to this website, sign this request and your public address will be sent to the server in a verifiable way.",
      createdAt: `${Date.now()}`,
    },
    primaryType: 'AuthRequest',
    types: {
      EIP712Domain: [
        { name: 'name', type: 'string' },
        { name: 'version', type: 'string' },
        { name: 'chainId', type: 'uint256' },
        { name: 'verifyingContract', type: 'address' },
      ],
      AuthRequest: [
        { name: 'prompt', type: 'string' },
        { name: 'createdAt', type: 'uint256' },
      ],
    },
  });


    const params = [account.value, typedData ]
    const method = "eth_signTypedData_v4"

    console.log("hasConnected=" + hasConnected.value)
    const promise = hasConnected ? Promise.resolve() : ethereum.enable()
    await promise
    console.log('Connected')
    if (!hasConnected.value) {
      hasConnected.value = true
    }
    (signature.value as any) = await ethereum.request({ method, params }) //, from: account.value})
    console.log('Signature: ' + signature.value)

    const recovered = recoverTypedSignature({ data: JSON.parse(typedData), signature: signature.value as string, version: SignTypedDataVersion.V4})
    console.log('Recovered Address: ' + recovered)
    recoveredAcccount.value = recovered
    verified.value = recovered.toLowerCase() === account.value.toLowerCase()
  }
  catch (error) {
    console.error("Signing", error)
  }
}

const abi = [{"inputs":[{"internalType":"string","name":"name","type":"string"},{"internalType":"string","name":"symbol","type":"string"},{"internalType":"uint256","name":"initialSupply","type":"uint256"}],"stateMutability":"nonpayable","type":"constructor"},{"inputs":[{"internalType":"address","name":"spender","type":"address"},{"internalType":"uint256","name":"allowance","type":"uint256"},{"internalType":"uint256","name":"needed","type":"uint256"}],"name":"ERC20InsufficientAllowance","type":"error"},{"inputs":[{"internalType":"address","name":"sender","type":"address"},{"internalType":"uint256","name":"balance","type":"uint256"},{"internalType":"uint256","name":"needed","type":"uint256"}],"name":"ERC20InsufficientBalance","type":"error"},{"inputs":[{"internalType":"address","name":"approver","type":"address"}],"name":"ERC20InvalidApprover","type":"error"},{"inputs":[{"internalType":"address","name":"receiver","type":"address"}],"name":"ERC20InvalidReceiver","type":"error"},{"inputs":[{"internalType":"address","name":"sender","type":"address"}],"name":"ERC20InvalidSender","type":"error"},{"inputs":[{"internalType":"address","name":"spender","type":"address"}],"name":"ERC20InvalidSpender","type":"error"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"owner","type":"address"},{"indexed":true,"internalType":"address","name":"spender","type":"address"},{"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}],"name":"Approval","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"from","type":"address"},{"indexed":true,"internalType":"address","name":"to","type":"address"},{"indexed":false,"internalType":"uint256","name":"value","type":"uint256"}],"name":"Transfer","type":"event"},{"inputs":[{"internalType":"address","name":"owner","type":"address"},{"internalType":"address","name":"spender","type":"address"}],"name":"allowance","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"spender","type":"address"},{"internalType":"uint256","name":"value","type":"uint256"}],"name":"approve","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"account","type":"address"}],"name":"balanceOf","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"decimals","outputs":[{"internalType":"uint8","name":"","type":"uint8"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"name","outputs":[{"internalType":"string","name":"","type":"string"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"symbol","outputs":[{"internalType":"string","name":"","type":"string"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"totalSupply","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"value","type":"uint256"}],"name":"transfer","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"from","type":"address"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"value","type":"uint256"}],"name":"transferFrom","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"from","type":"address"},{"internalType":"address","name":"to","type":"address"},{"internalType":"uint256","name":"amount","type":"uint256"},{"internalType":"string","name":"onchainData","type":"string"}],"name":"transferWithData","outputs":[{"internalType":"bool","name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"}];
// const abi = [
//   "function decimals() view returns (uint8)",
//   "function symbol() view returns (string)",
// ];
// const abi = [
//   {
//         "inputs": [
//         ],
//         "name": "decimals",
//         "outputs": [
//             {
//                 "internalType": "uint8",
//                 "name": "",
//                 "type": "uint8"
//             }
//         ],
//         "stateMutability": "view",
//         "type": "function"
//     },
//     {
//         "inputs": [
//         ],
//         "name": "name",
//         "outputs": [
//             {
//                 "internalType": "string",
//                 "name": "",
//                 "type": "string"
//             }
//         ],
//         "stateMutability": "view",
//         "type": "function"
//     },
//     {
//         "inputs": [
//         ],
//         "name": "symbol",
//         "outputs": [
//             {
//                 "internalType": "string",
//                 "name": "",
//                 "type": "string"
//             }
//         ],
//         "stateMutability": "view",
//         "type": "function"
//     },
//     {
//         "inputs": [
//             {
//                 "internalType": "address",
//                 "name": "to",
//                 "type": "address"
//             },
//             {
//                 "internalType": "uint256",
//                 "name": "value",
//                 "type": "uint256"
//             }
//         ],
//         "name": "transfer",
//         "outputs": [
//             {
//                 "internalType": "bool",
//                 "name": "",
//                 "type": "bool"
//             }
//         ],
//         "stateMutability": "nonpayable",
//         "type": "function"
//     },
//     {
//         "inputs": [
//             {
//                 "internalType": "address",
//                 "name": "from",
//                 "type": "address"
//             },
//             {
//                 "internalType": "address",
//                 "name": "to",
//                 "type": "address"
//             },
//             {
//                 "internalType": "uint256",
//                 "name": "amount",
//                 "type": "uint256"
//             },
//             {
//                 "internalType": "string",
//                 "name": "onchainData",
//                 "type": "string"
//             }
//         ],
//         "name": "transferWithData",
//         "outputs": [
//             {
//                 "internalType": "bool",
//                 "name": "",
//                 "type": "bool"
//             }
//         ],
//         "stateMutability": "nonpayable",
//         "type": "function"
//     }];

async function transferWithData() {
  console.log(`hasWallet=${hasWallet.value} Account=${account.value}`)
  if (!ethereum || !hasWallet.value || !account.value) return
  try {
    const provider = new ethers.BrowserProvider(ethereum);
    //0x87a2Fc0E620eECd7D2c012F5C045baF5cC12446d
    const contract = new ethers.Contract("0xa450cE20a38eB3064783a3AdC43e379140817912", abi, provider);

    const signer = await provider.getSigner('0xa693d26e6ab62829d511dff2fb9b65656f052247'); // Assumes Metamask or similar is injected in the browser
    const contractWithSigner = contract.connect(signer) as any;

    const sym = await contract.symbol();
    // The number of decimals the token uses
    const decimals = await contract.decimals();
    console.log('sym:', sym, 'decimals:', decimals);

    const balance = await contract.balanceOf(account.value);
    console.log('balance:', balance, ethers.formatUnits(balance, decimals));

    // const value = ethers.parseEther('0.01');
    // const tx = await contractWithSigner.transferWithData(account.value, "0xB59BB5883aabab8243C9a3008390a656636D3497", value, 'Some data');
    // // const tx = await contractWithSigner.getFunction('transfer')("0xB59BB5883aabab8243C9a3008390a656636D3497", value);
    // // const tx = await contract.transferWithData(account.value, "0xB59BB5883aabab8243C9a3008390a656636D3497", 3, 'Some data');
    // console.log('Transaction sent. Wating for confirmation...')
    // const resp = await tx.wait();
    // console.log('Transaction confirmed', resp, tx);
  }
  catch (error) {
    console.error("transferWithData", error)
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
