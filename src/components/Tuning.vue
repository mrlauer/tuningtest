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
            <v-btn 
              @click="muted = !muted;" 
              :active="muted"
              :color="muted ? 'primary' : 'default'" 
              class="mute-button" 
              density="compact">
              Mute
            </v-btn>
            <v-select
              v-model="temperament"
              :items="[Temperament.EvenTempered, Temperament.QuarterCommaMeantone,Temperament.ThirdCommaMeantone, Temperament.Just]"
              label="Select Temperament"
              density="compact"
              style="max-width: 250px; margin-top: 10px;"
            ></v-select>
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

enum Temperament {
  EvenTempered = "Even Tempered", 
  QuarterCommaMeantone = "1/4 Comma Meantone",
  ThirdCommaMeantone = "1/3 Comma Meantone",
  Just = "Just Intonation"
};

// Frequencies. TODO: encapsulate somewhere else.
const middle_c_freq = 261.6255653005986; // Middle C (C4) frequency in Hz
const even_tempered_ratio = Math.pow(2, 1 / 12); // Twelfth root of 2
const even_tempered_frequncies = Array.from({ length: 12 }, (_, i) => 
    middle_c_freq * Math.pow(even_tempered_ratio, i)
);

const qalpha = Math.pow(5, 0.25);
const qalphai = 1 / qalpha;
const quartercomma_ratios = [
    1, // C
    1 * qalphai ** 5 * 8, // C♯
    1 * qalpha ** 2 / 2, // D
    1 * qalphai ** 3 * 4, // E♭
    1 * qalpha ** 4 / 4, // E
    1 * qalphai * 2, // F
    1 * qalpha ** 6 / 8, // F♯
    1 * qalpha, // G
    1 * qalphai ** 4 * 8, // A♭
    1 * qalpha ** 3 / 2, // A
    1 * qalphai ** 2 * 4, // B♭
    1 * qalpha ** 5 / 4 // B
];

const talphai = Math.pow(3 / 10, 1 / 3);
const talpha = 1 / talphai; 
const third_comma_ratios = [
    1, // C
    1 * talphai ** 5 * 8, // C♯
    1 * talpha ** 2 / 2, // D
    1 * talphai ** 3 * 4, // E♭
    1 * talpha ** 4 / 4, // E
    1 * talphai * 2, // F
    1 * talpha ** 6 / 8, // F♯
    1 * talpha, // G
    1 * talphai ** 4 * 8, // A♭
    1 * talpha ** 3 / 2, // A
    1 * talphai ** 2 * 4, // B♭
    1 * talpha ** 5 / 4 // B
];

const just_ratios = [
    1,          // C
    16 / 15,   // C♯
    9 / 8,     // D
    6 / 5,     // E♭
    5 / 4,     // E
    4 / 3,     // F
    45 / 32,   // F♯
    3 / 2,     // G
    8 / 5,     // A♭
    5 / 3,     // A
    16 / 9,     // B♭
    15 / 8     // B
];

const frequencies = computed(() => {
  if (temperament.value === Temperament.QuarterCommaMeantone) {
    return quartercomma_ratios.map(ratio => middle_c_freq * ratio);
  } else if (temperament.value === Temperament.ThirdCommaMeantone) {
    return third_comma_ratios.map(ratio => middle_c_freq * ratio);
  } else if (temperament.value === Temperament.Just) {
    return just_ratios.map(ratio => middle_c_freq * ratio);
  } else {
    return even_tempered_frequncies;
  }
});

const temperament = ref(Temperament.EvenTempered);

const audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
const gain = audioCtx.createGain();
gain.gain.setValueAtTime(0.2, audioCtx.currentTime); // Set volume
gain.connect(audioCtx.destination);

let oscillators = [] as OscillatorNode[];
async function create_oscillators() {
  if (oscillators.length > 0) {
    await clear_oscillators();
  }
  await set_gain(0.0001);
  oscillators = sortedNotes.value.map(index => {
    const osc = audioCtx.createOscillator();
    osc.type = 'sine';
    const baseFreq = frequencies.value[index % 12] ?? middle_c_freq;
    const octaveMultiplier = Math.pow(2, Math.floor(index / 12));
    const freq = baseFreq * octaveMultiplier;
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
    osc.connect(gain);
    osc.start();
    return osc;
  });
  await set_gain(muted.value ? 0.0001 : 0.2);
}

async function clear_oscillators() {
  if (oscillators.length === 0) return;
  await set_gain(0.0001);
  oscillators.forEach(osc => {
    osc.stop();
    osc.disconnect();
  });
  oscillators = [];
  await set_gain(muted.value ? 0.0001 : 0.2);
}

const notenames = ref(['C', 'D♭', 'D', 'E♭', 'E', 'F', 'F♯', 'G', 'A♭', 'A', 'B♭', 'B']);
const rawSelection = ref([] as number[]);

const muted = ref(false);

watch(rawSelection, (newVal: number[], oldValue: number[]) => {
    create_oscillators();
});

watch(temperament, () => {
    // Recreate oscillators to reflect frequency changes
    create_oscillators();
});

watch(muted, (newVal) => {
    if (newVal) {
      set_gain(0.0001);
    } else {
      set_gain(0.2);
    }
});

async function set_gain(value: number) {
  const delay = 0.02;
  gain.gain.setValueAtTime(gain.gain.value, audioCtx.currentTime); 
  gain.gain.exponentialRampToValueAtTime(value, audioCtx.currentTime + delay);
  return new Promise(resolve => setTimeout(resolve, delay * 1000));
}

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
.clear-button, .mute-button {
  margin: 10px 5px 5px 0;
}
</style>