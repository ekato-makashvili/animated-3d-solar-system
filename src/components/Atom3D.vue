<template>
  <div ref="containerRef" class="atom-container">
    <!-- ვებსაიტის დასახელება / აღწერა -->
    <div class="site-header">
      <h1>3D Solar System Explorer</h1>
      <p>Explore the planets of our solar system in 3D. Click on any planet to learn its main properties and view it up close.</p>
    </div>

    <!-- პლანეტების ღილაკები -->
    <div class="planet-buttons">
      <button v-for="p in planetsData" :key="p.name" @click.stop="focusPlanet(p.name)">
        {{ p.name }}
      </button>
    </div>

    <!-- Overlay პლანეტის ინფორმაციით -->
    <div v-if="activePlanet" class="overlay" @click="overlayClick">
      <div class="overlay-content" ref="overlayContent">
        <img class="planet-image" :src="activePlanet.image" :alt="activePlanet.name"/>
        <div class="overlay-text">
          <h2>{{ activePlanet.name }}</h2>
          <ul>
            <li v-if="activePlanet.diameter">Diameter: {{ activePlanet.diameter }} km</li>
            <li v-if="activePlanet.distance">Distance from Sun: {{ activePlanet.distance }} million km</li>
            <li v-if="activePlanet.temp !== undefined">Average Temperature: {{ activePlanet.temp }} °C</li>
          </ul>
          <button @click="closeOverlay">Close</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const containerRef = ref(null)
const activePlanet = ref(null)
const overlayContent = ref(null)

const planetsData = [
  { name: 'Sun', color: 0xffdd55, size: 40, orbitRadius: 0, speed: 0.005, diameter: 1391016, distance: 0, temp: 5505, image: 'https://upload.wikimedia.org/wikipedia/commons/c/c3/Solar_sys8.jpg' },
  { name: 'Mercury', color: 0xb5b5b5, size: 4, orbitRadius: 50, speed: 0.02, diameter: 4880, distance: 57.9, temp: 167, image: 'https://upload.wikimedia.org/wikipedia/commons/4/4a/Mercury_in_true_color.jpg' },
  { name: 'Venus', color: 0xffe680, size: 6, orbitRadius: 80, speed: 0.015, diameter: 12104, distance: 108.2, temp: 464, image: 'https://upload.wikimedia.org/wikipedia/commons/e/e5/Venus-real_color.jpg' },
  { name: 'Earth', color: 0x2f73d2, size: 7, orbitRadius: 110, speed: 0.012, diameter: 12742, distance: 149.6, temp: 15, image: 'https://upload.wikimedia.org/wikipedia/commons/9/97/The_Earth_seen_from_Apollo_17.jpg' },
  { name: 'Mars', color: 0xff4500, size: 5, orbitRadius: 140, speed: 0.01, diameter: 6779, distance: 227.9, temp: -65, image: 'https://upload.wikimedia.org/wikipedia/commons/0/02/OSIRIS_Mars_true_color.jpg' },
  { name: 'Jupiter', color: 0xffd2a6, size: 12, orbitRadius: 180, speed: 0.007, diameter: 139820, distance: 778.5, temp: -110, image: 'https://upload.wikimedia.org/wikipedia/commons/e/e2/Jupiter.jpg' },
  { name: 'Saturn', color: 0xffff99, size: 10, orbitRadius: 220, speed: 0.006, diameter: 116460, distance: 1434, temp: -140, image: 'https://upload.wikimedia.org/wikipedia/commons/c/c7/Saturn_during_Equinox.jpg' },
  { name: 'Uranus', color: 0x7fffd4, size: 8, orbitRadius: 260, speed: 0.005, diameter: 50724, distance: 2871, temp: -195, image: 'https://upload.wikimedia.org/wikipedia/commons/3/3d/Uranus2.jpg' },
  { name: 'Neptune', color: 0x4169e1, size: 8, orbitRadius: 300, speed: 0.004, diameter: 49244, distance: 4495, temp: -200, image: 'https://upload.wikimedia.org/wikipedia/commons/5/56/Neptune_Full.jpg' }
]

let scene, camera, renderer, controls, frameId
const planets = []

