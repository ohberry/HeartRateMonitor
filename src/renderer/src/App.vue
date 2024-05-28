<script setup>
import { ref } from "vue";

const rate = ref(null);
const btDevice = ref(null);
const flag = ref(false);

async function test() {
  try {
    btDevice.value = await navigator.bluetooth.requestDevice({
      filters: [{ services: ["heart_rate"], name: "HUAWEI Band HR-C8A" }]
    });
    btDevice.value.addEventListener("gattserverdisconnected", onDisconnected);
    const server = await btDevice.value.gatt.connect();
    const service = await server.getPrimaryService("heart_rate");
    const characteristic = await service.getCharacteristic("heart_rate_measurement");
    await characteristic.startNotifications();
    characteristic.addEventListener("characteristicvaluechanged", handleCharacteristicValueChanged);
    flag.value = true;
    window.api.ignoreMouse(true);
  } catch (e) {
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
  console.log("> Bluetooth Device disconnected");
}
</script>

<template>
  <div class="rate">{{ rate }}</div>
  <div class="btn" v-if="!flag">
    <button @click="test">请求</button>
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
