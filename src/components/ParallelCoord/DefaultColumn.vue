<template>
  <g class="default-column" v-bind="positioningValues">
    <!-- Column Title -->
    <g class="title">
      <text fill="#17a2b8">{{ dimensionKey }}</text>
      <text fill="#17a2b8">{{ value | rounded }}</text>
    </g>

    <!-- Line -->
    <line :x1="0" :y1="0" :x2="0" :y2="height" fill="none" stroke="#006cab" />

    <!-- Tick Group -->
    <g class="ticks">
      <!-- Tick Item -->
      <g
        v-for="(item, index) in ticks"
        class="tick__text"
        :transform="item.transform"
        fill="#006cab"
        :key="index"
      >
        <!-- Tick Line -->
        <line x1="4" y1="0" x2="10" y2="0" stroke="#006cab" stroke-width="1" />

        <!-- Text -->
        <text>{{ item.v }}</text>
      </g>
    </g>

    <column-brush
      :domain="extent"
      :value="value"
      v-bind="{ x: -10, width, height }"
      @input="$emit('input', $event);"
    />
  </g>
</template>

<script>
import ColumnBrushVue from './ColumnBrush.vue'
// eslint-disable-next-line no-unused-vars
import { Dimension } from 'crossfilter2'
import { scaleLinear } from 'd3'
export default {
  props: {
    /** @type {Dimension} */
    cfDimension: {
      type: Object,
      required: true
    },
    dimensionKey: {
      type: String,
      required: true
    },
    value: {
      type: Array
    },
    x: {
      type: Number,
      default: 0
    },

    y: {
      type: Number,
      default: 0
    },

    width: {
      type: Number,
      default: 20
    },

    height: {
      type: Number,
      default: 500
    }
  },
  filters: {
    rounded (val) {
      if (!val) return null
      return val.map(v => Math.floor(v))
    }
  },
  model: {
    prop: 'value',
    event: 'input'
  },
  components: {
    ColumnBrush: ColumnBrushVue
  },
  computed: {
    extent () {
      /** @type {Dimension} */
      const d = this.cfDimension
      if (d) {
        const top = d.top(1)[0][this.dimensionKey]
        const bottom = d.bottom(1)[0][this.dimensionKey]

        return [top, bottom].sort((a, b) => a - b)
      }
    },
    scale () {
      if (!this.extent) return
      return scaleLinear()
        .domain(this.extent)
        .nice(10)
        .range([this.height, 0])
    },
    ticks () {
      if (this.scale) {
        const values = this.scale.ticks()
        return values.map(v => {
          return {
            y: this.scale(v),
            v,
            transform: `translate(-10 ${this.scale(v)})`
          }
        })
      }
    },
    positioningValues () {
      const transform = `translate(${this.x} ${this.y})`
      return {
        transform
      }
    }
  }
}
</script>

<style scoped lang="scss">
  .title {
    text-anchor: middle;
    font-size: 12px;
    transform: translateY(-20px);

    > text {
      &:first-child {
        dominant-baseline: text-after-edge;
      }
      &:last-child {
        dominant-baseline: hanging;
        font-weight: bold;
        fill: #00ff8d;
      }
    }
  }
  .tick {
    &__text {
      text-anchor: end;
      dominant-baseline: central;
      font-size: 10px;
      padding-right: 20px;
    }
  }
</style>
