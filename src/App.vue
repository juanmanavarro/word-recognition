<template>
  <div style="width:500px" class="container mt-5">
    {{ isReady }}
    <div class="row mb-3">
      <div class="col">
        <div class="card">
          <div class="card-header">
            <div class="">1. Configure</div>
          </div>
          <div class="card-body">
            <div class="input-group mb-3">
              <input
                @keyup.enter="createWord"
                ref="input"
                v-model="word"
                type="text"
                class="form-control"
                placeholder="Word"
                aria-label="Word"
              >
              <button
                @click="createWord"
                class="btn btn-outline-secondary"
                type="button"
              >
                Add
              </button>
            </div>
            <table class="w-100">
              <tr>
                <td>Background noise</td>
                <td class="text-end">
                  <button
                    @click="recordExample('_background_noise_')"
                    class="btn btn-sm btn-primary"
                    :disabled="recording"
                  >
                    Record example <span v-if="examples['_background_noise_']">({{ examples['_background_noise_'] }})</span>
                  </button>
                </td>
              </tr>
              <tr v-for="word in words">
                <td>{{ word }}</td>
                <td class="text-end">
                  <button
                    @click="recordExample(word)"
                    class="btn btn-sm btn-primary"
                    :disabled="recording"
                  >
                    Record example <span v-if="examples[word]">({{ examples[word] }})</span>
                  </button>
                </td>
              </tr>
            </table>
          </div>
        </div>
      </div>
    </div>
    <div class="row mb-3">
      <div class="col">
        <div class="card">
          <div class="card-header">
            <div class="">2. Train</div>
          </div>
          <div class="card-body">
            <div class="mb-3 row">
              <label for="inputPassword" class="col-sm-2 col-form-label">Epochs</label>
              <div class="col-sm-10">
                <input type="number" v-model="epochs" class="form-control" id="inputPassword">
              </div>
            </div>
            <button class="btn btn-primary" @click="train">Train</button>
            <span class="ms-4">{{ trainResult }}</span>
          </div>
        </div>
      </div>
    </div>
    <div class="row">
      <div class="col">
        <div class="card">
          <div class="card-header">
            <div class="">2. Test</div>
          </div>
          <div class="card-body">
            <button v-if="!listening" class="btn btn-primary" @click="listen">Start listening</button>
            <button v-else class="btn btn-primary" @click="stopListen">Stop listening</button>
            <h4>{{ recognizedWord }}</h4>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import * as ts from '@tensorflow/tfjs';
import * as speechCommands from '@tensorflow-models/speech-commands';
import { onMounted, ref, watch } from 'vue';

const isReady = ref(true);
const word = ref('');
const words = ref([]);
const examples = ref({});
const input = ref();
const status = ref('config');
const trainResult = ref('');
const listening = ref(false);
const epochs = ref(25);
const recognizedWord = ref('');
const recording = ref(false);

const createWord = () => {
  if ( words.value.includes(word.value) || !word.value ) return;
  words.value.push(word.value);
  word.value = '';
  input.value.focus();
}

const recordExample = async (word) => {
  if ( !transferRecognizer ) return;
  recording.value = true;
  await transferRecognizer.collectExample(word);
  examples.value = { ...examples.value, [word]: transferRecognizer.countExamples()[word] };
  recording.value = false;
  console.log(examples.value);
}

const train = async () => {
  await transferRecognizer.train({
    batchSize: words.value.length,
    epochs: epochs.value,
    callback: {
      onEpochEnd: async (epoch, logs) => {
        console.log(`Epoch ${epoch}: loss=${logs.loss}, accuracy=${logs.acc}`);
        trainResult.value = `Epoch ${epoch+1}: loss=${logs.loss}, accuracy=${logs.acc}`;
      }
    }
  });
}

const listen = async () => {
  listening.value = true;
  await transferRecognizer.listen(result => {
    const scores = Array.from(result.scores);
    const max = Math.max(...scores);
    const index =  scores.indexOf(max);
    console.log(transferRecognizer.wordLabels()[index]);
    recognizedWord.value = transferRecognizer.wordLabels()[index];
    // document.getElementById('result').innerHTML = recognizer.wordLabels()[index];
    // // - result.scores contains the scores for the new vocabulary, which
    // //   can be checked with:
    // const words = transferRecognizer.wordLabels();
    // // `result.scores` contains the scores for the new words, not the original
    // // words.
    // for (let i = 0; i < words.length; ++i) {
    //   console.log(`score for word '${words[i]}' = ${result.scores[i]}`);
    // }
  }, {probabilityThreshold: 0.75});
}

const stopListen = () => {
  transferRecognizer.stopListening();
  listening.value = false;
}

let transferRecognizer;

watch(words.value, async (newWords, oldWords) => {
  if ( newWords.length >= oldWords.length ) {
    await createTransferRecognizer();
  }
})

onMounted(async () => await createTransferRecognizer());

async function createTransferRecognizer() {
  isReady.value = false;
  const recognizer = speechCommands.create('BROWSER_FFT');
  await recognizer.ensureModelLoaded();
  transferRecognizer = recognizer.createTransfer('demo');
  isReady.value = true;
}

function download() {
  const serialized = transferRecognizer.serializeExamples();
  var file = new Blob([serialized], { type: 'text/plain' });
  if (window.navigator.msSaveOrOpenBlob) // IE10+
    window.navigator.msSaveOrOpenBlob(file, 'model.txt');
  else { // Others
    var a = document.createElement("a"),
      url = URL.createObjectURL(file);
    a.href = url;
    a.download = 'model.txt';
    document.body.appendChild(a);
    a.click();
    setTimeout(function () {
      document.body.removeChild(a);
      window.URL.revokeObjectURL(url);
    }, 0);
  }
}
</script>
