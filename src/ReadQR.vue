<template>
  <div>
    <h1>QR Code Scanner</h1>
    <div id="interactive" class="viewport"></div>
    <p>Results:</p>
    <ul>
      <li v-for="(order, index) in scannedOrders" :key="index">
        <strong>Name:</strong> {{ order.name }}, <strong>Price:</strong> {{ order.price }}, <strong>Total:</strong> {{ order.total }}
      </li>
    </ul>
  </div>
</template>

<script>
import Quagga from 'quagga';

export default {
  data() {
    return {
      scannedOrders: []
    };
  },
  mounted() {
    Quagga.init({
      inputStream: {
        name: "Live",
        type: "LiveStream",
        target: document.querySelector('#interactive'),
        constraints: {
          width: 480,
          height: 320,
          facingMode: "environment"
        },
      },
      decoder: {
        readers: ["ean_reader"]
      }
    }, (err) => {
      if (err) {
        console.error(err);
        return;
      }
      console.log("Initialization finished. Ready to start");
      Quagga.start();
    });

    Quagga.onDetected((data) => {
      const orderData = data.codeResult.code.split('|');
      this.scannedOrders.push({
        name: orderData[0],
        price: orderData[1],
        total: orderData[2]
      });
    });
  },
  beforeDestroy() {
    Quagga.stop();
  }
};
</script>

<style>
.viewport {
  width: 100%;
  height: 320px;
  position: relative;
}

.viewport > canvas, .viewport > video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
}
</style>
