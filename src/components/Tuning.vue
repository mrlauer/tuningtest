<template>
  <div class="tuning-component">
    <v-container>
      <v-row>
        <v-col cols="12">
            <v-card>
            <v-toolbar
                class="note-toolbar"
                width="auto"
                density="compact">
                <v-btn-toggle 
                    v-model="rawSelection"
                    multiple
                    divided
                    density="compact"
                    style="padding: 0 6px;">
                  <v-btn 
                    v-for="index in 25" 
                    :key="index-1" 
                    :value="index-1"
                    density="compact"
                    color="primary"
                    border
                    style="min-width:30px; height:28px; padding:0 6px; font-size:12px;"
                  >
                    {{ notenames[(index - 1) % 12] }}
                  </v-btn>
                </v-btn-toggle>
            </v-toolbar>
            </v-card>
            <v-btn>
                <template #default>
                    <span @click="rawSelection = []" style="cursor: pointer;">Clear</span>
                </template>
            </v-btn>
          <div class="selected-notes">
            <h3>Selected Notes:</h3>
            {{  sortedNotes.map(index => notenames[index % 12]).join(', ') }}
          </div>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script lang="ts" setup>
import { ref, computed, watch } from 'vue';

const notenames = ref(['C', 'D♭', 'D', 'E♭', 'E', 'F', 'F♯', 'G', 'A♭', 'A', 'B♭', 'B']);
const rawSelection = ref([] as number[]);

watch(rawSelection, (newVal: number[], oldValue: number[]) => {
});

const sortedNotes = computed(() => {
    const selectedCopy = rawSelection.value.slice();
    selectedCopy.sort((a, b) => a - b);
    return selectedCopy;
});


</script>


<style scoped>
.note-toolbar {
  height: 40px;
  min-height: 40px;
}
</style>