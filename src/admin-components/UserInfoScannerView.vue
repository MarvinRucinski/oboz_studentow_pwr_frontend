<script setup>
import TopBar from '../components/navigation/TopBar.vue'
import ScannerBaseView from './ScannerBaseView.vue'
import { IonPage, IonContent } from '@ionic/vue';
</script>

<template>
  <ion-page>
    <ion-content :fullscreen="false">
      <main>
        <TopBar title="Informacje o uczestniku" backLink="/skaner" />

        <div class="padding-main">
          <div class="center">
            <p v-if="error" class="error">{{ error }}</p>
            <ScannerBaseView @error="(err) => (error = err)" @result="(res) => {
              result = res
              showUserInfo()
            }
              " />
          </div>
        </div>
      </main>
    </ion-content>
  </ion-page>
</template>

<script>
export default {
  data() {
    return {
      result: '',

      resultLoading: false,

      user: '',
      error: '',
      success: null
    }
  },
  methods: {
    showUserInfo() {
      this.$router.push({ name: 'uczestnik', params: { id: this.result } })
    }
  }
}
</script>

<style scoped>
.center {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}
</style>
