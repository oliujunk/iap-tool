<template>
  <div class="home">

  </div>
</template>

<script>
import { SerialPort } from 'serialport';

export default {
  name: 'HomeView',

  mounted() {
    SerialPort.list().then((ports) => {
      console.log(ports);
    });
  },
};
</script>

<style lang="scss" scoped>
.home {
  width: 100%;
}

</style>
