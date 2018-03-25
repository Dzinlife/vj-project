<template>
  <div id="app">
    <h1>TOUCH LIGHT</h1>
    <canvas ref="canvas" @mousemove="handleMouseMove"></canvas>
  </div>
</template>

<script>
const THREE = window.THREE

export default {
  name: 'App',
  mounted() {
    this.scene = new THREE.Scene()
    this.scene1 = new THREE.Scene()
    this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)

    this.renderer = new THREE.WebGLRenderer({canvas: this.$refs.canvas, antialias: true})
    this.renderer.setPixelRatio(window.devicePixelRatio)
    this.renderer.shadowMap.enabled = true
    // this.renderer.shadowMap.type = THREE.PCFSoftShadowMap
    // this.renderer.gammaInput = true
    // this.renderer.gammaOutput = true
    this.renderer.setSize(window.innerWidth, window.innerHeight)
    this.renderer.autoClear = false

    const ambient = new THREE.AmbientLight(0xcccccc, 0.1)
    this.scene.add(ambient)

    this.rectLight = new THREE.RectAreaLight(0xffffff, 0.5, 200, 200)
    this.rectLight.rotation.x = Math.PI / 2
    this.rectLight.position.y = 300
    this.rectLight.castShadow = true
    this.scene.add(this.rectLight)

    // const spotLight = new THREE.SpotLight(0xffffff, 1)
    // spotLight.position.set(50, 100, 50)
    // spotLight.angle = Math.PI / 7
    // spotLight.penumbra = 0.8
    // spotLight.castShadow = true
    // this.scene.add(spotLight)

    // const rectLightMesh = new THREE.Mesh(new THREE.PlaneBufferGeometry(), new THREE.MeshBasicMaterial())
    // rectLightMesh.scale.x = this.rectLight.width
    // rectLightMesh.scale.y = this.rectLight.height
    // this.rectLight.add(rectLightMesh)

    // const rectLightMeshBack = new THREE.Mesh(new THREE.PlaneBufferGeometry(), new THREE.MeshBasicMaterial({color: 0x080808}))
    // rectLightMeshBack.rotation.y = Math.PI
    // rectLightMesh.add(rectLightMeshBack)

    // const geoFloor = new THREE.BoxGeometry(20000, 0.1, 20000)
    // const matStdFloor = new THREE.MeshStandardMaterial({color: 0x808080, roughness: 0, metalness: 0})
    // const mshStdFloor = new THREE.Mesh(geoFloor, matStdFloor)
    // mshStdFloor.position.y = -150
    // this.scene.add(mshStdFloor)

    const matStdObjects = new THREE.MeshStandardMaterial({color: 0xffffff, roughness: 0.6, metalness: 0})
    // const matStdObjects = new THREE.MeshPhongMaterial({color: 0xffff00})

    this.baseRadius = 200
    const baseGeometry = new THREE.CylinderGeometry(this.baseRadius, this.baseRadius, 1, 64)
    const baseMaterial = new THREE.MeshBasicMaterial({color: 0x000000})
    this.base = new THREE.Mesh(baseGeometry, baseMaterial)
    this.base.castShadow = true
    this.base.receiveShadow = true

    const ballGeometry = new THREE.SphereGeometry(30, 32, 32)
    this.bulbMat = new THREE.MeshStandardMaterial({
      emissive: 0xffffee,
      emissiveIntensity: 0.1,
      color: 0x000000
    })
    this.ball = new THREE.PointLight(0xffffff, 0, 50)
    this.ball.add(new THREE.Mesh(ballGeometry, this.bulbMat))
    this.ball.position.y = 30 + 5
    this.ball1 = this.ball.clone()

    this.device = new THREE.Object3D()
    this.device.add(this.base)
    this.device.add(this.ball)
    // this.device1 = new THREE.Object3D()
    // this.device1.add(this.ball1)

    this.scene.add(this.device)
    // this.scene1.add(this.device1)

    this.ballVelocity = new THREE.Vector3(0, 0, 0)

    this.camera.position.z = 400
    this.camera.position.y = 100
    this.camera.lookAt(new THREE.Vector3(0, 0, 0))

    this.initPostprocessing()

    this.outlinePass.selectedObjects = [this.base]
    this.outlinePass.edgeThickness = 1
    this.outlinePass.edgeStrength = 10
    this.outlinePass.visibleEdgeColor.set('#ffffff')
    this.outlinePass.hiddenEdgeColor.set('#000000')
    this.outlinePass2.selectedObjects = [this.ball]
    this.outlinePass2.edgeThickness = 1
    this.outlinePass2.edgeStrength = 10
    this.outlinePass2.visibleEdgeColor.set('#ffffff')
    this.outlinePass2.hiddenEdgeColor.set('#000000')

    this.animate()
  },
  beforeDestroy() {
    cancelAnimationFrame(this.raf)
    this.scene = null
  },
  computed: {
  },
  methods: {
    initPostprocessing() {
      const renderTarget = new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight, {
        minFilter: THREE.LinearFilter,
        magFilter: THREE.LinearFilter,
        format: THREE.RGBFormat,
        stencilBuffer: true
      })

      this.composer = new THREE.EffectComposer(this.renderer)
      this.composer.renderTarget1.stencilBuffer = true
      this.composer.renderTarget2.stencilBuffer = true

      const renderPass = new THREE.RenderPass(this.scene, this.camera)
      renderPass.clear = false
      const renderPass1 = new THREE.RenderPass(this.scene1, this.camera)
      renderPass1.clear = false

      const clearPass = new THREE.ClearPass()
      const clearMaskPass = new THREE.ClearMaskPass()
      const maskPass = new THREE.MaskPass(this.scene, this.camera)
      const maskPass1 = new THREE.MaskPass(this.scene1, this.camera)
      const outputPass = new THREE.ShaderPass(THREE.CopyShader)
      outputPass.renderToScreen = true

      const effectGrayScale = new THREE.ShaderPass(THREE.LuminosityShader)
      this.composer.addPass(effectGrayScale)

      const bloomPass = new THREE.UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 1, 0, 512) // 1.0, 9, 0.5, 512);
      console.log(bloomPass)
      // bloomPass.renderToScreen = true

      const effectSobel = new THREE.ShaderPass(THREE.SobelOperatorShader)
      // effectSobel.renderToScreen = true
      effectSobel.uniforms.resolution.value.x = window.innerWidth
      effectSobel.uniforms.resolution.value.y = window.innerHeight

      this.outlinePass = new THREE.OutlinePass( new THREE.Vector2(window.innerWidth, window.innerHeight), this.scene, this.camera)      
      this.outlinePass2 = new THREE.OutlinePass( new THREE.Vector2(window.innerWidth, window.innerHeight), this.scene, this.camera)      

      this.composer.addPass(clearPass)
      this.composer.addPass(renderPass)
      this.composer.addPass(this.outlinePass)
      this.composer.addPass(this.outlinePass2)
      // this.composer.addPass(renderPass1)
      // this.composer.addPass(maskPass)
      // this.composer.addPass(effectGrayScale)
      // this.composer.addPass(effectSobel)
      // this.composer.addPass(clearMaskPass)
      // this.composer.addPass(maskPass1)
      this.composer.addPass(bloomPass)
      // this.composer.addPass(clearMaskPass)
      this.composer.addPass(outputPass)
    },
    handleMouseMove(e) {
      const x = (e.clientX / this.width) - 0.5
      const y = (e.clientY / this.height) - 0.5
      this.$set(this.data, 1, x < 0 ? -x : 0)
      this.$set(this.data, 4, x > 0 ? x : 0)
      this.$set(this.data, 3, y < 0 ? -y : 0)
      this.$set(this.data, 2, y > 0 ? y : 0)
    },
    getRotation() {
      const x = Math.PI * (this.data[2] - this.data[3])
      const z = Math.PI * (this.data[1] - this.data[4])
      return THREE.Vector3(x, 0, z)
    },
    getDeviceYDirection() {
      const matrix = new THREE.Matrix4()
      matrix.extractRotation(this.device.matrix)

      const direction = new THREE.Vector3(0, 1, 0)
      direction.applyMatrix4(matrix)
      return direction
    },
    getAccerlation() {
      return this.getDeviceYDirection().projectOnPlane(new THREE.Vector3(0, 1, 0)).multiplyScalar(0.5)
    },
    animate() {
      this.raf = requestAnimationFrame(this.animate)
      const x = Math.PI * (this.data[2] - this.data[3])
      const z = Math.PI * (this.data[1] - this.data[4])
      this.device.rotation.x = x
      this.device.rotation.z = z
      // this.device1.rotation.x = x
      // this.device1.rotation.z = z

      this.ballVelocity.add(this.getAccerlation())
      let position = new THREE.Vector3(this.ball.position.x, 0, this.ball.position.z)

      position.x += this.ballVelocity.x
      position.z += this.ballVelocity.z

      if (position.length() >= this.baseRadius) {
        this.ballVelocity.reflect(position.clone().normalize()).multiplyScalar(0.64)
        position.clampLength(0, this.baseRadius)
      }
      this.ball.position.x = position.x
      this.ball.position.z = position.z
      // this.ball1.position.x = position.x
      // this.ball1.position.z = position.z

      const t = (Date.now() / 2000)
      this.$set(this.data, 0, (Math.sin(t) + 1) / 2)
      this.ball.intensity = this.data[0]
      this.ball.distance = this.data[0] * 300 + 50
      this.bulbMat.emissiveIntensity = this.data[0] * 1.5
      // const r = 400.0
      // const lx = r * Math.cos(t)
      // const lz = r * Math.sin(t)
      // const ly = 0
      // this.rectLight.position.set(lx, ly, lz)
      // this.rectLight.lookAt(new THREE.Vector3(0, 0, 0))
      this.composer.render()
      // this.renderer.render(this.scene, this.camera)
    }
  },
  data() {
    return {
      width: window.innerWidth,
      height: window.innerHeight,
      data: [0, 0, 0, 0, 0] // 0:light，1:left, 2:down, 3:up, 4:right
    }
  }
}
</script>

<style lang="stylus">
  html
    height 100%
    background gray
  body
    height 100%
    margin 0
</style>

<style lang="stylus" scoped>
#app {
  height 100%
  font-family 'Avenir', Helvetica, Arial, sans-serif
  -webkit-font-smoothing antialiased
  -moz-osx-font-smoothing grayscale
}

h1
  position absolute
  left 10px
  top 10px
  color white

.container
  height 100%
canvas
  height 100%
  width 100%

</style>
