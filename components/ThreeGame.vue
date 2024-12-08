<template>
  <div ref="container">
    <div class="hud">
      <v-row class="bg-transparent mr-auto ml-auto" style="width: max-content; height: auto" align="center">
        <v-select
          density="compact"
          v-model="difficulty"
          :items="difficultyOptions"
            @update:modelValue="updateDifficulty"
          class="text-black font-weight-bold border-lg border-black border-opacity-100 bg-white d-flex align-center justify-center"
          style="width: auto; height: auto" />
        <h5 class="text-black font-weight-bold px-4 border-lg border-black border-opacity-100 bg-white">
          <v-icon color="red" size="x-large">mdi-heart</v-icon> {{ gameState.lives }}
        </h5>

        <h5 class="text-black font-weight-bold border-lg px-4 border-black border-opacity-100 bg-white">
          <v-icon color="yellow" size="x-large">mdi-star</v-icon> {{ gameState.points }}
        </h5>
      </v-row>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, reactive } from 'vue';
import * as THREE from 'three';
import Swal from 'sweetalert2';

// Dificuldade e opções de dificuldade
const difficulty = ref('Fácil');
const difficultyOptions = ref(['Fácil', 'Médio', 'Difícil']);

const container = ref(null);
let scene, camera, renderer, animationFrameId;
let backgroundPlane;
const obstacleSpeed = ref( 0.02);
const baseObstacleCreationInterval = ref(75); // Intervalo base em milissegundos

const updateDifficulty = () => {
  console.log("Iniciou")
  switch (difficulty.value) {
    case 'Fácil':
      baseObstacleCreationInterval.value = 75;
      obstacleSpeed.value = 0.02;
      break;
    case 'Médio':
      baseObstacleCreationInterval.value = 5;
      obstacleSpeed.value = 0.05;
      break;
    case 'Difícil':
      baseObstacleCreationInterval.value = 10;
      obstacleSpeed.value = 0.08;
      break;
  }
  console.log(`Dificuldade: ${difficulty.value}, Velocidade: ${obstacleSpeed.value}, Intervalo: ${obstacleCreationInterval()}`);
};


// Objeto reativo para o estado do jogo
const gameState = reactive({
  lives: 3, // Vidas iniciais
  points: 0 // Pontuação inicial
});

// Variáveis para o personagem
let character,
  isJumping = false,
  jumpVelocity = 0,
  jumpHeight = 0.8,
  characterSpeed = 0.8;
let obstacles = []; // Array para armazenar os obstáculos
let obstacleInterval; // Intervalo para criar obstáculos


// Calcula o intervalo de criação baseado na velocidade do obstáculo
const obstacleCreationInterval = () => baseObstacleCreationInterval.value / obstacleSpeed.value;

