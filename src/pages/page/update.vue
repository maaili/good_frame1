<template>
  <div class="app-container">
    <save-device-page-form :is-add="false" />
  </div>
</template>

<script>
import SaveDevicePageForm from './components/SaveDevicePageForm'
export default {
  components: {
    SaveDevicePageForm
  }
}
</script>
