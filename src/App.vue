<template>
  <div class="container">
    <h1>Ovqat Hisob-kitobi </h1>
    <form @submit.prevent="calculateOrder">
      <div class="form-group-row" v-for="(order, index) in newOrders" :key="index">
        <div class="form-group">
          <label>Ism</label>
          <input v-model="order.name" placeholder="Ism" required/>
        </div>

        <div class="form-group">
          <label>Ovqat nomi</label>
          <input v-model="order.foodName" placeholder="Ovqat nomi" required/>
        </div>

        <div class="form-group">
          <label>Ovqat narxi</label>
          <input v-model.number="order.price" type="number" placeholder="Ovqat narxi" min="0.01" step="0.01" required/>
        </div>

        <button type="button" class="btn-add" @click="addNewOrderField">+</button>
        <button type="button" class="btn-remove" @click="removeOrderField(index)" v-if="newOrders.length > 1">-</button>
      </div>
      <div class="form-group">
        <label>Dostavka</label>
        <input v-model.number="deliveryPrice" type="number" placeholder="Dostavka" :disabled="isDineIn"
               @input="handleDeliveryInput" min="0"/>
      </div>
      <div class="form-group">
        <label>
          <input type="checkbox" v-model="isDineIn" :disabled="deliveryPrice > 0"/> Restoranda ovqatlanish
        </label>
      </div>
      <div class="form-group" v-if="isDineIn">
        <label>Ofitsant hizmati (%)</label>
        <input v-model.number="serviceFeeTotal" type="number" placeholder="Ofitsant hizmati" min="0" step="0.01"/>
      </div>
      <div class="button-group">
        <button type="submit">Hisoblash</button>
        <button type="button" class="btn-telegram" @click="sendToTelegram">Telegramdan yuborish</button>
      </div>
    </form>

    <h2>Buyurtmalar ro'yxati</h2>
    <table>
      <thead>
      <tr>
        <th>👤 Ism</th>
        <th>🍲 Ovqat nomi</th>
        <th>💵 Ovqat narxi</th>
        <th>💵 Umumiy narx</th>
      </tr>
      </thead>
      <tbody>
      <tr v-for="(order, index) in orders" :key="index">
        <td><strong>{{ order.name }}</strong></td>
        <td>{{ order.foodName }}</td>
        <td>{{ order.price }}</td>
        <td><strong>{{ order.total }}</strong></td>
      </tr>
      <tr>
        <td colspan="3" style="text-align: right;"><strong>Umumiy:</strong></td>
        <td><strong>{{ totalSum }}</strong></td>
      </tr>
      <tr v-if="deliveryPrice > 0">
        <td colspan="3" style="text-align: right;"><strong>Dostavka:</strong></td>
        <td><strong>{{ deliveryPrice }}</strong></td>
      </tr>
      <tr v-if="isDineIn && deliveryPrice === 0">
        <td colspan="3" style="text-align: right;"><strong>Ofitsant xizmati:</strong></td>
        <td><strong>{{ serviceFeeTotal }}%</strong></td>
      </tr>
      </tbody>
    </table>

    <div class="qrcode" v-if="orders.length">
      <qrcode-vue :value="qrcodeText" :size="200" level="L"></qrcode-vue>
    </div>
  </div>
</template>

<script>
import {ref, computed} from 'vue';
import QrcodeVue from 'qrcode.vue';
import axios from 'axios';