onMounted(() => {
  // Configuração da cena e da câmera
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x1fe3fb);

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 5;

  renderer = new THREE.WebGLRenderer({ alpha: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  container.value.appendChild(renderer.domElement);

  // Adicionando o plano de fundo
  const textureLoader = new THREE.TextureLoader();
  const backgroundTexture = textureLoader.load('/asfalt_road.jpg', () => {
    const planeGeometry = new THREE.PlaneGeometry(1, 1);
    const planeMaterial = new THREE.MeshBasicMaterial({
      map: backgroundTexture,
      transparent: true,
      side: THREE.DoubleSide
    });

    backgroundPlane = new THREE.Mesh(planeGeometry, planeMaterial);
    backgroundPlane.position.z = -10;
    scene.add(backgroundPlane);

    backgroundPlane.material.map.wrapS = backgroundPlane.material.map.wrapT = THREE.RepeatWrapping;
    backgroundPlane.material.map.repeat.set(1, 10); // Configura para repetir verticalmente
    resizePlane();
  });

  // Adicionando o personagem
  const texture2DLoader = new THREE.TextureLoader();
  const texture2D = texture2DLoader.load('../pou.png');
  const planeGeometry2D = new THREE.PlaneGeometry(1.2, 1.2);
  const planeMaterial2D = new THREE.MeshBasicMaterial({ map: texture2D, transparent: true });
  character = new THREE.Mesh(planeGeometry2D, planeMaterial2D);
  character.position.set(0, -5, -3);
  scene.add(character);

  // Função para criar um obstáculo
  const createObstacle = () => {
    const obstacleGeometry = new THREE.BoxGeometry(0.5, 0.5, 0.5);
    const obstacleMaterial = new THREE.MeshBasicMaterial({ color: 0xff0000 });
    const obstacle = new THREE.Mesh(obstacleGeometry, obstacleMaterial);
    obstacle.position.set((Math.random() - 0.5) * 5, Math.random() * 3 + 5, -3);
    scene.add(obstacle);
    obstacles.push(obstacle);
  };

  const originalColor = character.material.color.clone();

  const animate = () => {
    animationFrameId = requestAnimationFrame(animate);

    // Atualiza o plano de fundo para o movimento contínuo
    if (backgroundPlane) {
      backgroundPlane.material.map.offset.y += 0.01;
      if (backgroundPlane.material.map.offset.y >= 1) {
        backgroundPlane.material.map.offset.y = 0;
      }
    }

    // Variável para evitar múltiplas perdas de vidas na mesma atualização
    let lifeLost = false;

    // Atualiza a posição dos obstáculos
    obstacles.forEach((obstacle, index) => {
      obstacle.position.y -= obstacleSpeed.value;

      // Detecta colisão entre obstáculo e personagem
      if (character.position.distanceTo(obstacle.position) < 1 && !lifeLost) {
        lifeLost = true; // Marca que uma vida foi perdida

        // Verifica se ainda tem vidas
        if (gameState.lives > 0) {
          gameState.lives -= 1; // Perde uma vida
        }

        // Muda a cor do personagem para indicar a colisão
        character.material.color.set(0xff0000);
        setTimeout(() => {
          character.material.color.copy(originalColor); // Restaura a cor original após 1 segundo
        }, 1000);

        scene.remove(obstacle); // Remove o obstáculo colidido
        obstacles.splice(index, 1);

        // Verifica se as vidas acabaram
        if (gameState.lives <= 0) {
          // Pausa a animação e a criação de obstáculos
          cancelAnimationFrame(animationFrameId);
          clearInterval(obstacleInterval);

          // Exibe o alerta
          Swal.fire({
            title: 'Acabaram suas vidas!',
            text: 'Filha, comece novamente.',
            icon: 'warning',
            confirmButtonText: 'Jogar novamente'
          }).then(() => {
            // Reinicia as vidas e pontos
            gameState.lives = 3;
            gameState.points = 0;

            // Retoma a animação e a criação de obstáculos
            animate();
            obstacleInterval = setInterval(createObstacle, obstacleCreationInterval());
          });
          return; // Sai da função para evitar continuar a execução após reiniciar
        }
      }

      // Remove obstáculos que saem da tela
      if (obstacle.position.y < -5) {
        scene.remove(obstacle);
        obstacles.splice(index, 1);
        gameState.points += 10; // Ganha pontos por evitar obstáculos

        // Verifica se a pontuação é um múltiplo de 250
        if (gameState.points % 250 === 0) {
          gameState.lives += 1; // Adiciona uma vida
        }
      }
    });

    // Atualiza o pulo do personagem
    if (isJumping) {
      character.position.y += jumpVelocity;
      jumpVelocity -= 0.05;
      if (character.position.y <= -5) {
        character.position.y = -5;
        isJumping = false;
        jumpVelocity = 0;
      }
    }

    renderer.render(scene, camera);
  };

  // Iniciar a criação de obstáculos
  obstacleInterval = setInterval(createObstacle, obstacleCreationInterval());

  // Inicia a animação
  animate();

  // Movimento lateral e pulo do personagem
  container.value.addEventListener('click', (event) => {
    // Converte as coordenadas do mouse para a posição do mundo 3D
    const rect = container.value.getBoundingClientRect();
    const mouseX = ((event.clientX - rect.left) / rect.width) * 2 - 1; // Normaliza para [-1, 1]

    // Atualiza a posição x do personagem, garantindo que ele não ultrapasse os limites
    character.position.x = Math.max(-2.5, Math.min(2.5, mouseX * 5)); // Multiplica por 5 para ajustar o movimento
  });

  const characterWidth = 1.2; // Largura do personagem

  // Durante a movimentação lateral
  window.addEventListener('keydown', (event) => {
    // Calcule o limite direito e esquerdo
    const rightLimit = window.innerWidth / 2 / 100 - characterWidth / 2;
    const leftLimit = -(window.innerWidth / 2 / 100) + characterWidth / 2;

    if (event.key === 'ArrowRight') {
      // Movimento para a direita
      character.position.x = Math.min(rightLimit + characterWidth / 2, character.position.x + characterSpeed);
    } else if (event.key === 'ArrowLeft') {
      // Movimento para a esquerda
      character.position.x = Math.max(leftLimit - characterWidth / 2, character.position.x - characterSpeed);
    }
  });

  // Ajuste de tamanho da tela
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    if (backgroundPlane) resizePlane();
  });
});

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId);
  clearInterval(obstacleInterval);
  renderer.dispose();
});

function resizePlane() {
  const planeHeight =
    2 * Math.tan((camera.fov * Math.PI) / 360) * Math.abs(camera.position.z - backgroundPlane.position.z);
  const planeWidth = planeHeight * camera.aspect;
  backgroundPlane.scale.set(planeWidth, planeHeight, 1);
}
</script>

<style scoped>
div {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  margin: 0;
  box-sizing: border-box; /* Inclui padding e borda nas dimensões totais */
}

.hud {
  position: absolute;
  top: 10px;
  left: 10px;
  font-size: 1.5em;
  color: white;
  font-family: Arial, sans-serif;
  width: calc(100% - 20px); /* Ajusta a largura para garantir que não exceda os limites da viewport */
  overflow: hidden; /* Para evitar rolagem desnecessária */
}
</style>
