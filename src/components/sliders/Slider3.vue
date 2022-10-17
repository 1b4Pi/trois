<template>
  <Renderer ref="renderer" antialias resize pointer>
    <OrthographicCamera ref="camera" :position="{ z: 10 }" />
    <Scene ref="scene" />
  </Renderer>
</template>

<script>
import { defineComponent, ref, nextTick, onMounted, onBeforeUnmount } from 'vue'
import { TextureLoader, Vector2 } from 'three'
import { gsap, Power4 } from 'gsap'

// import { lerp, useTextures, OrthographicCamera, Renderer, Scene } from '../../../build/trois.module.js'
import { lerp, OrthographicCamera, Renderer, Scene } from '../../../build/trois.module.js'
import ZoomBlurImage from './ZoomBlurImage.js'

export default defineComponent({
  components: { OrthographicCamera, Renderer, Scene },
  props: {
    images: Array,
    events: { type: Object, default: () => { return { wheel: true, click: true, keyup: true } } },
  },
  setup(props) {
    const center = new Vector2()
    const renderer = ref(null);
    const camera = ref(null);
    const scene = ref(null);
    const three = ref(null);
    const image1 = ref(null);
    const image2 = ref(null);
    const progress = ref(0);
    const targetProgress = ref(0);

    const useTextures = function () {
      console.log("use");
      const obj = {
        loader: new TextureLoader(),
        count: 0,
        textures: [],
        loadProgress: 0,
        loadTextures,
        dispose
      };
      return obj;
      function loadTextures(images, cb) {
        // console.log("loadTextures", images, cb)
        obj.count = images.length;
        obj.textures.splice(0);
        obj.loadProgress = 0;
        Promise.all(images.map(loadTexture)).then(init());
      }
      function loadTexture(img, index) {
        console.log("loadTexture", img, index)
        return new Promise((resolve) => {
          obj.loader.load(img.src, (texture) => {
            obj.loadProgress += 1 / obj.count;
            obj.textures[index] = texture;
            resolve(texture);
          });
        });
      }
      function dispose() {
        obj.textures.forEach((t) => t.dispose());
      }
    };

    const loader = useTextures()

    const initScene = function() {
      console.log("initScene")
      console.log("domElement", renderer.value.renderer.domElement);
      // const scene = this.$refs.scene.scene
      image1.value = new ZoomBlurImage(renderer.value)
      image1.value.setMap(loader.textures[0])
      image2.value = new ZoomBlurImage(renderer.value)
      image2.value.setMap(loader.textures[1])
      setImagesProgress(0)

      scene.value.add(image1.value.mesh)
      scene.value.add(image2.value.mesh)
    };

    const init = function() {
      console.log("init???");
      initScene()
      gsap.fromTo(image1.value.uStrength,
        {
          value: -2,
        },
        {
          value: 0,
          duration: 2.5,
          ease: Power4.easeOut,
        }
      )

      const domElement = renderer.value.renderer.domElement
      if (props.events.click) domElement.addEventListener('click', onClick)
      if (props.events.wheel) domElement.addEventListener('wheel', onWheel)
      if (props.events.keyup) document.addEventListener('keyup', onKeyup)
      renderer.value.onBeforeRender(animate())
      renderer.value.onResize(onResize())
    };

    const animate = function() {
      console.log("animating?????");
      const { positionN } = renderer.value.three.pointer
      center.copy(positionN).divideScalar(2).addScalar(0.5)
      image1.value.uCenter.value.lerp(center, 0.1)
      image2.value.uCenter.value.lerp(center, 0.1)
      // lerpv2(this.image1.uCenter.value, this.center, 0.1)
      // lerpv2(this.image2.uCenter.value, this.center, 0.1)
      updateProgress()
    };

    const onResize = function() {
      image1.value.updateUV()
      image2.value.updateUV()
    };

    const onWheel = function(e) {
      // e.preventDefault()
      if (e.deltaY > 0) {
        setTargetProgress(targetProgress.value + 1 / 20)
      } else {
        setTargetProgress(targetProgress.value - 1 / 20)
      }
    };

    const onClick = function(e) {
      console.log("onClick", e);
      if (e.clientY < renderer.value.size.height / 2) {
        navPrevious()
      } else {
        navNext()
      }
    };

    const onKeyup = function(e) {
      if (e.keyCode === 37 || e.keyCode === 38) {
        navPrevious()
      } else if (e.keyCode === 39 || e.keyCode === 40) {
        navNext()
      }
    };

    const navNext = function() {
      if (Number.isInteger(targetProgress)) setTargetProgress(targetProgress.value + 1)
      else setTargetProgress(Math.ceil(targetProgress))
    };

    const navPrevious = function() {
      if (Number.isInteger(targetProgress)) setTargetProgress(targetProgress.value - 1)
      else setTargetProgress(Math.floor(targetProgress))
    };

    const setTargetProgress = function(value) {
      targetProgress.value = value
      if (targetProgress.value < 0) {
        progress += props.images.length
        targetProgress.value += props.images.length
      }
    };

    const updateProgress = function() {
      const progress1 = lerp(progress.value, targetProgress.value, 0.1)
      const pdiff = progress1 - progress.value
      if (pdiff === 0) return

      const p0 = progress.value % 1
      const p1 = progress1 % 1
      if ((pdiff > 0 && p1 < p0) || (pdiff < 0 && p0 < p1)) {
        const i = Math.floor(progress1) % props.images.length
        const j = (i + 1) % props.images.length
        image1.value.setMap(loader.textures[i])
        image2.value.setMap(loader.textures[j])
      }

      progress.value = progress1;
      setImagesProgress(progress.value % 1)
    };

    const setImagesProgress = function(progress) {
      image1.value.uStrength.value = progress
      image2.value.uStrength.value = -1 + progress
    };

    onMounted(() => {
      console.log("mounted props", props.images)
      // renderer.value = this.$refs.renderer
      // this.three = this.renderer.three
      nextTick(() => {
        three.value = renderer.value.three

        if (props.images.length < 2) {
          console.error('This slider needs at least 2 images.')
        } else {
          loader.loadTextures(props.images, init)
        }
      })
    });

    onBeforeUnmount(() => {
      loader.dispose()
      const domElement = renderer.value.renderer.domElement
      domElement.removeEventListener('click', onClick)
      domElement.removeEventListener('wheel', onWheel)
      document.removeEventListener('keyup', onKeyup)
    });

    return {
      camera,
      center,
      loader,
      renderer,
      scene,
      progress,
      targetProgress,
    }
  },
  // unmounted() {
  //   this.loader.dispose()
  //   const domElement = this.renderer.renderer.domElement
  //   domElement.removeEventListener('click', this.onClick)
  //   domElement.removeEventListener('wheel', this.onWheel)
  //   document.removeEventListener('keyup', this.onKeyup)
  // },
})
</script>
