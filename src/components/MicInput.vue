<template>
    <div>
      <h2>🎚️ Visualizador de micrófono</h2>
      <button @click="startMic">Iniciar</button>
      <canvas ref="canvas" width="500" height="100" style="border:1px solid #ccc;"></canvas>
    </div>
  </template>
  
  <script setup>
  import { onMounted, ref } from 'vue'
  
  const canvas = ref(null)
  let animationId
  
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
  
  async function startMic() {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
    const audioCtx = new AudioContext()
    const source = audioCtx.createMediaStreamSource(stream)
    const analyser = audioCtx.createAnalyser()
  
    analyser.fftSize = 256
    const bufferLength = analyser.frequencyBinCount
    const dataArray = new Uint8Array(bufferLength)
  
    source.connect(analyser)
  
    const canvasCtx = canvas.value.getContext('2d')
    draw(analyser, dataArray, canvasCtx)
  }
  </script>
  