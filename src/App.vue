<template>
  <v-layout class="overflow-visible">

    <!-- #region Первая вкладка (Замер показаний) -->
    <v-col v-if="navTab == 0" class="mt-10 mb-16 w-100">

      <v-form class="w-100" @submit.prevent="add">

        <v-text-field v-model="datetime" :rules="rules" type="datetime-local" label="Время"
          append-inner-icon="mdi-timer-sync-outline" variant="solo" class="mb-2"
          @click:append-inner="syncDateTime"></v-text-field>

        <v-text-field v-model="sugar" :rules="rules" type="number" label="Сахар" class="mb-2" variant="solo"
          @click="dialogRemove = true"></v-text-field>

        <v-text-field v-model="bread_units" :rules="rules" type="number" label="Хлебные еденицы" class="mb-2"
          variant="solo"></v-text-field>

        <v-text-field v-model="insulin_k" :rules="rules" type="number" label="Инсулин (К)" class="mb-2"
          variant="solo"></v-text-field>

        <v-text-field v-model="insulin_p" :rules="rules" type="number" label="Инсулин (П)" class="mb-2"
          variant="solo"></v-text-field>

        <v-textarea v-model="note" label="Заметка" variant="solo"></v-textarea>

        <v-btn variant="flat" type="submit" block color="success">Сохранить</v-btn>

      </v-form>

    </v-col>
    <!-- #endregion -->

    <!-- #region Вторя вкадка (История) -->
    <v-col class="mt-10 mb-16 w-100" v-if="navTab == 1">

      <!-- Диалоговое окно -->
      <v-dialog v-model="removeDialog" persistent>

        <v-card title="Удаление замера показаний">

          <template v-slot:actions>
            <v-spacer></v-spacer>

            <v-btn color="primary" @click="cancel">Отмена</v-btn>
            <v-btn color="error" @click="remove">Удалить</v-btn>

          </template>
        </v-card>
      </v-dialog>

      <!-- Средние показатели за 90 дней -->
      <v-card v-if="checks.length" class="mb-4">

        <template v-slot:title>Средние показатели за 90 дней</template>

        <v-card-text>

          <v-row class="px-3">

            <v-col class="px-0">
              <span class="font-weight-bold">Сахар:</span><br><span>{{ avgSugar.toFixed(2) }}</span>
            </v-col>

            <v-col class="px-0">
              <span class="font-weight-bold">Х.Е.:</span><br><span>{{ avgBread_units.toFixed(2) }}</span>
            </v-col>

            <v-col class="px-0">
              <span class="font-weight-bold">Инсулин К.:</span><br><span>{{ avgInsulin_k.toFixed(2) }}</span>
            </v-col>

            <v-col class="px-0">
              <span class="font-weight-bold">Инсулин П.:</span><br><span>{{ avgInsulin_p.toFixed(2) }}</span>
            </v-col>
          </v-row>

        </v-card-text>
      </v-card>

      <!-- Замеры -->
      <v-card v-for="(item, index) in checks" class="mb-4">

        <template v-slot:title>

          <div class="d-flex justify-space-between">

            <!-- Заголовок -->
            <span>{{ formatDate(item.datetime) }}</span>

            <!-- Кнопка удалить -->
            <v-btn color="error" @click="showRemoveDialog(index)">
              <v-icon>mdi-trash-can-outline</v-icon>
            </v-btn>

          </div>

        </template>

        <!-- Показатели -->
        <v-card-text>

          <v-row class="px-3">

            <v-col class="px-0">
              <span class="font-weight-bold">Сахар:</span><br><span>{{ item.sugar }}</span>
            </v-col>

            <v-col class="px-0">
              <span class="font-weight-bold">Х.Е.:</span><br><span>{{ item.bread_units }}</span>
            </v-col>

            <v-col class="px-0">
              <span class="font-weight-bold">Инсулин К.:</span><br><span>{{ item.insulin_k }}</span>
            </v-col>

            <v-col class="px-0">
              <span class="font-weight-bold">Инсулин П.:</span><br><span>{{ item.insulin_p }}</span>
            </v-col>
          </v-row>

          <!-- Заметка -->
          {{ item.note }}

        </v-card-text>
      </v-card>

    </v-col>
    <!-- #endregion -->

    <!-- #region Панель навигации -->
    <v-bottom-navigation v-model="navTab" color="teal" grow>

      <v-btn>
        <v-icon>mdi-heart</v-icon>
        Замер показаний
      </v-btn>

      <v-btn>
        <v-icon>mdi-history</v-icon>
        История
      </v-btn>

    </v-bottom-navigation>
    <!-- #endregion -->

  </v-layout>
