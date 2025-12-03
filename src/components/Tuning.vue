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
            <div class="current-frequencies" style="margin-top: 10px;">
              <h4>Current Frequencies:</h4>
              {{  current_frequencies.join(', ') }}
            </div>

            <div v-if="current_ratios.length > 0" class="current-frequencies" style="margin-top: 10px;">
              <h4>Current Ratios:</h4>
              {{  current_ratios.join(', ') }}
            </div>
           
              <div class="d-flex align-center" style="gap:8px; margin-top:8px;">
                <v-select
                  v-model="temperament"
                  :items="[Temperament.EvenTempered, Temperament.QuarterCommaMeantone, Temperament.ThirdCommaMeantone, Temperament.Just]"
                  density="compact"
                  hide-details
                  style="max-width: 250px;"
                ></v-select>

                <v-btn 
                  @click="just_chords = !just_chords;" 
                  :active="just_chords"
                  :color="just_chords ? 'primary' : 'default'" 
                  class="just-chords-button" 
                  density="comfortable">
                  Just Chords
                </v-btn>
              </div>
              <v-expansion-panels 
                v-model="oscilloscopeRunning">
                <v-expansion-panel
                  title="Oscilloscope"
                  density="compact"
                  value=true>
                  <v-expansion-panel-text>
                    <canvas id="oscilloscope-canvas" width="400" height="200" style="border:1px solid #ccc; margin-top: 10px;"></canvas>
                  </v-expansion-panel-text>
                </v-expansion-panel>
              </v-expansion-panels>
            
          <!--
                <div class="selected-notes">
                <h3>Selected Notes:</h3>
                {{  sortedNotes.map(index => notenames[index % 12]).join(', ') }}
                </div>
          -->
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script lang="ts" setup>
import { ref, computed, watch, onUnmounted, onMounted } from 'vue';

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
const just_chords = ref(false);

const muted = ref(false);

const audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
const gain: GainNode = audioCtx.createGain();
gain.gain.setValueAtTime(0.2, audioCtx.currentTime); // Set volume
gain.connect(audioCtx.destination);

// For the oscilloscope
const analyser = audioCtx.createAnalyser();
gain.connect(analyser);
analyser.fftSize = 8192;

let oscillators = ref([] as OscillatorNode[]);

function desiredFrequencies() {
  if (sortedNotes.value.length === 0) {
    return [];
  } else if (just_chords.value) {
    const rootIdx = sortedNotes.value[0] ?? 0;
    const rootFreq = (frequencies.value[rootIdx % 12] ?? 1) * Math.pow(2, Math.floor(rootIdx / 12));
    return sortedNotes.value.map(index => {
      const offset = index - rootIdx;
      const baseFreq = just_ratios[offset % 12] ?? 1;
      const octaveMultiplier = Math.pow(2, Math.floor(offset / 12));
      return rootFreq * baseFreq * octaveMultiplier;
    });
  } else {
    return sortedNotes.value.map(index => {
      const baseFreq = frequencies.value[index % 12] ?? middle_c_freq;
      const octaveMultiplier = Math.pow(2, Math.floor(index / 12));
      return baseFreq * octaveMultiplier;
    });
  }
}

const current_frequencies = computed(() => {
  return desiredFrequencies().map(freq => freq.toFixed(2));
});

const current_ratios = computed(() => {
  const abs_frequencies = desiredFrequencies();
  const results: string[] = [];
  let last = 0;
  if (abs_frequencies.length > 0) {
    const root = abs_frequencies[0] ?? 1
    for (let f of abs_frequencies.slice(1)) {
      const ratio = f / root;
      results.push(ratio.toFixed(4));
    }
  }
  return results;
});

const desired_gain = computed(() => {
  if (muted.value) {
    return 0.0001;
  } else if (oscillators.value.length > 0) {
    return Math.min(0.2, 0.6 / oscillators.value.length);
  } else {
    return 0.2;
  }
});

async function createOscillators() {
  const freqs = desiredFrequencies();
  if (oscillators.value.length === freqs.length) {
    for (const [index, osc] of oscillators.value.entries()) {
      osc.frequency.setValueAtTime(freqs[index] ?? osc.frequency.value, audioCtx.currentTime);
    }
    return;
  }
  if (oscillators.value.length > 0) {
    await clearOscillators();
  }
  await setGain(0.0001);
  oscillators.value = freqs.map((freq, index) => {
    const osc = audioCtx.createOscillator();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
    osc.connect(gain);
    osc.start();
    return osc;
  });
  await setGain(desired_gain.value);
}

async function clearOscillators() {
  if (oscillators.value.length === 0) return;
  await setGain(0.0001);
  oscillators.value.forEach(osc => {
    osc.stop();
    osc.disconnect();
  });
  oscillators.value = [];
  await setGain(desired_gain.value);
}

const notenames = ref(['C', 'D♭', 'D', 'E♭', 'E', 'F', 'F♯', 'G', 'A♭', 'A', 'B♭', 'B']);
const rawSelection = ref([] as number[]);

watch(rawSelection, (newVal: number[], oldValue: number[]) => {
    createOscillators();
});

watch(temperament, () => {
    // Recreate oscillators to reflect frequency changes
    createOscillators();
});

watch(just_chords, () => {
    // Recreate oscillators to reflect frequency changes
    createOscillators();
});

watch(muted, (newVal) => {
  setGain(desired_gain.value);
});

async function setGain(value: number) {
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

// Oscilloscope drawing
const oscilloscopeRunning = ref(false)

function drawOscilloscope() {
    const canvasElement = document.getElementById('oscilloscope-canvas');
    const canvas = canvasElement as HTMLCanvasElement;
    if (!canvas) return;
    const canvasCtx = canvas.getContext('2d');
    if (!canvasCtx) return;

    const bufferLength = analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);

    let count = 0;

    function draw() {
      if (!canvasCtx) return;
      if (!oscilloscopeRunning.value) return;
      requestAnimationFrame(draw);

      canvasCtx.stroke();

      analyser.getByteTimeDomainData(dataArray);

      canvasCtx.fillStyle = 'rgb(200, 200, 200)';
      canvasCtx.fillRect(0, 0, canvas.width, canvas.height);

      canvasCtx.lineWidth = 2;
      canvasCtx.strokeStyle = 'rgb(0, 0, 0)';

      canvasCtx.beginPath();

      const sliceWidth = canvas.width * 2.0 / bufferLength;
      let x = 0;

      // We want to begin at a multiple of the current frequency
      let offset = 0;
      const current_frequencies = desiredFrequencies();
      if (current_frequencies.length != 0) {
        const freq = current_frequencies[0]!;
        const t = audioCtx.currentTime;
        const period = 4/freq;
        const slop = period - t % period;
        const sampleRate = audioCtx.sampleRate;
        offset = Math.round(slop * sampleRate);
      }

      for (let i = 0; i < bufferLength/2; i++) {
        const idx = i + bufferLength / 2 - offset;
          const v = dataArray[i + offset]! / 128.0;
          const y = v * canvas.height / 2;

          if (i === 0) {
              canvasCtx.moveTo(x, y);
          } else {
              canvasCtx.lineTo(x, y);
          }

          x += sliceWidth;
      }

      canvasCtx.stroke();
    }

    draw();
}

watch(oscilloscopeRunning, (newVal) => {
    if (newVal) {
      setTimeout(drawOscilloscope, 0);
    }
});

onMounted(() => {
    drawOscilloscope();
});

onUnmounted(() => {
    clearOscillators().then(() => audioCtx.close());
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