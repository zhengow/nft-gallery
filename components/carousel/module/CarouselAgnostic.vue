<template>
  <div class="carousel-agnostic">
    <div class="navigation-wrapper">
      <div ref="wrapper" class="keen-slider">
        <div
          v-for="(item, index) in nfts"
          :key="`${item.id}-${index}`"
          class="keen-slider__slide carousel-item">
          <div>
            <CarouselMedia :item="item" />
            <CarouselInfo :item="item" />
          </div>
        </div>
      </div>
      <div class="arrow arrow-left" @click="slider?.prev()"></div>
      <div class="arrow arrow-right" @click="slider?.next()"></div>
    </div>
    <div v-if="slider && !isCollection" class="dots">
      <button
        v-for="(_slide, idx) in dotHelper"
        :key="idx"
        :class="{ dot: true, active: current === idx }"
        @click="slider?.moveToIdx(idx)"></button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import type { CarouselNFT } from '@/components/base/types'

import CarouselMedia from './CarouselMedia.vue'
import CarouselInfo from './CarouselInfo.vue'

import 'keen-slider/keen-slider.min.css'
import { useKeenSlider } from 'keen-slider/vue.es'
import { wheelControls } from '../utils/useCarousel'

const props = defineProps<{
  nfts: CarouselNFT[]
}>()

const url = inject('itemUrl') as string
const isCollection = computed(() => url.includes('collection'))
provide('isCollection', isCollection.value)

const current = ref(0)
const minWidths = [1280, 1024, 768, 640]
const [wrapper, slider] = useKeenSlider(
  {
    initial: current.value,
    rubberband: false,
    slideChanged: (s) => {
      current.value = s.track.details.rel
    },
    breakpoints: {
      '(min-width: 640px)': {
        slides: { perView: 1.5, spacing: 32 },
      },
      '(min-width: 768px)': {
        slides: { perView: 2.5, spacing: 32 },
      },
      '(min-width: 1024px)': {
        slides: { perView: 3.5, spacing: 32 },
      },
      '(min-width: 1280px)': {
        slides: { perView: 4.5, spacing: 32 },
      },
    },
    slides: { perView: 1.5, spacing: 32 },
  },
  [wheelControls]
)
const totalDots = computed(() => {
  const width = window.innerWidth

  for (const [index, breakpoint] of minWidths.entries()) {
    if (breakpoint <= width) {
      const perView = 4.5 - (index + 1)
      return Math.round(props.nfts.length - perView)
    }
  }
})
const dotHelper = computed(() =>
  slider.value ? [...Array(totalDots.value).keys()] : []
)
</script>
