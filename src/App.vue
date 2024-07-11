<script setup lang="ts">
import HelloWorld from './components/HelloWorld.vue'
import maxdemo from './components/maxdemo.vue'
import { ref } from 'vue';
interface NodeData {
  key: string;
  color: string;
}

interface LinkData {
  from: string;
  to: string;
}

interface Rect {
  x: number;
  y: number;
  width: number;
  height: number;
}

const nodes = ref<Array<NodeData>>([
      { key: "M1", color: "lightblue" },
      { key: "S2", color: "orange" },
      // { key: "Switch", n: "Paternal Great", s: "M", ux: -33, a: ["F", "H", "S"] },
    ]);

    const links = ref<Array<LinkData>>([
      { from: "M1", to: "S2" }
    ]);

  const restrictedArea = ref<Rect>({ x: 0, y: 0, width: 300, height: 200 });
</script>

<template>
  <div style="width: 99%; height: 100%;">
    <div>xxxx</div>
    <maxdemo/>
  </div>
 
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
