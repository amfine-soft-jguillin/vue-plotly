<template>
<div ref="container" class="vue-plotly"/>
</template>
<script>
import Plotly from 'plotly.js/dist/plotly'
import debounce from 'lodash/debounce'
import defaults from 'lodash/defaults'

const events = [
  'click',
  'hover',
  'unhover',
  'selecting',
  'selected',
  'restyle',
  'relayout',
  'autosize',
  'deselect',
  'doubleclick',
  'redraw',
  'animated'
]

const functions = [
  'restyle',
  'relayout',
  'update',
  'addTraces',
  'deleteTraces',
  'moveTraces',
  'extendTraces',
  'prependTraces',
  'purge'
]

const methods = functions.reduce((all, funcName) => {
  all[funcName] = function(...args) {
    return Plotly[funcName].apply(Plotly, [this.$refs.container].concat(args)).catch((err) => {
      console.log('[safe to ignore] failed during plotly\'s %s : (%s) %s', funcName, err.name, err.message);
    })
  }
  return all
}, {})

export default {
  props: {
    autoResize: Boolean,
    watchShallow: false,
    refresh: {
      required: false
    },
    options: {
      type: Object
    },
    data: {
      type: Array
    },
    layout: {
      type: Object
    }
  },
  data() {
    return {
      internalLayout: this.layout
    }
  },
  mounted() {
    this.newPlot()
    this.initEvents()

    this.$watch('data', this.newPlot, { deep: !this.watchShallow })
    this.$watch('options', this.newPlot, { deep: !this.watchShallow })
    this.$watch('layout', this.relayout, { deep: !this.watchShallow })
    this.$watch('refresh', this.onRefresh)
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.__resizeListener)
    this.__generalListeners.forEach(obj => this.$refs.container.removeAllListeners(obj.fullName))
    Plotly.purge(this.$refs.container)
  },
  methods: {
    onRefresh(newv, oldv) {
      if (newv !== oldv) {
        this.relayout({ autosize: this.autoResize });
      }
    },
    initEvents() {
      if (this.autoResize) {
        this.__resizeListener = debounce(this.newPlot, 200)
        window.addEventListener('resize', this.__resizeListener)
      }

      this.__generalListeners = events.map((eventName) => {
        return {
          fullName: 'plotly_' + eventName,
          handler: (...args) => {
            this.$emit.apply(this, [eventName].concat(args))
          }
        }
      })

      this.__generalListeners.forEach((obj) => {
        this.$refs.container.on(obj.fullName, obj.handler)
      })
    },
    ...methods,
    toImage(options) {
      var el = this.$refs.container
      var opts = defaults(options, {
        format: 'png',
        width: el.clientWidth,
        height: el.clientHeight
      })

      return Plotly.toImage(this.$refs.container, opts)
    },
    downloadImage(options) {
      var el = this.$refs.container
      var opts = defaults(options, {
        format: 'png',
        width: el.clientWidth,
        height: el.clientHeight,
        filename: (el.layout.title || 'plot') + ' - ' + new Date().toISOString()
      })

      return Plotly.downloadImage(this.$refs.container, opts)
    },
    plot() {
      return Plotly.plot(this.$refs.container, this.data, this.internalLayout, this.options)
    },
    newPlot() {
      return Plotly.newPlot(this.$refs.container, this.data, this.internalLayout, this.options).catch((err) => {
        console.log('[safe to ignore] failed during plotly\'s newPlot : (%s) %s', err.name, err.message);
      })
    }
  }
}
</script>
