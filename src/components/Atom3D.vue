<template>
  <div ref="containerRef" class="atom-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

const containerRef = ref(null)

const skills = [
  { label: 'Mercury', color: 0xb1b1b1, size: 4 },     // Mercury: gray
  { label: 'Venus', color: 0xf5deb3, size: 6 },       // Venus: pale yellow-beige
  { label: 'Earth', color: 0x2a73cc, size: 7 },       // Earth: blue
  { label: 'Mars', color: 0xb22222, size: 5 },        // Mars: red
  { label: 'Jupiter', color: 0xd9c59f, size: 12 },    // Jupiter: beige with bands
  { label: 'Saturn', color: 0xf0e68c, size: 10 },     // Saturn: pale yellow
  { label: 'Uranus', color: 0x7fffd4, size: 8 },      // Uranus: light cyan
  { label: 'Neptune', color: 0x4169e1, size: 8 }      // Neptune: deep blue
]
let renderer, scene, camera, controls, frameId
const planets = []

onMounted(() => {
  const container = containerRef.value
  if (!container) return

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(container.clientWidth, container.clientHeight)
  container.appendChild(renderer.domElement)

  // Scene & Camera
  scene = new THREE.Scene()
  scene.fog = new THREE.FogExp2(0x0b0c1c, 0.002)

camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 3000)
camera.position.set(0, 100, 400) // move camera further back


  // Lights
  scene.add(new THREE.AmbientLight(0xffffff, 0.4))
  const point = new THREE.PointLight(0xfff2cc, 1.5, 1000)
  point.position.set(0, 0, 0)
  scene.add(point)
  const rim = new THREE.DirectionalLight(0xffffff, 0.2)
  rim.position.set(100, 100, 50)
  scene.add(rim)

  // Sun
  const sunGeo = new THREE.SphereGeometry(20, 64, 64)
  const sunMat = new THREE.MeshStandardMaterial({
    color: 0xffd27f,
    emissive: 0xffa500,
    emissiveIntensity: 1.2,
    roughness: 0.2,
    metalness: 0.6
  })
  const sun = new THREE.Mesh(sunGeo, sunMat)
  sun.position.set(0,0,0)
  sun.userData = { rotationSpeed: 0.004 }
  scene.add(sun)

  // Sun label
  const canvasProf = document.createElement('canvas')
  canvasProf.width = 512
  canvasProf.height = 128
  const ctxProf = canvasProf.getContext('2d')
  ctxProf.fillStyle='white'
  ctxProf.font='bold 60px Poppins, sans-serif'
  ctxProf.textAlign='center'
  ctxProf.shadowColor = 'rgba(0,0,0,0.9)'
  ctxProf.shadowBlur = 8
  ctxProf.fillText('Sun', 256, 64)
  const profSprite = new THREE.Sprite(new THREE.SpriteMaterial({
    map: new THREE.CanvasTexture(canvasProf),
    transparent:true
  }))
  profSprite.scale.set(80,20,1)
  profSprite.position.set(0,25, 0)
  sun.add(profSprite)

  // Orbits and Planets
  const baseRadius = 50
  skills.forEach((s,i)=>{
    const radius = baseRadius + i*35
    const speed = 0.02 + i*0.005
    const rotationSpeed = 0.01 + i*0.002

    const planetRadius = s.size
    const geo = new THREE.SphereGeometry(planetRadius,32,32)
    const mat = new THREE.MeshStandardMaterial({
      color: s.color,
      emissive: s.color,
      emissiveIntensity: 0.7,
      roughness:0.2,
      metalness:0.8
    })
    const planet = new THREE.Mesh(geo, mat)
    planet.userData = { radius, speed, rotationSpeed }

    // Orbit
    const segments = 128
    const points = []
    for(let j=0;j<=segments;j++){
      const theta = (j/segments)*Math.PI*2
      points.push(new THREE.Vector3(Math.cos(theta)*radius,0,Math.sin(theta)*radius))
    }
    const orbitGeom = new THREE.BufferGeometry().setFromPoints(points)
    const orbitMat = new THREE.LineBasicMaterial({ color:0xffffff, transparent:true, opacity:0.25 })
    const orbitLine = new THREE.Line(orbitGeom, orbitMat)
    scene.add(orbitLine)

    // Skill label
    const canvas = document.createElement('canvas')
    canvas.width = 256
    canvas.height = 64
    const ctx = canvas.getContext('2d')
    ctx.fillStyle='white'
    ctx.font='bold 36px Poppins, sans-serif'
    ctx.textAlign='center'
    ctx.shadowColor = 'rgba(0,0,0,0.8)'
    ctx.shadowBlur = 6
    ctx.fillText(s.label, 128, 42)
    const texture = new THREE.CanvasTexture(canvas)
    const labelSprite = new THREE.Sprite(new THREE.SpriteMaterial({ map:texture, transparent:true }))
    labelSprite.scale.set(32,8,1)
    planet.add(labelSprite)
    labelSprite.position.set(0, planetRadius + 4, 0)

    scene.add(planet)
    planets.push(planet)
  })

  // Stars
  const starCount = 2000
  const starGeometry = new THREE.BufferGeometry()
  const starPositions = []
  for (let i = 0; i < starCount; i++) {
    const x = (Math.random() - 0.5) * 2000
    const y = (Math.random() - 0.5) * 2000
    const z = (Math.random() - 0.5) * 2000
    starPositions.push(x, y, z)
  }
  starGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starPositions, 3))
  const starMaterial = new THREE.PointsMaterial({
    color: 0xffffff,
    size: 1.5,
    transparent: true,
    opacity: 0.9
  })
  const stars = new THREE.Points(starGeometry, starMaterial)
  scene.add(stars)

  // Name
  const canvasName = document.createElement('canvas')
  canvasName.width = 700
  canvasName.height = 128
  const ctxName = canvasName.getContext('2d')
  ctxName.fillStyle = 'white'
  ctxName.font = 'bold 40px Poppins, sans-serif'
  ctxName.textAlign = 'center'
  ctxName.textBaseline = 'middle'
  ctxName.shadowColor = 'rgba(0,0,0,0.8)'
  ctxName.shadowBlur = 8
  ctxName.fillText('Solar System', canvasName.width/2, canvasName.height/2)
  const nameSprite = new THREE.Sprite(new THREE.SpriteMaterial({
    map: new THREE.CanvasTexture(canvasName),
    transparent:true
  }))
  nameSprite.scale.set(250,45,1)
  nameSprite.position.set(0, 120, 0)
  scene.add(nameSprite)

  // Alien plates
  const alienData = [
    { position: [0, 68, 0], label: '' },
    { position: [-60, 62, 0], label: '' },
    { position: [60, 62, 0], label: '' },
  ]

  alienData.forEach(a => {
    const baseGeo = new THREE.SphereGeometry(15, 32, 32)
    baseGeo.scale(1.5, 0.3, 1)
    const baseMat = new THREE.MeshStandardMaterial({ 
      color: 0x3b3f40, 
      emissive: 0x3b3f40,
      emissiveIntensity: 1.5,
      roughness: 0.1, 
      metalness: 0.9, 
      transparent: true, 
      opacity: 0.9 
    })
    const base = new THREE.Mesh(baseGeo, baseMat)
    base.position.set(...a.position)
    scene.add(base)

    const lineGeo = new THREE.TorusGeometry(11 * 1.5, 0.8, 8, 64)
    const lineMat = new THREE.MeshStandardMaterial({
      color: 0xff4500,
      emissive: 0xff4500,
      emissiveIntensity: 1.5,
      roughness: 0.1,
      metalness: 0.9
    })
    const line = new THREE.Mesh(lineGeo, lineMat)
    line.rotation.x = Math.PI / 2
    line.position.set(a.position[0], a.position[1] + 2.2, a.position[2])
    scene.add(line)

    const domeGeo = new THREE.SphereGeometry(7, 32, 32, 0, Math.PI*2, 0, Math.PI/2)
    const domeMat = new THREE.MeshStandardMaterial({ 
      color: 0xc8f7fd, 
      emissive: 0xc8f7fd,
      emissiveIntensity: 1.2,
      roughness: 0.2, 
      metalness: 0.6,
      transparent: true, 
      opacity: 0.8 
    })
    const dome = new THREE.Mesh(domeGeo, domeMat)
    dome.position.set(a.position[0], a.position[1]+2, a.position[2])
    scene.add(dome)

// Spots (concave parabola alignment)
const spotsCount = 6
const spread = 20      // რამდენად გაიშალოს X-ზე
const curveDepth = 2.5  // უარყოფითი რომ ჩაზნექილი იყოს

for (let i = 0; i < spotsCount; i++) {
  const t = (i / (spotsCount - 1)) * 2 - 1   // -1 → +1
  const x = t * spread + a.position[0]
  const z = a.position[2]
  const y = a.position[1] + curveDepth * (t * t) -2
  // ↑ y-ზე მივამატე curveDepth * t² → პარაბოლა, ჩაზნექილი რადგან curveDepth < 0

  const spotGeo = new THREE.SphereGeometry(1.2, 16, 16)
  const spotMat = new THREE.MeshStandardMaterial({
    color: 0xff0000,
    emissive: 0xff0000,
    emissiveIntensity: 1.2,
    roughness: 0.1,
    metalness: 0.8
  })
  const spot = new THREE.Mesh(spotGeo, spotMat)
  spot.position.set(x, y, z)
  scene.add(spot)
}


    const canvas = document.createElement('canvas')
    canvas.width = 256
    canvas.height = 64
    const ctx = canvas.getContext('2d')
    ctx.fillStyle = 'white'
    ctx.font = 'bold 50px Poppins, sans-serif'
    ctx.textAlign = 'center'
    ctx.shadowColor = 'rgba(0,0,0,0.9)'
    ctx.shadowBlur = 8
    ctx.fillText(a.label, 128, 42)
    const texture = new THREE.CanvasTexture(canvas)
    const labelSprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: texture, transparent: true }))
    labelSprite.scale.set(20, 5, 1)
    labelSprite.position.set(a.position[0], a.position[1]+12, a.position[2])
    scene.add(labelSprite)
  })

  // Controls
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.06
  controls.autoRotate = false
  controls.minDistance = 80
  controls.maxDistance = 600

