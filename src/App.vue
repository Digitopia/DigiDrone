<template>
  <div class="wrapper">
    <svg
      @click="improv($event)"
      id="canvas"
      :viewBox="`0 0 ${width} ${height}`"
    >
      <!-- Improv Circles -->
      <circle
        v-for="(circle, idx) in improvCircles"
        :key="idx"
        :cx="circle.x"
        :cy="circle.y"
        :r="circle.r"
        :id="circle.id"
        class="improv-circle"
      />

      <!-- curve path -->
      <path :d="curveStr" class="drone-area" />

      <!-- Control knobs -->
      <g
        v-for="knob in knobs"
        :key="knob.id"
        :id="knob.id"
        class="knob"
        @click.stop
      >
        <circle r="50" />
        <text :class="{ icons: knob.id === 'volume' }" dy=".3em">
          {{ knob.id === 'volume' ? '' : knob.activeLabel }}
        </text>
      </g>
    </svg>
    <div class="controls">
      <i
        class="fa play"
        :class="playing ? 'fa-stop ' : 'fa-play'"
        @click="toggleDrone"
      />
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import Tone from 'tone'
import teoria from 'teoria'
import Draggable from 'gsap/Draggable'
import { TweenLite, Linear } from 'gsap'
import sample from 'lodash.sample'
import random from 'lodash.random'
import * as d3 from 'd3'
window.teoria = teoria
window.Tone = Tone
window.map = map

function map(value, start1, stop1, start2, stop2) {
  return ((value - start1) / (stop1 - start1)) * (stop2 - start2) + start2
}

function mapWithFunction(value, start1, stop1, start2, stop2, fn) {
  const inT = map(value, start1, stop1, 0, 1)
  const outT = fn(inT)
  return map(outT, 0, 1, start2, stop2)
}

function printNotes(label, notes) {
  console.log(label, notes.map(note => note.scientific()))
}

