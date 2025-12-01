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
            <v-btn @click="rawSelection = []" class="clear-button" density="compact">
              Clear
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

// Frequencies. TODO: encapsulate somewhere else.
const middle_c_freq = 261.6255653005986; // Middle C (C4) frequency in Hz
const even_tempered_ratio = Math.pow(2, 1 / 12); // Twelfth root of 2
const even_tempered_frequncies = Array.from({ length: 12 }, (_, i) => 
    middle_c_freq * Math.pow(even_tempered_ratio, i)
);

const frequencies = computed(() => {
  even_tempered_frequncies
});

const audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
const gain = audioCtx.createGain();
gain.gain.setValueAtTime(0.1, audioCtx.currentTime); // Set volume
gain.connect(audioCtx.destination);

let oscillators = [] as OscillatorNode[];
function create_oscillators() {
  oscillators = sortedNotes.value.map(index => {
    const osc = audioCtx.createOscillator();
    osc.type = 'sine';
    const baseFreq = even_tempered_frequncies[index % 12] ?? middle_c_freq;
    const octaveMultiplier = Math.pow(2, Math.floor(index / 12));
    const freq = baseFreq * octaveMultiplier;
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
    osc.connect(gain);
    osc.start();
    return osc;
  });
}

function clear_oscillators() {
  oscillators.forEach(osc => {
    osc.stop();
    osc.disconnect();
  });
  oscillators = [];
}

const notenames = ref(['C', 'D♭', 'D', 'E♭', 'E', 'F', 'F♯', 'G', 'A♭', 'A', 'B♭', 'B']);
const rawSelection = ref([] as number[]);

watch(rawSelection, (newVal: number[], oldValue: number[]) => {
    clear_oscillators();
    if (newVal.length > 0) {
        create_oscillators();
    }
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
.clear-button {
  margin: 10px 0 5px 0;
}
</style>