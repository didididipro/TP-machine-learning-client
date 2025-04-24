<script setup lang="ts">
import { ref } from 'vue'
import { useToast } from 'primevue/usetoast'
import InputText from 'primevue/inputtext'
import Button from 'primevue/button'
import Toast from 'primevue/toast'
import DataTable from 'primevue/datatable'
import Column from 'primevue/column'

const toast = useToast()
const loading = ref(false)
const manualData = ref([{ id_patient: 'patient_1', DAO: '' }])
const predictionResults = ref([])
const sampleCounter = ref(2)

const addGeneInput = () => {
  manualData.value.push({
    id_patient: `patient_${sampleCounter.value++}`,
    DAO: ''
  })
}

const removeGeneInput = (index: number) => {
  if (manualData.value.length > 1) {
    manualData.value.splice(index, 1)
  } else {
    toast.add({
      severity: 'warn',
      summary: 'Avertissement',
      detail: 'Au moins une entrée est requise',
      life: 3000,
    })
  }
}

const launchPrediction = async () => {
  const cleanedData = manualData.value.map(({ id_patient, DAO }) => ({
    id_patient: id_patient.trim(),
    DAO: parseFloat(DAO),
  }))

  if (cleanedData.some(d => !d.id_patient || isNaN(d.DAO))) {
    toast.add({
      severity: 'error',
      summary: 'Erreur',
      detail: 'Toutes les entrées doivent être valides',
      life: 3000,
    })
    return
  }

  try {
    loading.value = true
    const response = await fetch('https://tp-machine-learning-server.onrender.com/api/predict/', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(cleanedData),
    })

    loading.value = false
    if (!response.ok) throw new Error('Erreur réseau')

    const result = await response.json()
    // Adapter les champs si le backend utilise "id_sample"
    predictionResults.value = result.map((r: any, i: number) => ({
      id_patient: manualData.value[i]?.id_patient || `patient_${i+1}`,

      prediction: r.prediction,
      confidence: r.confidence,
    }))

    toast.add({
      severity: 'success',
      summary: 'Succès',
      detail: 'Prédictions reçues',
      life: 3000,
    })
  } catch (err) {
    loading.value = false
    toast.add({
      severity: 'error',
      summary: 'Erreur',
      detail: 'Échec de la prédiction',
      life: 3000,
    })
  }
}
</script>


<template>
  <div class="min-h-screen bg-gray-50 flex flex-col items-center p-6">
    <div class="w-full max-w-4xl bg-white rounded-lg shadow-lg p-8">
      <h2 class="text-3xl font-bold text-gray-800 mb-4 text-center">
        Prédiction du Cancer du Côlon
      </h2>
      <p class="text-gray-600 mb-6 text-center">
        Entrez manuellement les données pour prédire les risques de cancer du côlon.
      </p>

      <!-- Formulaire dynamique -->
      <div class="mb-8 space-y-4">
        <h3 class="text-xl font-semibold text-gray-700 mb-4">Entrées Gènes</h3>
        <div
          v-for="(entry, index) in manualData"
          :key="index"
          class="flex items-center gap-4"
        >
          <InputText v-model="entry.id_patient" class="w-1/2" readonly />
          <InputText v-model="entry.DAO" placeholder="DAO" class="w-1/2" />
          <Button icon="pi pi-trash" severity="danger" text @click="removeGeneInput(index)" />
        </div>
        <Button icon="pi pi-plus" label="Ajouter une entrée" @click="addGeneInput" />
      </div>

      <!-- Bouton prédiction -->
      <Button
        :loading="loading"
        label="Lancer la Prédiction"
        icon="pi pi-play"
        severity="primary"
        class="w-full bg-blue-600 hover:bg-blue-700 text-white font-semibold py-2 px-4 rounded mb-8"
        @click="launchPrediction"
      />

      <!-- Résultats -->
      <div v-if="predictionResults.length > 0">
        <h3 class="text-xl font-semibold text-gray-700 mb-4">Résultats de la Prédiction</h3>
        <DataTable
          showGridlines
          :value="predictionResults"
          tableStyle="min-width: 20rem"
          class="p-datatable-sm"
          responsiveLayout="scroll"
        >
          <Column field="id_patient" header="ID" />
          <Column field="prediction" header="Prédiction" />
          <Column field="confidence" header="Confiance" />
        </DataTable>
      </div>

      <Toast position="top-center" />
    </div>
  </div>
</template>
