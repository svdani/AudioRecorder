<template>
    <div>
      <h2>🎵 Reproductor básico</h2>
      <button @click="playSound">Reproducir sonido</button>
    </div>
  </template>
  
  <script setup>
  function playSound() {
    const context = new AudioContext()
    const osc = context.createOscillator()
  
    osc.type = 'sine' // forma de onda: sine, square, triangle, sawtooth
    osc.frequency.setValueAtTime(440, context.currentTime) // frecuencia en Hz (A4)
  
    osc.connect(context.destination)
    osc.start()
    osc.stop(context.currentTime + 1) // suena durante 1 segundo
  }
  </script>
  