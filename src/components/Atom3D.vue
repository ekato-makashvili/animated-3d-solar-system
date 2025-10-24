<template>
  <div ref="containerRef" class="atom-container">
    <!-- Header -->
    <div class="site-header">
      <h1>3D Solar System Explorer</h1>
    </div>

    <!-- Planet buttons -->
    <div class="planet-buttons">
      <button
        v-for="(p, i) in planetsData"
        :key="p.name"
        @click="focusPlanet(p.name)"
      >
        {{ p.name }}
      </button>
    </div>

    <!-- Popup Overlay -->
    <div v-if="activePlanet" class="overlay" @click.self="closeOverlay">
      <div class="overlay-content">
        <img
          class="planet-image"
          :src="activePlanet.image"
          :alt="activePlanet.name"
        />
        <div class="overlay-text">
          <h2>{{ activePlanet.name }}</h2>
          <ul>
            <li v-if="activePlanet.diameter">
              Diameter: {{ activePlanet.diameter }} km
            </li>
            <li v-if="activePlanet.distance">
              Distance from Sun: {{ activePlanet.distance }} million km
            </li>
            <li v-if="activePlanet.temp !== undefined">
              Average Temperature: {{ activePlanet.temp }} °C
            </li>
          </ul>
          <button @click="closeOverlay">Close</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

const containerRef = ref(null);
const activePlanet = ref(null);

const planetsData = [
  {
    name: "Mercury",
    color: 0xb5b5b5,
    size: 4,
    orbitRadius: 50,
    speed: 0.02,
    diameter: 4880,
    distance: 57.9,
    temp: 167,
    image:
      "https://upload.wikimedia.org/wikipedia/commons/4/4a/Mercury_in_true_color.jpg",
  },
  {
    name: "Venus",
    color: 0xeed6a7,
    size: 6,
    orbitRadius: 80,
    speed: 0.015,
    diameter: 12104,
    distance: 108.2,
    temp: 464,
    image:
      "https://upload.wikimedia.org/wikipedia/commons/e/e5/Venus-real_color.jpg",
  },
  {
    name: "Earth",
    color: 0x2f73d2,
    size: 7,
    orbitRadius: 110,
    speed: 0.012,
    diameter: 12742,
    distance: 149.6,
    temp: 15,
    image:
      "https://upload.wikimedia.org/wikipedia/commons/9/97/The_Earth_seen_from_Apollo_17.jpg",
  },
  {
    name: "Mars",
    color: 0xc1440e,
    size: 5,
    orbitRadius: 140,
    speed: 0.01,
    diameter: 6779,
    distance: 227.9,
    temp: -65,
    image:
      "https://upload.wikimedia.org/wikipedia/commons/0/02/OSIRIS_Mars_true_color.jpg",
  },
  {
    name: "Jupiter",
    color: 0xd9c59f,
    size: 12,
    orbitRadius: 180,
    speed: 0.007,
    diameter: 139820,
    distance: 778.5,
    temp: -110,
    image: "https://upload.wikimedia.org/wikipedia/commons/e/e2/Jupiter.jpg",
  },
  {
    name: "Saturn",
    color: 0xf0e68c,
    size: 10,
    orbitRadius: 220,
    speed: 0.006,
    diameter: 116460,
    distance: 1434,
    temp: -140,
    image:
      "https://upload.wikimedia.org/wikipedia/commons/c/c7/Saturn_during_Equinox.jpg",
  },
  {
    name: "Uranus",
    color: 0x7fffd4,
    size: 8,
    orbitRadius: 260,
    speed: 0.005,
    diameter: 50724,
    distance: 2871,
    temp: -195,
    image: "https://upload.wikimedia.org/wikipedia/commons/3/3d/Uranus2.jpg",
  },
  {
    name: "Neptune",
    color: 0x4169e1,
    size: 8,
    orbitRadius: 300,
    speed: 0.004,
    diameter: 49244,
    distance: 4495,
    temp: -200,
    image:
      "https://upload.wikimedia.org/wikipedia/commons/5/56/Neptune_Full.jpg",
  },
  {
    name: "Sun",
    color: 0xffcc33,
    size: 30,
    orbitRadius: 0,
    speed: 0.005,
    diameter: 1391016,
    distance: 0,
    temp: 5505,
    image: "https://upload.wikimedia.org/wikipedia/commons/c/c3/Solar_sys8.jpg",
  },
];

