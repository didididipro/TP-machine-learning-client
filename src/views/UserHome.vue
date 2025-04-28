<script setup lang="ts">
import { ref } from "vue";
import { useToast } from "primevue/usetoast";
import InputText from "primevue/inputtext";
import Button from "primevue/button";
import Toast from "primevue/toast";
import DataTable from "primevue/datatable";
import Column from "primevue/column";
import Dialog from "primevue/dialog";
import { DateTime } from "luxon";

const visible = ref(false);
const predictions = ref([]);
const isGettingPredictions = ref(false);
const getPredictions = async () => {
  isGettingPredictions.value = true;
  const response = await fetch(
    `${import.meta.env.VITE_BASE_URL}/api/predictions/`,
    {
      method: "GET",
      headers: { "Content-Type": "application/json" },
    },
  );

  isGettingPredictions.value = false;
  if (!response.ok) throw new Error("Erreur réseau");

  const result = await response.json();
  const data = result.map((pred: any) => ({
    ...pred,
    confidence: `${pred.confidence * 100} %`,
    date: DateTime.fromISO(pred.date).toLocal().toFormat("dd MMM yyyy à HH:mm"),
  }));
  predictions.value = data;
};

getPredictions();

const toast = useToast();
const loading = ref(false);
const manualData = ref([{ nom_patient: "", UGP2: "" }]);
const predictionResults = ref([]);

const addGeneInput = () => {
  manualData.value.push({
    nom_patient: "",
    UGP2: "",
  });
};

const removeGeneInput = (index: number) => {
  if (manualData.value.length > 1) {
    manualData.value.splice(index, 1);
  } else {
    toast.add({
      severity: "warn",
      summary: "Avertissement",
      detail: "Au moins une entrée est requise",
      life: 3000,
    });
  }
};

const launchPrediction = async () => {
  const cleanedData = manualData.value.map(({ nom_patient, UGP2 }) => ({
    nom_patient: nom_patient.trim(),
    UGP2: parseFloat(UGP2),
  }));

  if (cleanedData.some((d) => !d.nom_patient || isNaN(d.UGP2))) {
    toast.add({
      severity: "error",
      summary: "Erreur",
      detail: "Toutes les entrées doivent être valides",
      life: 3000,
    });
    return;
  }

  try {
    loading.value = true;
    const response = await fetch(
      `${import.meta.env.VITE_BASE_URL}/api/predict/`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(cleanedData),
      },
    );

    loading.value = false;
    if (!response.ok) throw new Error("Erreur réseau");

    const result = await response.json();
    const data = result.map((pred: any) => ({
      ...pred,
      confidence: `${pred.confidence * 100} %`,
      date: DateTime.fromISO(pred.date)
        .toLocal()
        .toFormat("dd MMM yyyy à HH:mm"),
    }));

    predictionResults.value = data;

    toast.add({
      severity: "success",
      summary: "Succès",
      detail: "Prédictions reçues",
      life: 3000,
    });
    getPredictions();
  } catch (err) {
    loading.value = false;
    toast.add({
      severity: "error",
      summary: "Erreur",
      detail: "Échec de la prédiction",
      life: 3000,
    });
  }
};
</script>

<template>
  <div class="min-h-screen bg-gray-50 flex flex-col items-center p-6">
    <div class="w-full max-w-4xl bg-white rounded-lg shadow-lg p-8">
      <h2 class="text-3xl font-bold text-gray-800 mb-4 text-center">
        Prédiction du Cancer du Côlon
      </h2>
      <div class="flex justify-center p-2">
        <Button
          class="font-semibold py-2 px-4 rounded mb-8"
          :loading="isGettingPredictions"
          @click="visible = true"
          >Consulter l'Historique</Button
        >
      </div>
      <p class="text-gray-600 mb-6 text-center">
        Entrez manuellement les données pour prédire les risques de cancer du
        côlon.
      </p>

      <!-- Formulaire dynamique -->
      <div class="mb-8 space-y-4">
        <h3 class="text-xl font-semibold text-gray-700 mb-4">Entrées Gènes</h3>
        <div
          v-for="(entry, index) in manualData"
          :key="index"
          class="flex items-center gap-4"
        >
          <InputText
            v-model="entry.nom_patient"
            placeholder="Nom du patient"
            class="w-1/2"
          />
          <InputText v-model="entry.UGP2" placeholder="UGP2" class="w-1/2" />
          <Button
            icon="pi pi-trash"
            severity="danger"
            text
            @click="removeGeneInput(index)"
          />
        </div>
        <Button
          icon="pi pi-plus"
          label="Ajouter une entrée"
          @click="addGeneInput"
        />
      </div>

      <!-- Bouton prédiction -->
      <Button
        :loading="loading"
        label="Lancer la Prédiction"
        icon="pi pi-play"
        severity="primary"
        class="w-full font-semibold py-2 px-4 rounded mb-8"
        @click="launchPrediction"
      />

      <!-- Résultats -->
      <div v-if="predictionResults.length > 0">
        <h3 class="text-xl font-semibold text-gray-700 mb-4">
          Résultats de la Prédiction
        </h3>
        <DataTable
          showGridlines
          :value="predictionResults"
          tableStyle="min-width: 20rem"
          class="p-datatable-sm"
          responsiveLayout="scroll"
        >
          <Column field="nom_patient" header="Nom du patient" />
          <Column header="Prédiction">
            <template #body="slotProps">
              <span
                :class="
                  slotProps.data.pred === 'Tumoral'
                    ? 'text-red-500 font-bold'
                    : 'text-teal-500 font-bold'
                "
              >
                {{ slotProps.data.pred }}
              </span>
            </template>
          </Column>
          <Column field="confidence" header="Confiance" />
        </DataTable>
      </div>

      <Toast position="top-center" />
    </div>
  </div>

  <Dialog
    v-model:visible="visible"
    modal
    header="Historique des prédictions"
    :style="{ width: '50rem' }"
    :breakpoints="{ '1199px': '75vw', '575px': '90vw' }"
  >
    <DataTable
      showGridlines
      :value="predictions"
      tableStyle="min-width: 20rem"
      class="p-datatable-sm"
      responsiveLayout="scroll"
    >
      <Column field="nom_patient" header="Nom du patient" />
      <Column field="value" header="Gène UGP2" />
      <Column header="Prédiction">
        <template #body="slotProps">
          <span
            :class="
              slotProps.data.pred === 'Tumoral'
                ? 'text-red-500 font-bold'
                : 'text-teal-500 font-bold'
            "
          >
            {{ slotProps.data.pred }}
          </span>
        </template>
      </Column>
      <Column field="confidence" header="Confiance" />
      <Column field="date" header="Date" />
    </DataTable>
  </Dialog>
</template>
