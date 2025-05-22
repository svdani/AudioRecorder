<template>
    <div>
      <h2>🎙️ Grabadora de Micrófono con Ecualizador</h2>
  
      <button @click="startRecording" :disabled="isRecording">Grabar</button>
      <button @click="stopRecording" :disabled="!isRecording">Detener</button>
      <button @click="playRecording" :disabled="!audioUrl">Reproducir</button>
      <a v-if="audioUrl" :href="audioUrl" download="grabacion.wav">Descargar grabación</a>
  
      <p v-if="isRecording">🎧 Grabando...</p>
      <p v-if="audioUrl">✔ Grabación lista</p>
  
      <canvas ref="canvas" width="500" height="100" style="border:1px solid #ccc;"></canvas>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, onBeforeUnmount } from 'vue'
  
  const isRecording = ref(false)
  const audioUrl = ref(null)
  const canvas = ref(null)
  let mediaRecorder
  let audioChunks = []
  let analyser
  let audioContext
  let source
  let animationId
  
  // Inicia la grabación
  async function startRecording() {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    audioContext = new AudioContext()
    source = audioContext.createMediaStreamSource(stream)
    analyser = audioContext.createAnalyser()
  
    analyser.fftSize = 256
    const bufferLength = analyser.frequencyBinCount
    const dataArray = new Uint8Array(bufferLength)
  
    source.connect(analyser)
    
    // Configurar el MediaRecorder
    mediaRecorder = new MediaRecorder(stream)
    mediaRecorder.ondataavailable = (event) => {
      audioChunks.push(event.data)
    }
  
    mediaRecorder.onstop = () => {
      const audioBlob = new Blob(audioChunks, { type: 'audio/wav' })
      audioUrl.value = URL.createObjectURL(audioBlob)
      audioChunks = []
    }
  
    mediaRecorder.start()
    isRecording.value = true
  
    // Visualización del ecualizador
    const canvasCtx = canvas.value.getContext('2d')
    draw(analyser, dataArray, canvasCtx)
  }
  
  // Detiene la grabación
  function stopRecording() {
    mediaRecorder.stop()
    isRecording.value = false
  }
  
  // Reproduce la grabación
  function playRecording() {
    const audio = new Audio(audioUrl.value)
    audio.play()
  }
  
  // Función de dibujo para el ecualizador
  function draw(analyser, dataArray, canvasCtx) {
    const WIDTH = canvasCtx.canvas.width
    const HEIGHT = canvasCtx.canvas.height
  
    function render() {
      animationId = requestAnimationFrame(render)
  
      analyser.getByteFrequencyData(dataArray)
  
      canvasCtx.fillStyle = 'black'
      canvasCtx.fillRect(0, 0, WIDTH, HEIGHT)
  
      const barWidth = (WIDTH / dataArray.length) * 2.5
      let barHeight
      let x = 0
  
      for (let i = 0; i < dataArray.length; i++) {
        barHeight = dataArray[i]
        canvasCtx.fillStyle = `rgb(${barHeight + 100},50,50)`
        canvasCtx.fillRect(x, HEIGHT - barHeight / 2, barWidth, barHeight / 2)
        x += barWidth + 1
      }
    }
  
    render()
  }
  
  // Función para descargar la grabación
  function downloadRecording() {
    const downloadLink = document.createElement('a')
    downloadLink.href = audioUrl.value
    downloadLink.download = 'grabacion.wav'
    downloadLink.click()
  }
  
  // Limpia la animación cuando el componente se desmonte
  onBeforeUnmount(() => {
    if (animationId) cancelAnimationFrame(animationId)
  })
  </script>
  