export default {
  name: 'app',
  data() {
    return {
      width: 1600,
      height: 900,
      playing: false,
      lastImprovCircleId: 0,
      curvePoints: [],
      curveFunction: null,
      improvCircles: [],
      knobs: [
        {
          id: 'root',
          value: 'C',
          activeLabel: 'Dó',
          activeValue: 'C2',
          options: [
            { value: 'C2', label: 'Dó' },
            { value: 'C#2', label: 'Dó♯' },
            { value: 'D2', label: 'Ré' },
            { value: 'Eb2', label: 'Mi♭' },
            { value: 'E2', label: 'Mi' },
            { value: 'F2', label: 'Fá' },
            { value: 'F#2', label: 'Fá♯' },
            { value: 'G2', label: 'Sol' },
            { value: 'G#2', label: 'Sol♯' },
            { value: 'A2', label: 'Lá' },
            { value: 'Bb2', label: 'Si♭' },
            { value: 'B2', label: 'Si' },
          ],
        },
        {
          id: 'chord',
          activeLabel: 'aug',
          activeValue: 'aug',
          options: [
            { value: 'dim', label: 'dim' },
            { value: 'min7b5', label: 'min7b5' },
            { value: 'M', label: 'M' },
            { value: 'm', label: 'm' },
            { value: 'min7', label: 'min7' },
            { value: 'dom7', label: 'dom7' },
            { value: 'maj7', label: 'maj7' },
            { value: 'aug', label: 'aug' },
          ],
        },
        {
          id: 'pulse',
          activeLabel: '3',
          activeValue: '3',
          options: [
            { value: '1', label: '1' },
            { value: '2', label: '2' },
            { value: '3', label: '3' },
            { value: '4', label: '4' },
            { value: '5', label: '5' },
          ],
        },
        {
          id: 'volume',
          activeValue: -10,
          activeLabel: '&#xf027;',
          minValue: -60,
          maxValue: -3,
        },
      ],
      scales: {
        aug: 'wholetone',
        maj7: 'major',
        dom7: 'mixolydian',
        M: 'major',
        min7: 'dorian',
        m: 'aeolian',
        min7b5: 'locrian',
        dim: 'octatonic',
      },
    }
  },
  computed: {
    root() {
      return this.knobs.find(knob => knob.id === 'root').activeValue
    },

    chord() {
      return this.knobs.find(knob => knob.id === 'chord').activeValue
    },

    pulse() {
      return this.knobs.find(knob => knob.id === 'pulse').activeValue
    },

    volume() {
      return this.knobs.find(knob => knob.id === 'volume').activeValue
    },

    scale() {
      return this.scales[this.chord]
    },

    curveStr() {
      if (!this.curveFunction) return
      return this.curveFunction(this.curvePoints)
    },

    droneNotes() {
      const notes = teoria
        .note(this.root)
        .chord(this.chord)
        .notes()
      notes.push(notes.shift().interval('P8'))
      notes.push(notes.shift().interval('P8'))
      notes.push(notes[0].interval('P8'))
      return notes.map(note => note.interval('P8'))
    },

    improvNotes() {
      if (this.scale === 'octatonic') {
        let octatonic = []
        octatonic.push(teoria.note(this.root))
        for (let i = 1; i < 8; i++) {
          const prev = octatonic[i - 1]
          const interval = i % 2 === 1 ? 'M2' : 'm2'
          octatonic.push(teoria.interval.from(prev, interval))
        }
        return octatonic
      } else {
        const scale = teoria
          .note(this.root)
          .scale(this.scale)
          .interval('P8')
          .interval('P8')
          .notes()
        return [
          ...scale,
          teoria
            .note(this.root)
            .interval('P8')
            .interval('P8')
            .interval('P8'),
        ]
      }
    },
  },

  watch: {
    chord() {
      console.log('chord:', this.chord)
      printNotes('improvNotes:', this.improvNotes)
      printNotes('droneNotes:', this.droneNotes)
    },

    root() {
      console.log('root:', this.root)
      printNotes('improvNotes:', this.improvNotes)
      printNotes('droneNotes:', this.droneNotes)
      this.rootSynth.frequency.value = this.root
    },

    pulse() {
      console.log('pulse:', this.pulse)
    },

    volume() {
      console.log('vol:', this.volume)
      Tone.Master.volume.linearRampToValueAtTime(this.volume, Tone.now() + 0.01)
    },
  },

  created() {
    document.addEventListener('keypress', evt => {
      console.log(evt)
      if (evt.key === ' ') this.toggleDrone()
    })

    this.rootSynth = new Tone.Oscillator({
      type: 'sine',
      frequency: this.root,
      detune: 0,
      phase: 0,
      partials: [],
      partialCount: 0,
    }).toMaster()

    this.droneSynth = new Tone.PolySynth(6, Tone.Synth).toMaster()
    this.improvSynth = new Tone.PolySynth(6, Tone.Synth).toMaster()

    window.rootSynth = this.rootSynth
    window.droneSynth = this.droneSynth
    window.improvSynth = this.improvSynth
  },

  mounted() {
    document.addEventListener('resize', () => this.resize())
    this.resize()

    this.initKnobs()

    this.curveFunction = d3
      .area()
      .x(d => d.x)
      .y1(d => d.y)
      .y0(this.height)
      .curve(d3.curveCatmullRom.alpha(0.5))

    // generate random configuration
    this.knobs.forEach(knob => {
      if (knob.id === 'volume') return
      const r = Math.floor(Math.random() * knob.options.length)
      let y = map(r, 0, knob.options.length - 1, this.minY, this.maxY)
      this.setKnobPosition(knob, knob.x, y)
    })

    const volumeKnob = this.knobs.find(knob => knob.id === 'volume')
    this.setKnobPosition(volumeKnob, volumeKnob.x, volumeKnob.y)
  },

  methods: {
    improv(evt) {
      const id = `improv-circle-${++this.lastImprovCircleId}`

      // add circle
      this.improvCircles.push({
        x: evt.offsetX,
        y: evt.offsetY,
        r: 10,
        id,
      })

      // determine which note it is
      const { offsetX: x, offsetY: y } = evt
      const amp = Math.max(
        0,
        mapWithFunction(y, this.maxY, 0, 0, 1, x => x * x)
      )
      const idx = Math.floor(map(x, 0, this.width, 0, this.improvNotes.length))
      const n = this.improvNotes[idx]

      console.log('improv:', n.scientific(), '@', amp)

      this.improvSynth.triggerAttackRelease(
        n.scientific(),
        '1n',
        Tone.now(),
        amp
      )

      // animate circle
      Vue.nextTick(() => {
        TweenLite.to(`#${id}`, 2, {
          attr: {
            r: 50,
          },
          opacity: 0,
          ease: Linear.easeNone,
          onComplete: () => {
            console.log('finished', id)
            this.improvCircles = this.improvCircles.filter(
              circle => circle.id !== id
            )
          },
        })
      })
    },

    resize() {
      this.width = this.$el.clientWidth
      this.height = this.$el.clientHeight
    },

    initKnobs() {
      this.knobs.forEach((knob, idx) => {
        const div = this.width / this.knobs.length
        const x = div / 2 + div * idx
        const y = this.height / 2
        const id = `#${knob.id}`

        TweenLite.set(id, { x, y })
        knob.x = x
        knob.y = y

        const middleY = this.height / 2
        const halfThird = this.height / 2 / 3
        this.minY = middleY - halfThird
        this.maxY = middleY + halfThird

        const that = this

        knob.draggable = Draggable.create(id, {
          type: 'y',
          bounds: { minY: this.minY, maxY: this.maxY },
          cursor: 'pointer',
          onDrag() {
            that.setKnobPosition(knob, this.x, this.y)
          },
        })[0]
      })
    },
    setKnobPosition(knob, x, y) {
      TweenLite.set(`#${knob.id}`, { x, y })
      knob.x = x
      knob.y = y
      knob.draggable.update()

      if (knob.id === 'volume') {
        let vol = map(y, this.maxY, this.minY, knob.minValue, knob.maxValue)
        vol = vol <= knob.minValue + 0.5 ? -Infinity : vol
        console.log(vol)
        knob.activeValue = vol
      } else {
        const idx = Math.round(
          map(y, this.maxY, this.minY, 0, knob.options.length - 1)
        )
        knob.activeValue = knob.options[idx].value
        knob.activeLabel = knob.options[idx].label
      }
      this.updateCurvePoints()
    },

    updateCurvePoints() {
      const points = this.knobs.map(knob => {
        return { x: knob.x, y: knob.y }
      })
      // add left margin at middle height to curve too
      points.unshift({ x: 0, y: this.height / 2 })

      // add right margin at middle height to curve too
      points.push({ x: this.width, y: this.height / 2 })

      this.curvePoints = points
    },

    toggleDrone() {
      this.playing = !this.playing

      if (this.rootSynth.state === 'started') {
        this.rootSynth.stop()
        window.clearInterval(this.interval)
        return
      }

      this.rootSynth.start()
      this.chosenNote = teoria.note(this.root)

      this.interval = window.setInterval(() => {
        const r = Math.random()
        if (r > this.pulse) return
        const n = sample(
          this.droneNotes.filter(
            note => note.scientific() !== this.chosenNote.scientific()
          )
        )
        this.chosenNote = n
        const dur = Math.random() * 2 + 0.8
        console.log('playing', n.scientific(), 'for', dur.toFixed(2), 'seconds')
        this.droneSynth.triggerAttackRelease(
          n.scientific(),
          dur,
          Tone.now,
          random(0.6, 1)
        )
      }, 1000)
    },
  },
}
</script>