const onResize = () => {
  camera.aspect = container.clientWidth / container.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(container.clientWidth, container.clientHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2))
}
window.addEventListener('resize',onResize)

  const clock = new THREE.Clock()
  const animate = ()=>{
    frameId = requestAnimationFrame(animate)
    const t = clock.getElapsedTime()

    sun.rotation.y += sun.userData.rotationSpeed

    planets.forEach(p=>{
      const { radius,speed, rotationSpeed } = p.userData
      const x = Math.cos(t*speed)*radius
      const z = Math.sin(t*speed)*radius
      p.position.set(x,0,z)
      p.rotation.y += rotationSpeed
    })

    controls.update()
    renderer.render(scene,camera)
  }
  animate()

  onBeforeUnmount(()=>{
    cancelAnimationFrame(frameId)
    window.removeEventListener('resize',onResize)
    controls.dispose()
    renderer.dispose()
    scene.traverse(obj=>{
      if(obj.geometry) obj.geometry.dispose()
      if(obj.material){
        if(Array.isArray(obj.material)) obj.material.forEach(m=>m.dispose())
        else obj.material.dispose()
      }
      if(obj.texture) obj.texture.dispose()
    })
    if(container.firstChild) container.removeChild(container.firstChild)
  })
})
</script>

<style scoped>
.atom-container{
  width:100%;
  height:100vh;
  background: radial-gradient(circle at center, #0b0c1c 0%, #1a1a2e 100%);
  display:block;
}
</style>