let scene, camera, renderer, controls, frameId;
const planets = [];

onMounted(() => {
  const container = containerRef.value;
  if (!container) return;

  const inclination = THREE.MathUtils.degToRad(20);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(container.clientWidth, container.clientHeight);
  container.appendChild(renderer.domElement);

  scene = new THREE.Scene();
  scene.fog = new THREE.FogExp2(0x0b0c1c, 0.0015);

  // Camera – მოასწორეთ ასე, რომ ყველა პლანეტა ჩანდეს
  camera = new THREE.PerspectiveCamera(
    60,
    container.clientWidth / container.clientHeight,
    0.1,
    3000
  );
  camera.position.set(0, 100, 435);
  camera.lookAt(0, 0, 0);

  // Lights
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));
  const sunLight = new THREE.PointLight(0xffffff, 2.5, 3000);
  sunLight.position.set(0, 0, 0);
  scene.add(sunLight);

  // Orbits
  planetsData.forEach((p) => {
    if (p.orbitRadius > 0) {
      const orbitGeom = new THREE.RingGeometry(
        p.orbitRadius - 0.05,
        p.orbitRadius + 0.05,
        128
      );
      const orbitMat = new THREE.MeshBasicMaterial({
        color: 0xffffff,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.2,
      });
      const orbit = new THREE.Mesh(orbitGeom, orbitMat);
      orbit.rotation.x = Math.PI / 2;
      orbit.rotation.z = inclination;
      scene.add(orbit);
    }
  });

  // Planets
  // inside onMounted, after creating each planet
  planetsData.forEach((data) => {
    // Planet mesh
    const geo = new THREE.SphereGeometry(data.size, 64, 64);
    const mat =
      data.name === "Sun"
        ? new THREE.MeshStandardMaterial({
            color: data.color,
            emissive: data.color,
            emissiveIntensity: 2,
          })
        : new THREE.MeshStandardMaterial({
            color: data.color,
            roughness: 0.3,
            metalness: 0.1,
          });
    const mesh = new THREE.Mesh(geo, mat);
    const angle = Math.random() * Math.PI * 2;
    const yOffset = (Math.random() - 0.5) * 3;
    mesh.position.set(
      Math.cos(angle) * data.orbitRadius,
      yOffset,
      Math.sin(angle) * data.orbitRadius
    );
    scene.add(mesh);

    // Planet label as Sprite
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    ctx.font = "28px Arial";
    ctx.fillStyle = "white";
    ctx.textAlign = "center";
    ctx.fillText(data.name, canvas.width / 2, 28);
    const texture = new THREE.CanvasTexture(canvas);
    const labelMat = new THREE.SpriteMaterial({
      map: texture,
      transparent: true,
    });
    const label = new THREE.Sprite(labelMat);
    label.scale.set(data.size * 4, data.size * 2, 1); // scale relative to planet size
    label.position.set(0, data.size + 2, 0); // above planet
    mesh.add(label); // attach to planet mesh

    planets.push({ mesh, label, ...data, angle });
  });

  // Stars
  const starCount = 2500;
  const starGeo = new THREE.BufferGeometry();
  const positions = new Float32Array(starCount * 3);
  for (let i = 0; i < starCount * 3; i++)
    positions[i] = (Math.random() - 0.5) * 4000;
  starGeo.setAttribute("position", new THREE.BufferAttribute(positions, 3));
  const starMat = new THREE.PointsMaterial({
    color: 0xffffff,
    size: 1.2,
    transparent: true,
    opacity: 0.85,
  });
  scene.add(new THREE.Points(starGeo, starMat));

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.06;
  controls.enablePan = true;
  controls.autoRotate = false;
  controls.minDistance = 60;
  controls.maxDistance = 1200;

  // Animation
  const clock = new THREE.Clock();
  const animate = () => {
    frameId = requestAnimationFrame(animate);
    const t = clock.getElapsedTime();
    planets.forEach((p) => {
      if (p.name !== "Sun") {
        p.angle += p.speed;
        p.mesh.position.x = Math.cos(p.angle) * p.orbitRadius;
        p.mesh.position.z = Math.sin(p.angle) * p.orbitRadius;
        p.mesh.position.y = Math.sin(p.angle * 1.5) * 2;
      } else {
        p.mesh.rotation.y += 0.005;
      }
      p.mesh.rotation.y += 0.01;
    });
    controls.update();
    renderer.render(scene, camera);
  };
  animate();

  const onResize = () => {
    camera.aspect = container.clientWidth / container.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.clientWidth, container.clientHeight);
  };
  window.addEventListener("resize", onResize);

  onBeforeUnmount(() => {
    cancelAnimationFrame(frameId);
    window.removeEventListener("resize", onResize);
    controls.dispose();
    renderer.dispose();
    scene.traverse((obj) => {
      if (obj.geometry) obj.geometry.dispose();
      if (obj.material) {
        if (Array.isArray(obj.material))
          obj.material.forEach((m) => m.dispose());
        else obj.material.dispose();
      }
    });
    if (container.firstChild) container.removeChild(container.firstChild);
  });

  // Planet buttons glow animation
  const buttons = container.querySelectorAll(".planet-buttons button");
  buttons.forEach((btn, i) =>
    setTimeout(() => btn.classList.add("show"), i * 150)
  );
});