</template>

<script>
export default {
  data: () => ({
    navTab: 0,
    removeDialog: false,
    removeIndex: null,
    datetime: '',
    sugar: '',
    bread_units: '',
    insulin_k: '',
    insulin_p: '',
    avgSugar: 0,
    avgBread_units: 0,
    avgInsulin_k: 0,
    avgInsulin_p: 0,
    note: '',
    rules: [
      value => {
        if (value) return true
        return 'Заполните значение'
      },
    ],
    checks: []
  }),
  methods: {
    async add(event) {

      const results = await event
      if (results.valid) {

        this.checks.push({
          datetime: Date.parse(this.datetime),
          sugar: parseFloat(this.sugar),
          bread_units: parseFloat(this.bread_units),
          insulin_k: parseFloat(this.insulin_k),
          insulin_p: parseFloat(this.insulin_p),
          note: this.note,
        })

        this.saveChecks()

        event.target.reset()
      }

    },
    remove() {
      this.checks.splice(this.removeIndex, 1);

      this.removeIndex = null
      this.removeDialog = false

      this.saveChecks()
    },
    cancel() {
      this.removeIndex = null
      this.removeDialog = false
    },
    showRemoveDialog(index) {
      this.removeIndex = index
      this.removeDialog = true
    },
    calcAvgChecks() {
      let countChecks = 0

      this.avgSugar = 0
      this.avgBread_units = 0
      this.avgInsulin_k = 0
      this.avgInsulin_p = 0

      this.checks.forEach((check, index) => {

        this.avgSugar += parseFloat(check.sugar)
        this.avgBread_units += parseFloat(check.bread_units)
        this.avgInsulin_k += parseFloat(check.insulin_k)
        this.avgInsulin_p += parseFloat(check.insulin_p)

        countChecks++
      });

      this.avgSugar = this.avgSugar / countChecks
      this.avgBread_units = this.avgBread_units / countChecks
      this.avgInsulin_k = this.avgInsulin_k / countChecks
      this.avgInsulin_p = this.avgInsulin_p / countChecks
    },
    saveChecks() {
      this.checks.sort((a, b) => a.datetime - b.datetime);
      localStorage.setItem('checks', JSON.stringify(this.checks))

      this.calcAvgChecks()
    },
    syncDateTime() {
      this.datetime = (new Date()).toLocaleString("sv-SE", {
        year: "numeric",
        month: "2-digit",
        day: "2-digit",
        hour: "2-digit",
        minute: "2-digit"
      }).replace(" ", "T");
    },
    formatDate(dateStr) {
      let date = new Date(dateStr)
      return (
        [
          this.padTo2Digits(date.getDate()),
          this.padTo2Digits(date.getMonth() + 1),
          date.getFullYear(),
        ].join('.') + ' ' +
        [
          this.padTo2Digits(date.getHours()),
          this.padTo2Digits(date.getMinutes())
        ].join(':')
      );
    },
    padTo2Digits(num) {
      return num.toString().padStart(2, '0');
    }
  },
  mounted() {
    this.checks = JSON.parse(localStorage.getItem('checks')) || []

    this.checks.sort((a, b) => a.datetime - b.datetime);

    this.calcAvgChecks()
  }
}
</script>
