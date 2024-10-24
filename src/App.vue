<template>
  <div>
    <ThreeJs />
  </div>
</template>

<script>
// 引入ThreeJs组件
import ThreeJs from "./components/threejs.vue";
export default {
  name: "App",
  components: {
    ThreeJs, // 注册ThreeJs组件
  },
};
</script>

<style>
* {
  padding: 0;
  margin: 0;
}
</style>