onMounted(()=>{
  const container = containerRef.value
  if(!container) return
  const inclination = THREE.MathUtils.degToRad(25)

  renderer = new THREE.WebGLRenderer({ antialias:true, alpha:true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2))
  renderer.setSize(container.clientWidth, container.clientHeight)
  container.appendChild(renderer.domElement)

  scene = new THREE.Scene()
  scene.fog = new THREE.FogExp2(0x0b0c1c,0.0015)

  camera = new THREE.PerspectiveCamera(60, container.clientWidth/container.clientHeight, 0.1, 5000)
  camera.position.set(0,100,420)
  camera.lookAt(0,0,0)

  scene.add(new THREE.AmbientLight(0xffffff,0.3))
  const sunLight = new THREE.PointLight(0xffffff,4,3000)
  sunLight.position.set(0,0,0)
  scene.add(sunLight)

  // Planets
  planetsData.forEach(data=>{
    const geo = new THREE.SphereGeometry(data.size,64,64)
    const mat = new THREE.MeshStandardMaterial({
      color:data.color,
      roughness:0.2,
      metalness:0.1,
      emissive:data.color,
      emissiveIntensity:data.name==='Sun'?1.5:0.8
    })
    const mesh = new THREE.Mesh(geo, mat)
    const angle = Math.random()*Math.PI*2
    mesh.position.set(Math.cos(angle)*data.orbitRadius,0,Math.sin(angle)*data.orbitRadius)
    mesh.rotation.x = inclination
    scene.add(mesh)
    planets.push({...data, mesh, angle})
  })

  // Orbits
  planetsData.forEach(p=>{
    if(p.orbitRadius>0){
      const orbitGeom = new THREE.RingGeometry(p.orbitRadius-0.05,p.orbitRadius+0.05,128)
      const orbitMat = new THREE.MeshBasicMaterial({ color:0xffffff, side:THREE.DoubleSide, transparent:true, opacity:0.15 })
      const orbit = new THREE.Mesh(orbitGeom, orbitMat)
      orbit.rotation.x = Math.PI/2
      orbit.rotation.z = inclination
      scene.add(orbit)
    }
  })

  // Stars
  const starCount=2500
  const starGeo=new THREE.BufferGeometry()
  const starPos=new Float32Array(starCount*3)
  for(let i=0;i<starCount*3;i++) starPos[i]=(Math.random()-0.5)*4000
  starGeo.setAttribute('position',new THREE.BufferAttribute(starPos,3))
  const starMat=new THREE.PointsMaterial({color:0xffffff,size:1.5,transparent:true,opacity:0.9})
  scene.add(new THREE.Points(starGeo,starMat))

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping=true
  controls.dampingFactor=0.05
  controls.enablePan=true
  controls.autoRotate=false
  controls.minDistance=60
  controls.maxDistance=1500

  const clock=new THREE.Clock()
  const animate=()=>{
    frameId=requestAnimationFrame(animate)
    planets.forEach(p=>{
      if(p.name!=='Sun'){
        p.angle+=p.speed
        p.mesh.position.x=Math.cos(p.angle)*p.orbitRadius
        p.mesh.position.z=Math.sin(p.angle)*p.orbitRadius
        p.mesh.position.y=Math.sin(p.angle*1.5)*1.5
      }
      p.mesh.rotation.y+=0.01
      p.mesh.rotation.x=inclination
    })
    controls.update()
    renderer.render(scene,camera)
  }
  animate()

  const onResize=()=>{
    camera.aspect=container.clientWidth/container.clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(container.clientWidth,container.clientHeight)
  }
  window.addEventListener('resize',onResize)

  onBeforeUnmount(()=>{
    cancelAnimationFrame(frameId)
    window.removeEventListener('resize',onResize)
    controls.dispose()
    renderer.dispose()
    scene.traverse(obj=>{
      if(obj.geometry) obj.geometry.dispose()
      if(obj.material) Array.isArray(obj.material)?obj.material.forEach(m=>m.dispose()):obj.material.dispose()
    })
    if(container.firstChild) container.removeChild(container.firstChild)
  })
})

// Popup functions
function focusPlanet(name){
  const planet=planets.find(p=>p.name===name)
  if(!planet) return
  activePlanet.value={
    name:planet.name,
    diameter:planet.diameter,
    distance:planet.distance,
    temp:planet.temp,
    image:planet.image
  }

  const targetPos=new THREE.Vector3()
  planet.mesh.getWorldPosition(targetPos)
  const start=camera.position.clone()
  const end=targetPos.clone().add(new THREE.Vector3(0,10,20))
  let progress=0
  function moveCamera(){
    if(progress<1){
      progress+=0.02
      camera.position.lerpVectors(start,end,progress)
      camera.lookAt(targetPos)
      requestAnimationFrame(moveCamera)
    }
  }
  moveCamera()
}

function closeOverlay(){
  activePlanet.value=null
}

function overlayClick(event){
  if(event.target.classList.contains('overlay')){
    closeOverlay()
  }
}
</script>

<style scoped>
.atom-container{width:100%;height:100vh;background:radial-gradient(circle at center,#0b0c1c 0%,#1a1a2e 100%);position:relative;overflow:hidden;font-family:'Poppins',sans-serif}
.site-header{position:absolute;top:20px;left:50%;transform:translateX(-50%);text-align:center;color:white;z-index:10}
.planet-buttons{position:absolute;top:120px;left:20px;display:flex;flex-direction:column;gap:10px;z-index:10}
.planet-buttons button{padding:8px 16px;background:linear-gradient(145deg, rgba(255,255,255,0.08), rgba(255,255,255,0.15));border:none;border-radius:8px;color:white;cursor:pointer;font-weight:600;transition:0.3s}
.planet-buttons button:hover{background:linear-gradient(145deg, rgba(255,255,255,0.15), rgba(255,255,255,0.25));transform:scale(1.05)}
.overlay{position:absolute;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.85);display:flex;justify-content:center;align-items:center;z-index:100}
.overlay-content{display:flex;flex-direction:column;align-items:center;background:#111;border-radius:12px;padding:24px;width:360px;max-width:90%;color:white}
.overlay-content img.planet-image{width:100%;border-radius:8px;margin-bottom:16px}
.overlay-text{width:100%;text-align:left}
.overlay-text h2{margin-bottom:8px;font-size:22px}
.overlay-text ul{padding-left:20px;list-style:disc}
.overlay-text button{margin-top:12px;padding:6px 12px;background:#222;color:white;border-radius:6px;cursor:pointer;font-weight:600}
.overlay-text button:hover{background:#333}
@media(max-width:768px){.planet-buttons{top:10px;left:10px}.overlay-content{width:90%;padding:16px}}
</style>
