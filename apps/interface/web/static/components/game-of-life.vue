<template>
  <div class="my-app" id="pew" style="text-align: center">
    <h1>Conway's Game of Life</h1>
    <h4>{{ active }} cells are alive | <i class="fa fa-circle" aria-hidden="true" v-bind:style="{ color: colour }"></i> {{ state }}</h4>
    <div id="info">
      <div class="data">Generation: {{ generation }}</div>
      <div class="data">Maximum number of alive cell: {{ maxAlive }}</div>
    </div>
    <canvas id="canvas"></canvas>
    <p>Press <kbd>SPACEBAR</kbd> to start and pause, <kbd>T</kbd> to advance the universe of one tick, <kbd>R</kbd> to reset.</p>
    <p>
      <b><span v-bind:class="{ selected: pattern == 'diehard' }">DIEHARD</span></b> <kbd>D</kbd> | 
      <b><span v-bind:class="{ selected: pattern == 'glider' }">GLIDER</span></b> <kbd>G</kbd> |
      <b><span v-bind:class="{ selected: pattern == 'tumbler' }">TUMBLER</span></b> <kbd>U</kbd>
      <b><span v-bind:class="{ selected: pattern == 'eight' }">FIGURE EIGHT</span></b> <kbd>E</kbd>
    </p>
  </div>
</template>

<script>
import {Socket} from 'phoenix'

export default {

  data () {
    return {
      canvas: null,
      context: null,
      scale: 10,
      interval: 100,
      simulating: false,
      active: 0,
      channel: null,
      generation: 0,
      maxAlive: 0,
      pattern: 'diehard'
    }
  },

  created () {
    window.addEventListener('keydown', (e) => {
      if ((e.keyCode || e.which) == 32) { // SPACEBAR
        if (this.active == 0) {
          this.reset() 
          this.maxAlive = 0
        } else {
          this.simulating = !this.simulating
          this.simulate()
        }
      } else if ((e.keyCode || e.which) == 82) { // R
        this.reset()
        this.maxAlive = 0
      } else if ((e.keyCode || e.which) == 84 && !this.simulating) { // T
        this.simulate()
      } else if ((e.keyCode || e.which) == 68 && !this.simulating) { // D
        this.pattern = 'diehard'
        this.reset()
        this.maxAlive = 0
      } else if ((e.keyCode || e.which) == 71 && !this.simulating) { // G
        this.pattern = 'glider'
        this.reset()
        this.maxAlive = 0
      } else if ((e.keyCode || e.which) == 85 && !this.simulating) { // U
        this.pattern = 'tumbler'
        this.reset()
        this.maxAlive = 0
      } else if ((e.keyCode || e.which) == 69 && !this.simulating) { // E
        this.pattern = 'eight'
        this.reset()
        this.maxAlive = 0
      }
    }, true)
  },

  mounted () {
    this.setupCanvas()
    this.setupSocket()
  },

  computed: {
    state () {
      return this.simulating ? this.state = 'RUNNING' : this.state = 'STOP'
    },

    colour () {
      return this.simulating ? this.colour = 'rgb(58, 184, 130)' : this.colour = 'rgb(218, 89, 97)'
    }
  },

  methods: {
    setupCanvas () {
      this.canvas = document.getElementById("canvas")
      this.context = this.canvas.getContext("2d")
      let ratio = this.getPixelRatio(this.context)
      this.canvas.width = document.getElementById("pew").clientWidth * ratio
      this.canvas.height = (document.getElementById("pew").clientWidth - 120) * ratio
      this.canvas.style.width = `${document.getElementById("pew").clientWidth}px`
      this.canvas.style.height = `${document.getElementById("pew").clientWidth - 120}px`
      this.context.scale(ratio, ratio)
      this.context.fillStyle = 'rgb(111, 168, 220)'
    },

    getPixelRatio (context) {
      var backingStore = context.backingStorePixelRatio ||
              context.webkitBackingStorePixelRatio ||
              context.mozBackingStorePixelRatio ||
              context.msBackingStorePixelRatio ||
              context.oBackingStorePixelRatio ||
              context.backingStorePixelRatio || 1

      return (window.devicePixelRatio || 1) / backingStore
    },

    render (positions) {
      this.context.clearRect(0, 0, this.canvas.width, this.canvas.height)
      positions.forEach(({x, y, n}) => {
          this.context.fillRect(x * this.scale, y * this.scale, this.scale, this.scale)
          if (n == 0) {
            this.context.fillStyle = 'rgb(32, 80, 129)'
          } else if (n == 1) {
            this.context.fillStyle = 'rgb(255, 217, 102)'
          } else if (n == 2) {
            this.context.fillStyle = 'rgb(218, 89, 97)'
          } else {
            this.context.fillStyle = 'rgb(111, 168, 220)'
          }
      })
      this.active = positions.length
      if (positions.length > this.maxAlive) {
        this.maxAlive = positions.length
      } 
      
    },

    setupSocket () {
      let socket = new Socket("/socket")
      socket.connect()

      this.channel = socket.channel("life", {});
      this.channel.join()
          .receive("ok", resp => {
            this.reset()
          })
          .receive("error", resp => console.error)
    },

    reset () {
      this.simulating = false
      this.channel.push("reset", {pattern: this.pattern})
      this.channel.on("reset", cells => {})
      this.generation = 0
      this.simulate()
    },

    simulate () {
      this.channel.on("tick", cells => {
        this.render(cells.positions)
        if (this.active == 0) {
          this.simulating = false
        }
      })

      setTimeout(function tick() {
          vm.$children[0].$data.channel.push("tick");
          vm.$children[0].$data.generation++
          if (vm.$children[0].$data.simulating) {
              setTimeout(tick, 100)
          }
      }, 100)
    }
  }
}
</script>

<style lang="sass">
#canvas {
  background-color: rgb(245, 245, 245);
  border-radius: 5pt;
}
#info {
  background-color: rgb(245, 245, 245);
  border-radius: 5pt;
  height: 30px;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  padding-left: 10px;
  padding-right: 10px;
  justify-content: center;
}

.data {
  width: 40%;
}

.selected {
  color: rgb(218, 89, 97);
}
</style>