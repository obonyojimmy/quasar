<template>
  <div class="q-slider" :class="{fullscreen: inFullscreen}">
    <div class="q-slider-inner">
      <div
        ref="track"
        class="q-slider-track"
        :class="{'with-arrows': arrows, 'with-toolbar': toolbar}"
        v-touch-pan.horizontal="__pan"
      >
        <slot name="slide"></slot>
      </div>
      <div
        v-if="arrows"
        class="q-slider-left-button row items-center justify-center"
        :class="{hidden: slide === 0}"
      >
        <i @click="goToSlide(slide - 1)">keyboard_arrow_left</i>
      </div>
      <div
        v-if="arrows"
        class="q-slider-right-button row items-center justify-center"
        :class="{hidden: slide === slidesNumber - 1}"
        @click="goToSlide(slide + 1)"
      >
        <i>keyboard_arrow_right</i>
      </div>
      <div v-if="toolbar" class="q-slider-toolbar row items-center justify-end">
        <div class="q-slider-dots auto row items-center justify-center">
          <i
            v-if="dots"
            v-for="n in slidesNumber"
            @click="goToSlide(n - 1)"
            v-text="(n - 1) !== slide ? 'panorama_fish_eye' : 'lens'"
          ></i>
        </div>
        <div class="row items-center">
          <slot name="action"></slot>
          <i v-if="fullscreen" @click="toggleFullscreen()">
            <span v-show="!inFullscreen">fullscreen</span>
            <span v-show="inFullscreen">fullscreen_exit</span>
          </i>
        </div>
      </div>
      <slot></slot>
    </div>
  </div>
</template>

<script>
import Platform from '../../features/platform'

export default {
  props: {
    arrows: Boolean,
    dots: Boolean,
    fullscreen: Boolean,
    actions: Boolean
  },
  data () {
    return {
      position: 0,
      slide: 0,
      slidesNumber: 0,
      inFullscreen: false
    }
  },
  watch: {
    slide (value) {
      this.$emit('slide', value)
    }
  },
  computed: {
    toolbar () {
      return this.dots || this.fullscreen || this.actions
    }
  },
  methods: {
    __pan (event) {
      if (!this.hasOwnProperty('initialPosition')) {
        this.initialPosition = this.position
        Velocity(this.$refs.track, 'stop')
      }

      let delta = (event.direction === 'left' ? -1 : 1) * event.distance.x

      if (
        this.slide === 0 && delta > 0 ||
        this.slide === this.slidesNumber - 1 && delta < 0
      ) {
        delta = delta / 10
      }

      this.position = this.initialPosition + delta / this.$refs.track.offsetWidth * 100
      this.$refs.track.style.transform = 'translateX(' + this.position + '%)'

      if (event.isFinal) {
        if (event.distance.x < 100) {
          this.goToSlide(this.slide)
        }
        else {
          this.goToSlide(event.direction === 'left' ? this.slide + 1 : this.slide - 1)
        }
        delete this.initialPosition
      }
    },
    goToSlide (slide, noAnimation) {
      this.slide = Math.min(this.slidesNumber - 1, Math.max(0, slide))

      Velocity(this.$refs.track, 'stop')
      Velocity(this.$refs.track, {
        translateX: [-this.slide * 100 + '%', this.position + '%']
      }, noAnimation ? {duration: 0} : undefined)

      this.position = -this.slide * 100
    },
    toggleFullscreen () {
      if (this.inFullscreen) {
        if (Platform.within.iframe) {
          this.inFullscreen = false
        }
        else {
          window.history.go(-1)
        }
        return
      }

      this.inFullscreen = true
      if (!Platform.within.iframe) {
        window.history.pushState({}, '')
        window.addEventListener('popstate', this.__popState)
      }
    },
    __popState () {
      if (this.inFullscreen) {
        this.inFullscreen = false
      }
      window.removeEventListener('popstate', this.__popState)
    }
  },
  mounted () {
    this.$nextTick(() => {
      this.slidesNumber = this.$refs.track.children.length
    })
  }
}
</script>