// Popup functions
function focusPlanet(name) {
  const planet = planets.find((p) => p.name === name);
  if (!planet) return;
  activePlanet.value = {
    name: planet.name,
    diameter: planet.diameter,
    distance: planet.distance,
    temp: planet.temp,
    image: planet.image,
  };

  const targetPos = new THREE.Vector3();
  planet.mesh.getWorldPosition(targetPos);
  const start = camera.position.clone();
  const end = targetPos.clone().add(new THREE.Vector3(0, 10, 20));
  let progress = 0;
  function moveCamera() {
    if (progress < 1) {
      progress += 0.02;
      camera.position.lerpVectors(start, end, progress);
      camera.lookAt(targetPos);
      requestAnimationFrame(moveCamera);
    }
  }
  moveCamera();
}

function closeOverlay() {
  activePlanet.value = null;
}
</script>

<style scoped>
.atom-container {
  width: 100%;
  height: 100vh;
  background: radial-gradient(circle at center, #0b0c1c 0%, #1a1a2e 100%);
  position: relative;
  overflow: hidden;
  font-family: "Poppins", sans-serif;
}
.site-header {
  position: absolute;
  top: 50px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  color: white;
  z-index: 10;
  max-width: 90%;
}
.site-header h1 {
  white-space: nowrap;
  font-size: 32px;
  margin: 0;
}
.planet-buttons {
  position: absolute;
  top: 150px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  flex-direction: row;
  gap: 12px;
  z-index: 10;
}
.planet-buttons button {
  padding: 8px 16px;
  background: linear-gradient(
    145deg,
    rgba(255, 255, 255, 0.08),
    rgba(255, 255, 255, 0.15)
  );
  border: none;
  border-radius: 8px;
  color: white;
  cursor: pointer;
  font-weight: 600;
  transition: transform 0.3s, background 0.3s, opacity 0.6s;
  opacity: 0;
  transform: translateY(20px);
}
.planet-buttons button.show {
  opacity: 1;
  transform: translateY(0);
}
.planet-buttons button:hover {
  background: linear-gradient(
    145deg,
    rgba(255, 255, 255, 0.15),
    rgba(255, 255, 255, 0.25)
  );
  transform: scale(1.05);
}
.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.85);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}
.overlay-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #111;
  border-radius: 12px;
  padding: 24px;
  width: 360px;
  max-width: 90%;
  color: white;
}
.overlay-content img.planet-image {
  width: 100%;
  border-radius: 8px;
  margin-bottom: 16px;
}
.overlay-text {
  width: 100%;
  text-align: left;
}
.overlay-text h2 {
  margin-bottom: 8px;
  font-size: 22px;
}
.overlay-text ul {
  padding-left: 20px;
  list-style: disc;
}
.overlay-text button {
  margin-top: 12px;
  padding: 6px 12px;
  background: #222;
  color: white;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
}
.overlay-text button:hover {
  background: #333;
}

@media (max-width: 768px) {
  .site-header h1 {
    font-size: 28px;
  }
  .planet-buttons {
    flex-direction: column;
    top: 120px;
    gap: 10px;
  }
  .planet-buttons button {
    width: 120px;
    padding: 10px 16px;
    font-size: 14px;
  }
}
@media (max-width: 480px) {
  .site-header h1 {
    font-size: 20px;
  }
  .planet-buttons {
    gap: 8px;
  }
  .planet-buttons button {
    width: 100px;
    padding: 8px 12px;
    font-size: 13px;
  }
}
</style>
