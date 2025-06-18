<template> 
  <div class="dual-mic-recorder">
    <h2>Grabadora de Doble Micrófono</h2>

    <!-- MONITOR -->
    <div class="monitor-controls">
      <button @click="toggleMonitor(1)">
        {{ monitoringMic1 ? 'Detener Monitor Mic 1' : 'Iniciar Monitor Mic 1' }}
      </button>
      <button @click="toggleMonitor(2)">
        {{ monitoringMic2 ? 'Detener Monitor Mic 2' : 'Iniciar Monitor Mic 2' }}
      </button>
    </div>

    <div>
      <h3>Monitor Mic 1</h3>
      <canvas ref="canvasMonitor1" width="600" height="100"></canvas>
      <canvas ref="vuMeterMonitor1" class="vu-meter" width="300" height="20"></canvas>
    </div>

    <div>
      <h3>Monitor Mic 2</h3>
      <canvas ref="canvasMonitor2" width="600" height="100"></canvas>
      <canvas ref="vuMeterMonitor2" class="vu-meter" width="300" height="20"></canvas>
    </div>

    <!-- SELECCIÓN -->
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

      <div class="settings">
        <label>
          Frecuencia:
          <input type="number" v-model.number="sampleRate" min="8000" max="96000" step="1000" />
        </label>
        <label>
          Bits:
          <select v-model.number="bitDepth">
            <option :value="8">8</option>
            <option :value="16">16</option>
            <option :value="24">24</option>
            <option :value="32">32 (Float)</option>
          </select>
        </label>
      </div>

      <div class="controls">
        <button @click="startRecording" :disabled="isRecording">Grabar</button>
        <button @click="stopRecording" :disabled="!isRecording">Detener</button>
      </div>

      <div>
        <h3>Pista 1</h3>
        <div ref="waveform1" class="waveform"></div>
        <div ref="spectrogram1" class="spectrogram"></div>
        <audio ref="audio1" :src="audioURL1" @play="syncWaveformPlay(1)" @pause="syncWaveformPause(1)" controls></audio>
        <a :href="audioURL1" download="pista1.wav">Descargar</a>
        <canvas ref="vuMeter1" class="vu-meter" width="300" height="20"></canvas>
      </div>

      <div>
        <h3>Pista 2</h3>
        <div ref="waveform2" class="waveform"></div>
        <div ref="spectrogram2" class="spectrogram"></div>
        <audio ref="audio2" :src="audioURL2" @play="syncWaveformPlay(2)" @pause="syncWaveformPause(2)" controls></audio>
        <a :href="audioURL2" download="pista2.wav">Descargar</a>
        <canvas ref="vuMeter2" class="vu-meter" width="300" height="20"></canvas>
      </div>
    </div>

    <div v-else>
      <p>Se necesitan al menos dos micrófonos.</p>
    </div>
  </div>
</template>

