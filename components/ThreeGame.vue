<template>

    <v-container class="d-flex justify-center align-center fill-height">
     
      <div ref="threeCanvas" class="three-canvas"></div>
    </v-container>
  </template>
  
  <script setup>
  import { onMounted, ref } from 'vue'; 
  import * as THREE from 'three'; 
  
  // Cria uma referência para a div que será usada como o canvas onde o Three.js irá renderizar
  const threeCanvas = ref(null);
  

  onMounted(() => {
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  const renderer = new THREE.WebGLRenderer();

  // Alterar a cor de fundo do canvas (neste exemplo, cor preta)
  renderer.setClearColor(0x000000); 
  
  // Definir o tamanho do canvas (neste exemplo, 800x600 pixels)
  renderer.setSize(1000, 600); 
  
  // Adicionar o canvas à div referenciada
  threeCanvas.value.appendChild(renderer.domElement);

  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
  const cube = new THREE.Mesh(geometry, material);
  scene.add(cube);

  camera.position.z = 5;

  const animate = () => {
    requestAnimationFrame(animate);
    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;
    renderer.render(scene, camera);
  };

  animate();
  
  window.addEventListener('resize', () => {
    const width = window.innerWidth;
    const height = window.innerHeight;
    renderer.setSize(width, height);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
  });
});

  </script>
  
  <style scoped>
  /* Estiliza a div onde o Three.js renderiza, para preencher a tela completamente */
  .three-canvas {
    width: 100%;
    height: 100vh;
  }
  </style>
  