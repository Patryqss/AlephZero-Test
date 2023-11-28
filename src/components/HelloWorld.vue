<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import { ApiPromise, WsProvider, Keyring } from '@polkadot/api'
import { base64Encode, mnemonicGenerate } from '@polkadot/util-crypto';
import { getSpecTypes } from '@polkadot/types-known';

import typesBundle from './typesBundle.mts';

// const wsUrl = 'wss://ws.azero.dev'; // MainNet
const wsUrl = 'wss://ws.test.azero.dev'; // TestNet

defineProps({
  msg: String,
})

const metaData = ref({});

const state = reactive({
  metaData: {},
  mnemonic: '',
  keyring: {},
  account: null,
  accData: null,
  allFetched: false,

  receiver: '',
  amount: '0',
  transferSuccess: false,
})

const accountBalance = computed(() => {
  if (!state.accData) return 0;
  const accBalanceRaw = state.accData.data.free.toString();
  return Number(accBalanceRaw) / (10 ** state.metaData.tokenDecimals);
})

onMounted(async () => {
  const api = await getApi();
  const chainName = await api.rpc.system.chain();

  state.metaData = {
    chain: chainName.toString(),
    chainType: 'substrate',
    color: '#00CCAB',
    genesisHash: api.genesisHash.toHex(),
    metaCalls: base64Encode(api.runtimeMetadata.asCallsOnly.toU8a()),
    specVersion: api.runtimeVersion.specVersion.toNumber(),
    ss58Format: api.registry.chainSS58,
    tokenDecimals: api.registry.chainDecimals[0],
    tokenSymbol: api.registry.chainTokens[0],
    types: getSpecTypes(api.registry, chainName, api.runtimeVersion.specName, api.runtimeVersion.specVersion),
    icon: 'substrate',
  };

  state.keyring = new Keyring({ type: 'sr25519' });
  state.keyring.setSS58Format(metaData.ss58Format);
});

async function getApi() {
  return await ApiPromise.create({ provider: new WsProvider(wsUrl), typesBundle });
}

function generateSeed() {
  state.mnemonic = mnemonicGenerate(12);
}

async function fetchAccount() {
  if (!state.mnemonic) return;
  const api = await getApi();
  state.account = state.keyring.createFromUri(state.mnemonic, { name: 'sr25519' });
  state.accData = await api.query.system.account(state.account.address);
  state.allFetched = true;
}

async function sendTxn() {
  const api = await getApi();
  const txnHash = await api.tx.balances
    .transfer(state.receiver, state.amount * (10 ** state.metaData.tokenDecimals))
    .signAndSend(state.account);

  console.log(`Submitted with hash ${txnHash}`);
  state.transferSuccess = true
}
</script>

<template>
  <h1>{{ msg }}</h1>

  <div class="container">
    <p>Your mnemonic:</p>
    <input type="text" v-model="state.mnemonic" placeholder="Enter your mnemonic or generate a new one">
    <button @click="generateSeed">Generate New</button>
  </div>

  <div class="container">
    <button @click="fetchAccount">Fetch Account</button>
    <div v-if="state.allFetched">
      <p>Your account: {{ state.account.address }}</p>
      <p>Your balance: {{ accountBalance }}</p>
    </div>
  </div>

  <div v-if="state.allFetched" class="container">
    <h2>Make a transaction</h2>
    <p>Receiver:</p>
    <input type="text" v-model="state.receiver" placeholder="Enter the receiver address">
    <p>Amount:</p>
    <input type="text" v-model="state.amount" placeholder="Enter the amount">
    <button @click="sendTxn">Send transaction</button>
  </div>

  <p v-if="state.transferSuccess" :style="{ color: 'green' }">Transfer Success!</p>
</template>

<style scoped>
input {
  width: 750px;
  max-width: 75%;
  display: block;
}

.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 20px;
}
</style>
