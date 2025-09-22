<template>
  <div class="container">
    <div ref="canvasContainer" class="canvas"></div>

    <aside class="controls">
      <h3>Door controls</h3>
      <label>Width: {{ doorWidth.toFixed(2) }} m
        <input type="range" min="0.1" max="10.0" step="0.01" v-model.number="doorWidth" @input="updateWidthKnob" />
      </label>
      <label>Height: {{ doorHeight.toFixed(2) }} m
        <input type="range" min="0.1" max="10.0" step="0.01" v-model.number="doorHeight" @input="updateHeightKnob" />
      </label>
      <label>Lock proportions
        <input type="checkbox" v-model="lockProportions" @input="updateLockProportions" />
      </label>
      <button @click="resetDoor">Reset</button>

      <hr />
      <h4>Scene</h4>
      <label>
        Show grid
        <input type="checkbox" v-model="showHelpers" @change="toggleHelpers" />
      </label>
      Move with <span class="key">W</span>/<span class="key">A</span>/<span class="key">S</span>/<span class="key">D</span>
    </aside>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onBeforeUnmount, ref, watch } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { PointerLockControls } from 'three/examples/jsm/controls/PointerLockControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { RoomEnvironment } from 'three/examples/jsm/environments/RoomEnvironment'

export default defineComponent({
  name: 'ThreeDoorScene',
  setup() {
    const canvasContainer = ref<HTMLDivElement | null>(null)

    // door parameters (meters)
    const doorWidth = ref(0.9)
    const doorHeight = ref(2.1)
    const doorDepth = ref(0.04)
    const lockProportions = ref(true)

    const showHelpers = ref(false)

    let renderer: THREE.WebGLRenderer
    let scene: THREE.Scene
    let camera: THREE.PerspectiveCamera
    let controls: PointerLockControls
    let pmremGenerator: THREE.PMREMGenerator

    // scene objects
    let doorMesh: THREE.Mesh | null = null
    let floorMesh: THREE.Mesh
    let geoSphere: THREE.Mesh
    let geoTorus: THREE.Mesh
    let dirLight: THREE.DirectionalLight
    let gridHelper: THREE.GridHelper
    let axesHelper: THREE.AxesHelper
    let doorGroup: THREE.Group;
    let doorSize: THREE.Vector3;
    let direction: THREE.Vector3 = new THREE.Vector3();
    let meta: THREE.Vector3 = new THREE.Vector3();
    let r: THREE.Vector3 = new THREE.Vector3();
    let up: THREE.Vector3 = new THREE.Vector3(0, 1, 0);

    // kb controls
    let kb: { [key: string]: boolean } = {}

    const woodTexturePath = '/assets/wood.jpg' // put a wood.jpg into public/assets/wood.jpg


    function createRenderer(container: HTMLDivElement) {
      renderer = new THREE.WebGLRenderer({ antialias: true })
      renderer.setPixelRatio(window.devicePixelRatio)
      renderer.shadowMap.enabled = true
      renderer.shadowMap.type = THREE.PCFSoftShadowMap
      renderer.setSize(container.clientWidth, container.clientHeight)
      container.appendChild(renderer.domElement)
    }

    function initScene(container: HTMLDivElement) {
      scene = new THREE.Scene()

      // Camera
      camera = new THREE.PerspectiveCamera(60, container.clientWidth / container.clientHeight, 0.1, 100)
      camera.position.set(2.5, 1.45, 2.5)
      camera.lookAt(0, 1, 0)

      // Controls
      controls = new PointerLockControls(camera, renderer.domElement)
      //controls.target.set(0, 1.0, 0)
      //controls.update()

      // Environment (RoomEnvironment + PMREM for good reflections)
      pmremGenerator = new THREE.PMREMGenerator(renderer)
      pmremGenerator.compileEquirectangularShader()
      const roomEnv = new RoomEnvironment()
      const envMap = pmremGenerator.fromScene(roomEnv as unknown as THREE.Scene).texture
      scene.environment = envMap
      scene.background = new THREE.Color(0xaaaaaa)

      // Light: directional for crisp shadows
      dirLight = new THREE.DirectionalLight(0xffffff, 1.0)
      dirLight.position.set(2, 4, 2)
      dirLight.castShadow = true
      dirLight.shadow.mapSize.set(2048, 2048)
      dirLight.shadow.camera.left = -5
      dirLight.shadow.camera.right = 5
      dirLight.shadow.camera.top = 5
      dirLight.shadow.camera.bottom = -5
      dirLight.shadow.camera.near = 0.5
      dirLight.shadow.camera.far = 20
      scene.add(dirLight)

      // ambient fill
      scene.add(new THREE.AmbientLight(0xffffff, 0.25))

      // Floor (receives shadows)
      const textureLoader = new THREE.TextureLoader();
      const normalMap = textureLoader.load("assets/floortile_N.jpg")
      const colorMap = textureLoader.load("assets/floortile_S.jpg")
      colorMap.repeat.set(10, 10);
      colorMap.wrapS = THREE.RepeatWrapping;
      colorMap.wrapT = THREE.RepeatWrapping;
      normalMap.repeat.set(10, 10);
      normalMap.wrapS = THREE.RepeatWrapping;
      normalMap.wrapT = THREE.RepeatWrapping;
      const floorMat = new THREE.MeshStandardMaterial({ metalness: 0.1, roughness: 0.8, map: colorMap, normalMap })
      const floorGeo = new THREE.PlaneGeometry(20, 20)
      floorMesh = new THREE.Mesh(floorGeo, floorMat)
      floorMesh.rotateX(-Math.PI / 2)
      floorMesh.receiveShadow = true
      scene.add(floorMesh)

      // Two geometric shapes
      const mat1 = new THREE.MeshStandardMaterial({ metalness: 0.6, roughness: 0.2 })
      const mat2 = new THREE.MeshStandardMaterial({ metalness: 0.0, roughness: 0.6 })

      const sphereGeo = new THREE.SphereGeometry(0.3, 32, 24)
      geoSphere = new THREE.Mesh(sphereGeo, mat1)
      geoSphere.position.set(-1.0, 0.3, 0)
      geoSphere.castShadow = true
      geoSphere.receiveShadow = true
      scene.add(geoSphere)

      const torusGeo = new THREE.TorusKnotGeometry(0.2, 0.06, 120, 16)
      geoTorus = new THREE.Mesh(torusGeo, mat2)
      geoTorus.position.set(1.0, 0.5, -0.5)
      geoTorus.castShadow = true
      geoTorus.receiveShadow = true
      scene.add(geoTorus)

      // helpers
      gridHelper = new THREE.GridHelper(10, 20)
      axesHelper = new THREE.AxesHelper(1.5)
      scene.add(gridHelper)
      scene.add(axesHelper)
      toggleHelpers()

      // skybox cubemap
      const cubeLoader = new THREE.CubeTextureLoader();
      cubeLoader.setPath("/assets/");
      const textureCube = cubeLoader.load( [
      	'posx.jpg', 'negx.jpg',
      	'posy.jpg', 'negy.jpg',
      	'posz.jpg', 'negz.jpg'
      ] );
      scene.environment = textureCube;
      scene.background = textureCube;

      // load GLB
      const loader = new GLTFLoader()
      doorGroup = new THREE.Group();
      doorSize = new THREE.Vector3();
      scene.add(doorGroup);
      loader.load("/assets/wooden_door.glb", gltf => {
          const model = gltf.scene
          // add shadow
          model.traverse(obj => {
	    if (obj.isMesh) {
	      obj.castShadow = true
	      obj.receiveShadow = true
	    }
	  })
          new THREE.Box3().setFromObject(model).getSize(doorSize);
          resetDoor();
          doorGroup.add(model)
      }, undefined, err => {
          console.error("Error loading GLB:", err)
      })


      window.addEventListener('resize', onWindowResize)
      window.addEventListener('keydown', onKeydown)
      window.addEventListener('keyup', onKeyup)
      renderer.domElement.addEventListener("click", lockControls);
    }

    function onKeydown(e) {
      kb[e.code] = true;
    }

    function onKeyup(e) {
      kb[e.code] = false;
    }

    function lockControls() {
      controls.lock();
    }

    function updateLockProportions() {
      if (lockProportions.value) {
        updateWidthKnob();
      }
    }

    function updateWidthKnob() {
      const v = doorWidth.value
      if (lockProportions.value) {
        doorHeight.value = v * doorSize.y / doorSize.x
      }
      doorDepth.value = v * doorSize.z / doorSize.x
      updateDoor()
    }

    function updateHeightKnob() {
      const v = doorHeight.value
      if (lockProportions.value) {
        doorWidth.value = v * doorSize.x / doorSize.y
      }
      doorDepth.value = v * doorSize.z / doorSize.y
      updateDoor()
    }


    function updateDoor() {
      doorGroup.scale.set(doorWidth.value/ doorSize.x, doorHeight.value / doorSize.y, doorDepth.value / doorSize.z);
    }

    function resetDoor() {
      doorWidth.value = 1.
      updateWidthKnob()
    }

    function toggleHelpers() {
      gridHelper.visible = showHelpers.value
      axesHelper.visible = showHelpers.value
    }

    function onWindowResize() {
      if (!canvasContainer.value) return
      const w = canvasContainer.value.clientWidth
      const h = canvasContainer.value.clientHeight
      camera.aspect = w / h
      camera.updateProjectionMatrix()
      renderer.setSize(w, h)
    }

    function moveCamera(delta: number) {
      const v = kb["ShiftLeft"] ? 0.006 : 0.002;
      camera.getWorldDirection(direction);
      direction.y = 0;
      direction.normalize();
      meta.copy(direction);
      r.set(0, 0, 0);
      if (kb["KeyW"]) r.add(direction.multiplyScalar(delta * v));
      direction.copy(meta);
      if (kb["KeyS"]) r.add(direction.multiplyScalar(-delta * v));
      direction.copy(meta);
      if (kb["KeyA"])
        r.add(
          direction
            .cross(up)
            .normalize()
            .multiplyScalar(-delta * v)
        );
      direction.copy(meta);
      if (kb["KeyD"])
        r.add(
          direction
            .cross(up)
            .normalize()
            .multiplyScalar(delta * v)
        );
      direction.copy(meta);

      camera.position.add(r);
    }

    let rafId = 0
    let prevTime = 0
    function animate(time) {
      rafId = requestAnimationFrame(animate)

      // calculate delta
      const delta = (time || 0) - prevTime;
      prevTime = time || 0;

      // move camera
      moveCamera(delta);

      // animate torus
      geoTorus.rotation.x += 0.000375 * delta

      controls.update()
      renderer.render(scene, camera)
    }

    onMounted(() => {
      if (!canvasContainer.value) return
      createRenderer(canvasContainer.value)
      initScene(canvasContainer.value)
      animate()
    })

    onBeforeUnmount(() => {
      window.removeEventListener('resize', onWindowResize)
      window.removeEventListener('keydown', onKeydown)
      window.removeEventListener('keyup', onKeyup)
      renderer.domElement.removeEventListener("click", lockControls)
      cancelAnimationFrame(rafId)
      controls.dispose()
      pmremGenerator.dispose()
      renderer.dispose()
    })

    return {
      canvasContainer,
      doorWidth,
      doorHeight,
      doorDepth,
      lockProportions,
      updateLockProportions,
      updateDoor,
      updateWidthKnob,
      updateHeightKnob,
      resetDoor,
      showHelpers,
      toggleHelpers,
    }
  },
})
</script>

<style scoped>
.container {
  display: grid;
  grid-template-columns: 1fr 320px;
  height: 100vh;
  width: 100vw;
}
.canvas {
  width: 100%;
  height: 100%;
  background: #ddd;
}
.controls {
  padding: 12px;
  background: #f7f7f7;
  border-left: 1px solid #e0e0e0;
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial;
  color: black;
}
.controls label { display: block; margin-bottom: 10px }
.controls input[type=range] { width: 100% }
.controls button { margin-top: 8px }
.key {
  display: inline-block;
  padding: 0.01em 0.25em;
  border-radius: 4px;
  border: 1px solid #b1b2b3;
  box-shadow: 1px 2px 3px #b1b2b3;
}
</style>
