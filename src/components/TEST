<template>
    <div class="stereo-recorder">
      <h2>Grabadora 2 Canales - Scarlett 2i2</h2>
  
      <div v-if="audioInputs.length">
        <label>
          Selecciona dispositivo:
          <select v-model="selectedDeviceId">
            <option v-for="device in audioInputs" :key="device.deviceId" :value="device.deviceId">
              {{ device.label || 'Dispositivo ' + device.deviceId }}
            </option>
          </select>
        </label>
  
        <br /><br />
  
        <label>
          Sample Rate:
          <select v-model.number="sampleRate">
            <option :value="44100">44100 Hz</option>
            <option :value="48000">48000 Hz</option>
            <option :value="96000">96000 Hz</option>
          </select>
        </label>
  
        <label style="margin-left: 20px;">
          Bit Depth:
          <select v-model.number="bitDepth">
            <option :value="16">16 bits</option>
            <option :value="24">24 bits</option>
          </select>
        </label>
  
        <div class="buttons" style="margin-top:10px;">
          <button @click="start" :disabled="isRecording">Grabar</button>
          <button @click="stop" :disabled="!isRecording">Detener</button>
        </div>
  
        <div v-if="audioURLs[0] || audioURLs[1]">
          <div class="channel">
            <h3>Canal Izquierdo</h3>
            <div ref="waveformLeft" class="waveform"></div>
            <canvas ref="spectrogramLeft" class="spectrogram"></canvas>
            <canvas ref="vumeterLeft" class="vumeter"></canvas>
            <audio :src="audioURLs[0]" controls></audio>
            <a :href="audioURLs[0]" download="canal-izquierdo.wav">Descargar Izquierdo</a>
          </div>
  
          <div class="channel">
            <h3>Canal Derecho</h3>
            <div ref="waveformRight" class="waveform"></div>
            <canvas ref="spectrogramRight" class="spectrogram"></canvas>
            <canvas ref="vumeterRight" class="vumeter"></canvas>
            <audio :src="audioURLs[1]" controls></audio>
            <a :href="audioURLs[1]" download="canal-derecho.wav">Descargar Derecho</a>
          </div>
        </div>
      </div>
      <div v-else>
        <p>No se encontraron dispositivos de audio.</p>
      </div>
    </div>
  </template>
  
  <script>
  import WaveSurfer from "wavesurfer.js";
  
  export default {
    name: "StereoRecorder",
    data() {
      return {
        audioInputs: [],
        selectedDeviceId: null,
        isRecording: false,
        audioContext: null,
        processor: null,
        sourceNode: null,
        splitter: null,
        analyserLeft: null,
        analyserRight: null,
        leftChannelData: [],
        rightChannelData: [],
        bufferSize: 4096,
        sampleRate: 44100,
        bitDepth: 16, // Nuevo: bit depth (16 o 24)
        audioURLs: [null, null],
        waveformLeft: null,
        waveformRight: null,
        stream: null,
        fftSize: 512,
        spectrogramWidth: 600,
        spectrogramHeight: 128,
        spectrogramLeftCtx: null,
        spectrogramRightCtx: null,
        rafId: null,
        vumeterLeftCtx: null,
        vumeterRightCtx: null,
        vumeterWidth: 600,
        vumeterHeight: 50,
      };
    },
    async mounted() {
      const devices = await navigator.mediaDevices.enumerateDevices();
      this.audioInputs = devices.filter((d) => d.kind === "audioinput");
      if (this.audioInputs.length) {
        this.selectedDeviceId = this.audioInputs[0].deviceId;
      }
  
      this.initSpectrogramContexts();
      this.initVUMeterContexts();
    },
    methods: {
      initSpectrogramContexts() {
        const canvasLeft = this.$refs.spectrogramLeft;
        if (canvasLeft) {
          canvasLeft.width = this.spectrogramWidth;
          canvasLeft.height = this.spectrogramHeight;
          this.spectrogramLeftCtx = canvasLeft.getContext("2d");
          this.spectrogramLeftCtx.fillStyle = "black";
          this.spectrogramLeftCtx.fillRect(0, 0, this.spectrogramWidth, this.spectrogramHeight);
        }
        const canvasRight = this.$refs.spectrogramRight;
        if (canvasRight) {
          canvasRight.width = this.spectrogramWidth;
          canvasRight.height = this.spectrogramHeight;
          this.spectrogramRightCtx = canvasRight.getContext("2d");
          this.spectrogramRightCtx.fillStyle = "black";
          this.spectrogramRightCtx.fillRect(0, 0, this.spectrogramWidth, this.spectrogramHeight);
        }
      },
      initVUMeterContexts() {
        const vumeterLeft = this.$refs.vumeterLeft;
        if (vumeterLeft) {
          vumeterLeft.width = this.vumeterWidth;
          vumeterLeft.height = this.vumeterHeight;
          this.vumeterLeftCtx = vumeterLeft.getContext("2d");
          this.vumeterLeftCtx.fillStyle = "black";
          this.vumeterLeftCtx.fillRect(0, 0, this.vumeterWidth, this.vumeterHeight);
        }
        const vumeterRight = this.$refs.vumeterRight;
        if (vumeterRight) {
          vumeterRight.width = this.vumeterWidth;
          vumeterRight.height = this.vumeterHeight;
          this.vumeterRightCtx = vumeterRight.getContext("2d");
          this.vumeterRightCtx.fillStyle = "black";
          this.vumeterRightCtx.fillRect(0, 0, this.vumeterWidth, this.vumeterHeight);
        }
      },
      async start() {
        if (!this.selectedDeviceId) return alert("Selecciona un dispositivo");
  
        this.leftChannelData = [];
        this.rightChannelData = [];
  
        try {
          this.stream = await navigator.mediaDevices.getUserMedia({
            audio: {
              deviceId: this.selectedDeviceId,
              channelCount: 2,
              sampleRate: this.sampleRate,
            },
          });
  
          this.audioContext = new AudioContext({ sampleRate: this.sampleRate });
          this.sourceNode = this.audioContext.createMediaStreamSource(this.stream);
  
          this.processor = this.audioContext.createScriptProcessor(this.bufferSize, 2, 2);
  
          this.analyserLeft = this.audioContext.createAnalyser();
          this.analyserRight = this.audioContext.createAnalyser();
  
          this.analyserLeft.fftSize = this.fftSize;
          this.analyserRight.fftSize = this.fftSize;
  
          this.splitter = this.audioContext.createChannelSplitter(2);
  
          this.sourceNode.connect(this.splitter);
          this.splitter.connect(this.analyserLeft, 0);
          this.splitter.connect(this.analyserRight, 1);
          this.splitter.connect(this.processor);
  
          this.processor.onaudioprocess = (e) => {
            const left = e.inputBuffer.getChannelData(0);
            const right = e.inputBuffer.getChannelData(1);
  
            this.leftChannelData.push(new Float32Array(left));
            this.rightChannelData.push(new Float32Array(right));
          };
  
          this.processor.connect(this.audioContext.destination);
  
          this.isRecording = true;
  
          if (!this.spectrogramLeftCtx || !this.spectrogramRightCtx) {
            this.initSpectrogramContexts();
          }
          if (!this.vumeterLeftCtx || !this.vumeterRightCtx) {
            this.initVUMeterContexts();
          }
  
          this.updateVisuals();
        } catch (err) {
          alert("Error accediendo al micrófono: " + err.message);
        }
      },
      stop() {
        this.isRecording = false;
  
        if (this.processor) {
          this.processor.disconnect();
          this.processor.onaudioprocess = null;
          this.processor = null;
        }
        if (this.sourceNode) {
          this.sourceNode.disconnect();
          this.sourceNode = null;
        }
        if (this.splitter) {
          this.splitter.disconnect();
          this.splitter = null;
        }
        if (this.analyserLeft) {
          this.analyserLeft.disconnect();
          this.analyserLeft = null;
        }
        if (this.analyserRight) {
          this.analyserRight.disconnect();
          this.analyserRight = null;
        }
        if (this.stream) {
          this.stream.getTracks().forEach((t) => t.stop());
          this.stream = null;
        }
        if (this.audioContext) {
          this.audioContext.close();
          this.audioContext = null;
        }
  
        if (this.rafId) {
          cancelAnimationFrame(this.rafId);
          this.rafId = null;
        }
  
        const leftBuffer = this.mergeBuffers(this.leftChannelData);
        const rightBuffer = this.mergeBuffers(this.rightChannelData);
  
        const wavLeft = this.encodeWAV(leftBuffer, this.sampleRate, this.bitDepth);
        const wavRight = this.encodeWAV(rightBuffer, this.sampleRate, this.bitDepth);
  
        const blobLeft = new Blob([wavLeft], { type: "audio/wav" });
        const blobRight = new Blob([wavRight], { type: "audio/wav" });
  
        if (this.audioURLs[0]) URL.revokeObjectURL(this.audioURLs[0]);
        if (this.audioURLs[1]) URL.revokeObjectURL(this.audioURLs[1]);
  
        this.audioURLs[0] = URL.createObjectURL(blobLeft);
        this.audioURLs[1] = URL.createObjectURL(blobRight);
  
        this.loadWaveform(0, this.audioURLs[0], this.$refs.waveformLeft);
        this.loadWaveform(1, this.audioURLs[1], this.$refs.waveformRight);
  
        this.drawFullSpectrogram(leftBuffer, this.spectrogramLeftCtx);
        this.drawFullSpectrogram(rightBuffer, this.spectrogramRightCtx);
        this.clearVUMeters();
      },
      mergeBuffers(bufferArrays) {
        let length = 0;
        for (const arr of bufferArrays) {
          length += arr.length;
        }
        const result = new Float32Array(length);
        let offset = 0;
        for (const arr of bufferArrays) {
          result.set(arr, offset);
          offset += arr.length;
        }
        return result;
      },
      encodeWAV(samples, sampleRate, bitDepth) {
        const bytesPerSample = bitDepth / 8;
        const blockAlign = bytesPerSample * 1; // Mono = 1 channel since mono buffer for each channel
        const buffer = new ArrayBuffer(44 + samples.length * bytesPerSample);
        const view = new DataView(buffer);
  
        function writeString(view, offset, string) {
          for (let i = 0; i < string.length; i++) {
            view.setUint8(offset + i, string.charCodeAt(i));
          }
        }
  
        const length = 44 + samples.length * bytesPerSample;
  
        writeString(view, 0, "RIFF");
        view.setUint32(4, length - 8, true);
        writeString(view, 8, "WAVE");
  
        writeString(view, 12, "fmt ");
        view.setUint32(16, 16, true);
        view.setUint16(20, 1, true); // PCM
        view.setUint16(22, 1, true); // Mono
        view.setUint32(24, sampleRate, true);
        view.setUint32(28, sampleRate * blockAlign, true);
        view.setUint16(32, blockAlign, true);
        view.setUint16(34, bitDepth, true);
  
        writeString(view, 36, "data");
        view.setUint32(40, samples.length * bytesPerSample, true);
  
        let offset = 44;
        if (bitDepth === 16) {
          for (let i = 0; i < samples.length; i++) {
            let s = Math.max(-1, Math.min(1, samples[i]));
            s = s < 0 ? s * 0x8000 : s * 0x7fff;
            view.setInt16(offset, s, true);
            offset += 2;
          }
        } else if (bitDepth === 24) {
          // 24 bits: se guarda en 3 bytes little-endian
          for (let i = 0; i < samples.length; i++) {
            let s = Math.max(-1, Math.min(1, samples[i]));
            s = s < 0 ? s * 0x800000 : s * 0x7fffff;
            const val = Math.floor(s);
            view.setUint8(offset, val & 0xff);
            view.setUint8(offset + 1, (val >> 8) & 0xff);
            view.setUint8(offset + 2, (val >> 16) & 0xff);
            offset += 3;
          }
        } else {
          throw new Error("Bit depth no soportado");
        }
  
        return view;
      },
      loadWaveform(index, audioURL, container) {
        if (index === 0 && this.waveformLeft) {
          this.waveformLeft.destroy();
          this.waveformLeft = null;
        }
        if (index === 1 && this.waveformRight) {
          this.waveformRight.destroy();
          this.waveformRight = null;
        }
  
        const ws = WaveSurfer.create({
          container,
          waveColor: index === 0 ? "violet" : "lightblue",
          progressColor: "purple",
          height: 100,
        });
  
        ws.load(audioURL);
  
        if (index === 0) this.waveformLeft = ws;
        else this.waveformRight = ws;
      },
      updateVisuals() {
        if (!this.isRecording) return;
        if (!this.spectrogramLeftCtx || !this.spectrogramRightCtx) return;
        if (!this.vumeterLeftCtx || !this.vumeterRightCtx) return;
  
        try {
          const freqDataLeft = new Uint8Array(this.analyserLeft.frequencyBinCount);
          const freqDataRight = new Uint8Array(this.analyserRight.frequencyBinCount);
  
          this.analyserLeft.getByteFrequencyData(freqDataLeft);
          this.analyserRight.getByteFrequencyData(freqDataRight);
  
          // Actualizar espectrogramas
          this.shiftSpectrogram(this.spectrogramLeftCtx, this.spectrogramWidth, this.spectrogramHeight);
          this.shiftSpectrogram(this.spectrogramRightCtx, this.spectrogramWidth, this.spectrogramHeight);
  
          this.drawSpectrogramColumn(this.spectrogramLeftCtx, freqDataLeft);
          this.drawSpectrogramColumn(this.spectrogramRightCtx, freqDataRight);
  
          // Actualizar VU meters
          this.drawVUMeter(this.vumeterLeftCtx, this.getRMS(this.analyserLeft), this.vumeterWidth, this.vumeterHeight);
          this.drawVUMeter(this.vumeterRightCtx, this.getRMS(this.analyserRight), this.vumeterWidth, this.vumeterHeight);
  
          this.rafId = requestAnimationFrame(this.updateVisuals);
        } catch (e) {
          // Puede lanzar error si contexto se cierra
        }
      },
      shiftSpectrogram(ctx, width, height) {
        const imageData = ctx.getImageData(1, 0, width - 1, height);
        ctx.putImageData(imageData, 0, 0);
        ctx.fillStyle = "black";
        ctx.fillRect(width - 1, 0, 1, height);
      },
      drawSpectrogramColumn(ctx, freqData) {
        const height = ctx.canvas.height;
        const len = freqData.length;
  
        for (let i = 0; i < len; i++) {
          const value = freqData[i];
          const y = height - Math.floor((i / len) * height);
          const color = `hsl(${(value / 255) * 240}, 100%, 50%)`;
          ctx.fillStyle = color;
          ctx.fillRect(ctx.canvas.width - 1, y, 1, 1);
        }
      },
      getRMS(analyser) {
        const timeData = new Uint8Array(analyser.fftSize);
        analyser.getByteTimeDomainData(timeData);
        let sum = 0;
        for (let i = 0; i < timeData.length; i++) {
          const val = (timeData[i] - 128) / 128;
          sum += val * val;
        }
        return Math.sqrt(sum / timeData.length);
      },
      drawVUMeter(ctx, rms, width, height) {
        ctx.clearRect(0, 0, width, height);
  
        const barWidth = Math.floor(width * rms);
  
        // Background
        ctx.fillStyle = "#333";
        ctx.fillRect(0, 0, width, height);
  
        // Volume bar
        ctx.fillStyle = rms > 0.6 ? "red" : rms > 0.3 ? "yellow" : "lime";
        ctx.fillRect(0, 0, barWidth, height);
      },
      drawFullSpectrogram(buffer, ctx) {
        const width = ctx.canvas.width;
        const height = ctx.canvas.height;
        ctx.clearRect(0, 0, width, height);
  
        // Para simplicidad: mapeamos la amplitud en tiempo al espectrograma, simulando frecuencia
  
        const step = Math.floor(buffer.length / width);
        for (let x = 0; x < width; x++) {
          const start = x * step;
          let sum = 0;
          for (let i = 0; i < step; i++) {
            sum += Math.abs(buffer[start + i]);
          }
          const amplitude = sum / step;
          const colorValue = Math.min(255, amplitude * 255 * 5);
          const color = `rgb(${colorValue},${colorValue},${colorValue})`;
          ctx.fillStyle = color;
          ctx.fillRect(x, 0, 1, height);
        }
      },
    },
    beforeDestroy() {
      if (this.isRecording) this.stop();
    },
  };
  </script>
  
  <style scoped>
  .stereo-recorder {
    max-width: 700px;
    margin: 10px auto;
    font-family: Arial, sans-serif;
  }
  
  .channel {
    margin-top: 20px;
    border: 1px solid #ccc;
    padding: 8px;
    border-radius: 6px;
  }
  
  .waveform {
    width: 100%;
    height: 100px;
    margin-bottom: 10px;
    background: #222;
  }
  
  .spectrogram {
    width: 600px;
    height: 128px;
    background: black;
    margin-bottom: 10px;
    display: block;
  }
  
  .vumeter {
    width: 600px;
    height: 50px;
    background: #222;
    margin-bottom: 10px;
    display: block;
  }
  
  .buttons button {
    margin-right: 10px;
    padding: 6px 12px;
    font-size: 14px;
    cursor: pointer;
  }
  
  select {
    font-size: 14px;
    padding: 3px;
    margin-left: 6px;
  }
  </style>
  