<style lang="scss">
:root {
  --improv-bg: teal;
  --drone-bg: mediumseagreen;
  --text-color: #707070;
}
html,
body {
  margin: 0;
  background: teal;
}
#app {
  font-family: 'Raleway', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.wrapper {
  position: absolute;
  width: 100%;
  height: 100vh;
}
#canvas {
  position: absolute;
}
.knob {
  circle {
    fill: white;
  }
  text {
    text-anchor: middle;
    fill: var(--text-color);
    font-family: 'Raleway', sans-serif;
    font-size: 20px;
  }
}
.icons {
  font-family: 'FontAwesome' !important;
}
.improv-circle {
  fill: #ffffff;
  opacity: 0.45;
}
.drone-area {
  fill: var(--drone-bg);
}
.controls {
  position: absolute;
  bottom: 10px;
  margin: 0 auto;
  width: 100%;
  text-align: center;
  font-size: 45px;
  .play {
    color: white;
    &:hover {
      cursor: pointer;
    }
  }
}

// .overlay {
//   position: absolute;
//   bottom: 0;
//   margin: 0 auto;
//   width: 100%;
//   text-align: center;
//   margin-bottom: 30px;
//   color: white;
//   &.i.fas {
//     display: inline-block;
//     border-radius: 50%;
//     box-shadow: 0 0 2px #888;
//     padding: 0.5em 0.6em;
//   }
// }
</style>
