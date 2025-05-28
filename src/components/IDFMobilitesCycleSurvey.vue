<template>
  <SurveyTemplate
    :survey-questions="idfMobilitesCycleSurveyQuestions"
    firebase-collection-name="idfMobilitesCycleSurveyData"
    :stations-list="[]" 
    :streets-list="[]"
  />
</template>

<script setup>
import SurveyTemplate from './SurveyTemplate.vue';
import { idfMobilitesCycleSurveyQuestions } from './idfMobilitesCycleSurveyQuestions.js';

// This survey does not seem to require dynamic station or street lists based on the provided questions.
// If they were needed for a 'station' or 'street' type question, they would be fetched or defined here.
// For example:
// import { ref, onMounted } from 'vue';
// const localStationsList = ref([]);
// onMounted(async () => {
//   const response = await fetch('/path/to/specific_stations.json');
//   localStationsList.value = await response.json();
// });
</script>

<style scoped>
/* Add any survey-specific styles here if needed */
</style> 