export default {
  name: 'App',
  components: {
    'qrcode-vue': QrcodeVue
  },
  setup() {
    const orders = ref([]);
    const newOrders = ref([{name: '', foodName: '', price: 0}]);
    const deliveryPrice = ref(0);
    const serviceFeeTotal = ref(0);
    const isDineIn = ref(false);

    const calculateOrder = () => {
      const numOrders = newOrders.value.length;
      const deliveryPerPerson = deliveryPrice.value / numOrders;
      const serviceFeePercentage = serviceFeeTotal.value / 100;
      const ordersWithTotal = [];

      newOrders.value.forEach(order => {
        const totalBeforeServiceFee = (order.price || 0) + (deliveryPerPerson || 0);
        const serviceFeePerOrder = totalBeforeServiceFee * serviceFeePercentage;
        const total = totalBeforeServiceFee + serviceFeePerOrder;
        ordersWithTotal.push({...order, total});
      });

      orders.value = ordersWithTotal;
    };

    const sendToTelegram = async () => {
      const token = '6533471879:AAFsHM0c39uwPa4v-D0oJ2v6km8N7fdd-QI'; // token
      const chat_id = '-610176840'; //  guruh chat ID
      let message = 'Ovqat hisob-kitobi:\n\n';
      orders.value.forEach(order => {
        message += `*👤 Ism*: **${order.name}**\n🍲 Ovqat nomi: ${order.foodName}\n💵 Ovqat narxi: ${order.price}\n*💵 Umumiy narx*: **${order.total}**\n\n`;
      });
      message += `*💵 Umumiy*: **${totalSum.value}**\n`;
      if (deliveryPrice.value > 0) {
        message += `*📦 Dostavka*: **${deliveryPrice.value}**\n`;
      }
      if (isDineIn.value && deliveryPrice.value === 0) {
        message += `*🧾 Ofitsant xizmati*: **${serviceFeeTotal.value}%**`;
      }

      try {
        const response = await axios.post(`https://api.telegram.org/bot${token}/sendMessage`, {
          chat_id: chat_id,
          text: message,
          parse_mode: 'Markdown'
        });
      } catch (error) {
      }
    };

    const addNewOrderField = () => {
      newOrders.value.push({name: '', foodName: '', price: 0});
    };

    const removeOrderField = (index) => {
      newOrders.value.splice(index, 1);
    };

    const handleDeliveryInput = () => {
      if (deliveryPrice.value > 0) {
        isDineIn.value = false;
      }
    };

    const totalCostPerPerson = computed(() => {
      if (orders.value.length === 0) return 0;
      const totalCost = orders.value.reduce((sum, order) => sum + (order.total || 0), 0);
      return (totalCost / orders.value.length).toFixed(2);
    });

    const qrcodeText = computed(() => {
      let result = '';
      for (let i = 0; i < orders.value.length; i++) {
        const order = orders.value[i];
        result += `Ism: ${order.name}, Ovqat nomi: ${order.foodName || 'Noma\'lum'}, Ovqat narxi: ${order.price || 0}, Umumiy narx: ${order.total || 0}\n`;
      }
      return result.trim();
    });

    const totalSum = computed(() => {
      if (orders.value.length === 0) return 0;
      return orders.value.reduce((sum, order) => sum + (order.total || 0), 0);
    });

    return {
      orders,
      newOrders,
      deliveryPrice,
      serviceFeeTotal,
      isDineIn,
      calculateOrder,
      sendToTelegram,
      addNewOrderField,
      removeOrderField,
      handleDeliveryInput,
      totalCostPerPerson,
      qrcodeText,
      totalSum
    };
  },
};
</script>


<style>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  background: #0e1e28;
  border-radius: 25px;
}

form {
  display: grid;
  gap: 20px;
  margin-bottom: 20px;
}

form .form-group {
  display: flex;
  flex-direction: column;
  margin-right: 10px;
}

form .form-group-row {
  display: flex;
  align-items: flex-start;
}

form label {
  font-weight: bold;
  margin-bottom: 5px;
}

form input {
  padding: 10px;
  font-size: 16px;
}

form button {
  padding: 10px;
  font-size: 16px;
  color: white;
  border: none;
  cursor: pointer;
  margin-top: 30px;
}

form button[type="button"] {
  margin-left: 10px;
}

form .btn-add {
  background-color: #2196F3;
}

form .btn-add:hover {
  background-color: #1976D2;
}

form .btn-remove {
  background-color: #f44336;
}

form .btn-remove:hover {
  background-color: #d32f2f;
}

form button[type="submit"] {
  background-color: #4CAF50;
}

form button[type="submit"]:hover {
  background-color: #45a049;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
}

table th, table td {
  border: 1px solid #ddd;
  padding: 8px;
  color: #d8d7d7;
}

table th {
  background-color: #f2f2f2;
  text-align: left;
  color: #181818;
  font-weight: bold;
}

h2 {
  margin-top: 20px;
  color: #ffffff;
  text-align: center;
}

h1 {
  margin-bottom: 20px;
  color: #ffffff;

}

.qrcode {
  text-align: center;
  margin-top: 20px;
}

.button-group {
  display: flex;
  gap: 10px;
}

.btn-telegram {
  background-color: #28a745;
  color: white;
  border: none;
  padding: 10px 20px;
  cursor: pointer;
}

.btn-telegram:hover {
  background-color: #218838;
}
</style>