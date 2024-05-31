<script setup>
import { ref } from "vue";

const rate = ref(null);
const btDevice = ref(null);
const flag = ref(false);
let reconnectInterval;

async function request() {
  try {
    btDevice.value = await navigator.bluetooth.requestDevice({
      filters: [{ services: ["heart_rate"], name: "HUAWEI Band HR-C8A" }]
    });
    btDevice.value.addEventListener("gattserverdisconnected", onDisconnected);
    await connect();
    flag.value = true;
    window.api.ignoreMouse(true);
  } catch (e) {
  }
}

async function connect() {
  const server = await btDevice.value.gatt.connect();
  const service = await server.getPrimaryService("heart_rate");
  const characteristic = await service.getCharacteristic("heart_rate_measurement");
  await characteristic.startNotifications();
  characteristic.addEventListener("characteristicvaluechanged", handleCharacteristicValueChanged);
  if (reconnectInterval) {
    clearInterval(reconnectInterval);
    reconnectInterval = null;
  }
}

async function disconnect() {
  if (!btDevice.value) return;
  if (btDevice.value.gatt.connected) {
    await btDevice.value.gatt.disconnect();
  }
}

const handleCharacteristicValueChanged = (event) => {
  rate.value = event.target.value.getUint8(1);
};

function onDisconnected() {
  rate.value = 0;
  reconnectInterval = setInterval(async () => {
    if (!btDevice.value.gatt.connected) {
      try {
        await connect();
        clearInterval(reconnectInterval);
      } catch (e) {
      }
    }
  }, 2000);
}
</script>

<template>
  <div class="rate">{{ rate }}</div>
  <div class="btn" v-if="!flag">
    <button @click="request">请求</button>
    <button @click="disconnect">断开</button>
  </div>
</template>

<style>
.rate {
  color: rgb(13, 245, 35);
  font-size: 20px;
}

.btn {
  display: flex;
}
</style>
