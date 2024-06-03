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
    window.api.ignoreMouse();
  } catch (e) {
  }
}

/**
 * 异步函数：连接到蓝牙设备的心率服务并启用通知
 * 
 * 本函数旨在通过蓝牙低功耗（BLE）协议连接到一个心率服务设备，并启动心率测量特性通知。
 * 这使得应用程序可以实时接收心率测量值的更新，而无需轮询设备。
 * 
 * @remarks
 * 此函数假设 btDevice 已经通过某种方式被初始化，并且它的 value 属性包含一个有效的 BluetoothDevice 对象。
 * 
 * @returns {void}
 */
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

/**
 * 异步函数 disconnect 用于断开与 Bluetooth 设备的连接。
 * 
 * 此函数首先检查是否存在已选中的 Bluetooth 设备。如果不存在，则直接返回，
 * 不执行任何断开连接的操作。如果存在已选中的设备且该设备当前与客户端建立了 GATT 连接，
 * 则函数将尝试异步断开此连接。
 * 
 * 注意：此函数不处理断开连接的失败情况，调用方应适当处理可能的异常。
 */
async function disconnect() {
  // 检查是否存在已选中的 Bluetooth 设备
  if (!btDevice.value) return;
  
  // 如果设备已连接，则尝试断开 GATT 连接
  if (btDevice.value.gatt.connected) {
    await btDevice.value.gatt.disconnect();
  }
}

/**
 * 处理特征值变化事件
 * @param {*} event 
 */
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
