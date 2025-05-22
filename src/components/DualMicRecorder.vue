<template>
    <div class="dual-mic-recorder">
      <h2>Grabadora de Doble Micrófono</h2>
  
      <div v-if="audioInputs.length >= 2">
        <label>Micrófono 1:
          <select v-model="selectedMic1Id">
            <option v-for="device in audioInputs" :key="device.deviceId" :value="device.deviceId">
              {{ device.label || 'Micrófono ' + device.deviceId }}
            </option>
          </select>
        </label>
        <br>
        <label>Micrófono 2:
          <select v-model="selectedMic2Id">
            <option v-for="device in audioInputs" :key="device.deviceId" :value="device.deviceId">
              {{ device.label || 'Micrófono ' + device.deviceId }}
            </option>
          </select>
        </label>
  
        <div class="controls">
          <button @click="startRecording" :disabled="isRecording">Grabar</button>
          <button @click="stopRecording" :disabled="!isRecording">Detener</button>
        </div>
  
        <div v-if="audioURL1">
          <h3>Pista 1</h3>
          <audio :src="audioURL1" controls></audio>
          <a :href="audioURL1" download="pista1.webm">Descargar Pista 1</a>
        </div>
  
        <div v-if="audioURL2">
          <h3>Pista 2</h3>
          <audio :src="audioURL2" controls></audio>
          <a :href="audioURL2" download="pista2.webm">Descargar Pista 2</a>
        </div>
      </div>
  
      <div v-else>
        <p>Se necesitan al menos dos micrófonos conectados.</p>
      </div>
    </div>
  </template>
  
  <script>
  export default {
    name: 'DualMicRecorder',
    data() {
      return {
        audioInputs: [],
        selectedMic1Id: '',
        selectedMic2Id: '',
        isRecording: false,
        recorder1: null,
        recorder2: null,
        chunks1: [],
        chunks2: [],
        audioURL1: null,
        audioURL2: null
      };
    },
    async mounted() {
      const devices = await navigator.mediaDevices.enumerateDevices();
      this.audioInputs = devices.filter(device => device.kind === 'audioinput');
      if (this.audioInputs.length >= 2) {
        this.selectedMic1Id = this.audioInputs[0].deviceId;
        this.selectedMic2Id = this.audioInputs[1].deviceId;
      }
    },
    methods: {
      async startRecording() {
        try {
          const stream1 = await navigator.mediaDevices.getUserMedia({ audio: { deviceId: this.selectedMic1Id } });
          const stream2 = await navigator.mediaDevices.getUserMedia({ audio: { deviceId: this.selectedMic2Id } });
  
          this.chunks1 = [];
          this.chunks2 = [];
  
          this.recorder1 = new MediaRecorder(stream1);
          this.recorder2 = new MediaRecorder(stream2);
  
          this.recorder1.ondataavailable = e => this.chunks1.push(e.data);
          this.recorder2.ondataavailable = e => this.chunks2.push(e.data);
  
          this.recorder1.onstop = () => {
            const blob1 = new Blob(this.chunks1, { type: 'audio/webm' });
            this.audioURL1 = URL.createObjectURL(blob1);
          };
  
          this.recorder2.onstop = () => {
            const blob2 = new Blob(this.chunks2, { type: 'audio/webm' });
            this.audioURL2 = URL.createObjectURL(blob2);
          };
  
          this.recorder1.start();
          this.recorder2.start();
          this.isRecording = true;
        } catch (err) {
          console.error('Error al iniciar la grabación:', err);
          alert('No se pudo acceder a uno o ambos micrófonos.');
        }
      },
      stopRecording() {
        if (this.recorder1 && this.recorder1.state !== 'inactive') this.recorder1.stop();
        if (this.recorder2 && this.recorder2.state !== 'inactive') this.recorder2.stop();
        this.isRecording = false;
      }
    }
  };
  </script>
  
  <style scoped>
  .dual-mic-recorder {
    padding: 1rem;
    border: 1px solid #ccc;
    border-radius: 8px;
    max-width: 600px;
    margin: auto;
  }
  .controls {
    margin: 1rem 0;
  }
  audio {
    display: block;
    margin-top: 0.5rem;
  }
  </style>
  