<script>
import WaveSurfer from 'wavesurfer.js';
// Importar el plugin espectrograma desde la versión ESM
import Spectrogram from 'wavesurfer.js/dist/plugins/spectrogram.esm.js';

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
      audioURL2: null,
      sampleRate: 44100,
      bitDepth: 24,
      waveform1: null,
      waveform2: null,
      playingTrack: null,
      monitoringMic1: false,
      monitoringMic2: false,
      monitorStream1: null,
      monitorStream2: null,
      monitorAudioContext1: null,
      monitorAudioContext2: null,
      monitorSource1: null,
      monitorSource2: null,
      monitorAnalyser1: null,
      monitorAnalyser2: null,
      monitorAnimationId1: null,
      monitorAnimationId2: null,
      playbackContexts: [null, null],
      playbackAnalyserNodes: [null, null],
      playbackAnimationIds: [null, null],
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
      const stream1 = await navigator.mediaDevices.getUserMedia({ audio: { deviceId: this.selectedMic1Id } });
      const stream2 = await navigator.mediaDevices.getUserMedia({ audio: { deviceId: this.selectedMic2Id } });

      this.recorder1 = new MediaRecorder(stream1);
      this.recorder2 = new MediaRecorder(stream2);
      this.chunks1 = [];
      this.chunks2 = [];

      this.recorder1.ondataavailable = e => this.chunks1.push(e.data);
      this.recorder2.ondataavailable = e => this.chunks2.push(e.data);

      this.recorder1.onstop = () => {
        const blob = new Blob(this.chunks1, { type: 'audio/webm' });
        this.audioURL1 = URL.createObjectURL(blob);
        this.loadWaveform(1);
      };
      this.recorder2.onstop = () => {
        const blob = new Blob(this.chunks2, { type: 'audio/webm' });
        this.audioURL2 = URL.createObjectURL(blob);
        this.loadWaveform(2);
      };

      this.recorder1.start();
      this.recorder2.start();
      this.isRecording = true;
    },

    stopRecording() {
      if (this.recorder1 && this.recorder1.state !== 'inactive') this.recorder1.stop();
      if (this.recorder2 && this.recorder2.state !== 'inactive') this.recorder2.stop();
      this.isRecording = false;
    },

    loadWaveform(micNumber) {
      const waveformRef = micNumber === 1 ? 'waveform1' : 'waveform2';
      const spectrogramRef = micNumber === 1 ? 'spectrogram1' : 'spectrogram2';
      const audioURL = micNumber === 1 ? this.audioURL1 : this.audioURL2;
      const existingWaveform = micNumber === 1 ? this.waveform1 : this.waveform2;

      if (existingWaveform) existingWaveform.destroy();

      const ws = WaveSurfer.create({
        container: this.$refs[waveformRef],
        waveColor: micNumber === 1 ? 'violet' : 'lightblue',
        progressColor: micNumber === 1 ? 'purple' : 'blue',
        height: 100,
        sampleRate: this.sampleRate,
      });

      // Registrar el plugin espectrograma
      ws.registerPlugin(
        Spectrogram.create({
          container: this.$refs[spectrogramRef],
          labels: true,
          height: 200,
          splitChannels: true,
          scale: 'mel', // otras opciones: 'linear', 'logarithmic', 'bark', 'erb'
          frequencyMax: 8000,
          frequencyMin: 0,
          fftSamples: 1024,
          labelsBackground: 'rgba(0, 0, 0, 0.1)',
        })
      );

      ws.load(audioURL);

      if (micNumber === 1) this.waveform1 = ws;
      else this.waveform2 = ws;
    },

    syncWaveformPlay(micNumber) {
      const audioRef = micNumber === 1 ? this.$refs.audio1 : this.$refs.audio2;
      if (!audioRef) return;

      if (micNumber === 1 && this.waveform2) this.waveform2.pause();
      if (micNumber === 2 && this.waveform1) this.waveform1.pause();

      const waveform = micNumber === 1 ? this.waveform1 : this.waveform2;
      if (waveform) waveform.play();

      this.setupVUMeterPlayback(micNumber);
    },

    syncWaveformPause(micNumber) {
      if (micNumber === 1 && this.waveform1) this.waveform1.pause();
      if (micNumber === 2 && this.waveform2) this.waveform2.pause();
      cancelAnimationFrame(this.playbackAnimationIds[micNumber - 1]);
    },

    setupVUMeterPlayback(micNumber) {
      const audio = micNumber === 1 ? this.$refs.audio1 : this.$refs.audio2;
      const canvas = micNumber === 1 ? this.$refs.vuMeter1 : this.$refs.vuMeter2;

      if (!audio || !canvas) return;

      if (this.playbackContexts[micNumber - 1]) {
        this.playbackContexts[micNumber - 1].close();
      }

      const audioContext = new AudioContext();
      const source = audioContext.createMediaElementSource(audio);
      const analyser = audioContext.createAnalyser();
      analyser.fftSize = 256;
      source.connect(analyser);
      analyser.connect(audioContext.destination);

      this.playbackContexts[micNumber - 1] = audioContext;
      this.playbackAnalyserNodes[micNumber - 1] = analyser;

      const ctx = canvas.getContext('2d');
      const width = canvas.width;
      const height = canvas.height;
      const dataArray = new Uint8Array(analyser.frequencyBinCount);

      const draw = () => {
        analyser.getByteFrequencyData(dataArray);
        const avg = dataArray.reduce((a, b) => a + b, 0) / dataArray.length;
        ctx.clearRect(0, 0, width, height);
        ctx.fillStyle = 'lime';
        ctx.fillRect(0, 0, (avg / 255) * width, height);
        this.playbackAnimationIds[micNumber - 1] = requestAnimationFrame(draw);
      };
      draw();
    },

    toggleMonitor(micNumber) {
      if (micNumber === 1) {
        if (this.monitoringMic1) {
          this.stopMonitor(1);
        } else {
          this.startMonitor(1);
        }
      } else if (micNumber === 2) {
        if (this.monitoringMic2) {
          this.stopMonitor(2);
        } else {
          this.startMonitor(2);
        }
      }
    },

    async startMonitor(micNumber) {
      const deviceId = micNumber === 1 ? this.selectedMic1Id : this.selectedMic2Id;
      const stream = await navigator.mediaDevices.getUserMedia({ audio: { deviceId } });
      const canvasMonitor = micNumber === 1 ? this.$refs.canvasMonitor1 : this.$refs.canvasMonitor2;
      const vuMeterCanvas = micNumber === 1 ? this.$refs.vuMeterMonitor1 : this.$refs.vuMeterMonitor2;

      const audioContext = new AudioContext();
      const source = audioContext.createMediaStreamSource(stream);
      const analyser = audioContext.createAnalyser();
      analyser.fftSize = 256;
      source.connect(analyser);

      // Connect to destination to enable monitoring
      source.connect(audioContext.destination);

      const ctx = canvasMonitor.getContext('2d');
      const vuCtx = vuMeterCanvas.getContext('2d');
      const bufferLength = analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);

      const drawMonitor = () => {
        analyser.getByteTimeDomainData(dataArray);

        ctx.fillStyle = 'black';
        ctx.fillRect(0, 0, canvasMonitor.width, canvasMonitor.height);

        ctx.lineWidth = 2;
        ctx.strokeStyle = 'lime';

        ctx.beginPath();
        const sliceWidth = canvasMonitor.width / bufferLength;
        let x = 0;

        for (let i = 0; i < bufferLength; i++) {
          const v = dataArray[i] / 128.0;
          const y = (v * canvasMonitor.height) / 2;
          if (i === 0) ctx.moveTo(x, y);
          else ctx.lineTo(x, y);
          x += sliceWidth;
        }
        ctx.lineTo(canvasMonitor.width, canvasMonitor.height / 2);
        ctx.stroke();

        // VU Meter
        const avg = dataArray.reduce((a, b) => a + b, 0) / dataArray.length;
        vuCtx.clearRect(0, 0, vuMeterCanvas.width, vuMeterCanvas.height);
        vuCtx.fillStyle = 'red';
        vuCtx.fillRect(0, 0, (avg / 255) * vuMeterCanvas.width, vuMeterCanvas.height);

        if (micNumber === 1) this.monitorAnimationId1 = requestAnimationFrame(drawMonitor);
        else this.monitorAnimationId2 = requestAnimationFrame(drawMonitor);
      };

      drawMonitor();

      if (micNumber === 1) {
        this.monitorStream1 = stream;
        this.monitorAudioContext1 = audioContext;
        this.monitorSource1 = source;
        this.monitorAnalyser1 = analyser;
        this.monitoringMic1 = true;
      } else {
        this.monitorStream2 = stream;
        this.monitorAudioContext2 = audioContext;
        this.monitorSource2 = source;
        this.monitorAnalyser2 = analyser;
        this.monitoringMic2 = true;
      }
    },

    stopMonitor(micNumber) {
      if (micNumber === 1) {
        if (this.monitorAnimationId1) cancelAnimationFrame(this.monitorAnimationId1);
        if (this.monitorAudioContext1) this.monitorAudioContext1.close();
        if (this.monitorStream1) {
          this.monitorStream1.getTracks().forEach(track => track.stop());
          this.monitorStream1 = null;
        }
        this.monitoringMic1 = false;
      } else {
        if (this.monitorAnimationId2) cancelAnimationFrame(this.monitorAnimationId2);
        if (this.monitorAudioContext2) this.monitorAudioContext2.close();
        if (this.monitorStream2) {
          this.monitorStream2.getTracks().forEach(track => track.stop());
          this.monitorStream2 = null;
        }
        this.monitoringMic2 = false;
      }
    }
  },
  beforeDestroy() {
    // Limpiar WaveSurfer y audio contexts
    if (this.waveform1) this.waveform1.destroy();
    if (this.waveform2) this.waveform2.destroy();

    if (this.monitorAudioContext1) this.monitorAudioContext1.close();
    if (this.monitorAudioContext2) this.monitorAudioContext2.close();

    this.playbackContexts.forEach(ctx => {
      if (ctx) ctx.close();
    });
  }
};
</script>

<style scoped>
.dual-mic-recorder {
  font-family: Arial, sans-serif;
  max-width: 800px;
  margin: auto;
}

.monitor-controls {
  margin-bottom: 10px;
}

.controls button {
  margin-right: 10px;
  margin-top: 10px;
}

.waveform, .spectrogram {
  margin-top: 10px;
  border: 1px solid #ccc;
}

.vu-meter {
  margin-top: 5px;
  background: #222;
  border-radius: 4px;
}
</style>
