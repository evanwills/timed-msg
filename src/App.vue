<script setup>
import TimedMsg from './components/TimedMsg.vue';

const times = [];

const now = Date.now();
const warnAt = 30;
const noticeAt = 60;

let a = -250;
let b = 1;
while (a < 400) {
  const tmp = new Date(now + (a * 1000));
  times.push({ time: tmp.toISOString(), id: `item-${b}`});
  a += 45;
  b += 1;
}
</script>

<template>
  <p v-for="(time) of times">
    {{ time.id }}&nbsp;
    <TimedMsg :when="time.time" :id="time.id" :warn-at="warnAt" :notice-at="noticeAt" />
    {{ time.time.replace(/T([0-9]{2}(?::[0-9]{2})+)\.[0-9]+Z/, ' $1') }}
  </p>
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
