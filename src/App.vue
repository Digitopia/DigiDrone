<template>
  <div class="wrapper">
    <svg
      @click="improv($event)"
      id="canvas"
      :viewBox="`0 0 ${width} ${height}`"
    >
      <!-- Improv Circles -->
      <circle
        v-for="circle in improvCircles"
        :key="circle.id"
        :cx="circle.x"
        :cy="circle.y"
        :r="circle.r"
        :id="circle.id"
        class="improv-circle"
      />

      <!-- curve path -->
      <path :d="curveStr" class="drone-area" @click.stop />

      <!-- Control knobs -->
      <g
        v-for="knob in knobs"
        :key="knob.id"
        :id="knob.id"
        class="knob"
        @click.stop
      >
        <circle :r="knobRadius" />
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
import { sample, random } from 'lodash'
import * as d3 from 'd3'
import { disableBodyScroll } from 'body-scroll-lock'
window.teoria = teoria
window.Tone = Tone
window.map = map

function map(value, start1, stop1, start2, stop2) {
  return ((value - start1) / (stop1 - start1)) * (stop2 - start2) + start2
}

// function mapWithFunction(value, start1, stop1, start2, stop2, fn) {
//   const inT = map(value, start1, stop1, 0, 1)
//   const outT = fn(inT)
//   return map(outT, 0, 1, start2, stop2)
// }

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
      improvCircles: [],
      improvNoteDuration: Tone.Time('4n'),
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
            { value: 'dim', label: 'o' },
            { value: 'min7b5', label: 'ø' },
            { value: 'm', label: 'm' },
            { value: 'min7', label: 'm7' },
            { value: 'dom7', label: '7' },
            { value: 'M', label: 'M' },
            { value: 'maj7', label: 'Δ7' },
            { value: 'aug', label: '+' },
          ],
        },
        {
          id: 'pulse',
          activeLabel: '3',
          activeValue: 3,
          options: [
            { value: '1n', label: '●' },
            { value: '2n', label: '●●' },
            { value: '4n', label: '●●●' },
            { value: '8n', label: '●●●●' },
            { value: '16n', label: '●●●●●' },
          ],
        },
        {
          id: 'volume',
          activeValue: -10,
          activeLabel: '&#xf027;',
          minValue: -30,
          maxValue: -3,
        },
      ],
      scales: {
        aug: 'wholetone',
        maj7: 'lydian',
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
    middleY() {
      return this.height / 2
    },

    halfThird() {
      return this.height / 2 / 3
    },

    minY() {
      return this.middleY - this.halfThird
    },

    maxY() {
      return this.middleY + this.halfThird
    },

    knobRadius() {
      const per = 0.05 * this.width
      const r = Math.max(25, Math.min(50, per))
      console.log({ r, per })
      return r
    },

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

    curveFunction() {
      return d3
        .area()
        .x(d => d.x)
        .y1(d => d.y)
        .y0(this.height)
        .curve(d3.curveCatmullRom.alpha(0.5))
    },

    droneNotes() {
      const notes = teoria
        .note(this.root)
        .chord(this.chord)
        .notes()
      notes.push(notes.shift().interval('P8'))
      notes.push(notes.shift().interval('P8'))
      notes.push(notes[0].interval('P8'))
      return notes
      // return notes.map(note => note.interval('P8'))
    },

    improvNotes() {
      if (this.scale === 'octatonic') {
        let octatonic = []
        octatonic.push(teoria.note(this.root))
        for (let i = 1; i < 8; i++) {
          const prev = octatonic[i - 1]
          const interval = i % 2 === 1 ? 'M2' : 'm2'
          const note = teoria.interval.from(prev, interval)
          octatonic.push(note)
        }
        return octatonic.map(note => note.interval('P8').interval('P8'))
      } else {
        const scale = teoria
          .note(this.root)
          .scale(this.scale)
          .interval('P8')
          .interval('P8')
          .interval('P8')
          .notes()
        return [
          ...scale,
          teoria
            .note(this.root)
            .interval('P8')
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
      this.loop.interval = this.pulse
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

    this.initSound()
    this.initDroneLoop()
  },

  mounted() {
    this.initKnobs()
    this.setRandomConfiguration()

    window.addEventListener('resize', () => this.resize())
    this.resize()

    disableBodyScroll(document.querySelectorAll('#canvas'))
  },

  methods: {
    initDroneLoop() {
      this.loop = new Tone.Loop(() => {
        // const r = random(
        //   0,
        //   this.knobs.find(knob => knob.id === 'pulse').options.length
        // )
        // console.log({ r, pulse: this.pulse })
        // if (r > this.pulse) return
        // if (Math.random() > 0.5) return

        // get a random note from droneNotes that is not the previous one
        if (!this.chosenNote) this.chosenNote = teoria.note(this.root)
        const n = sample(
          this.droneNotes.filter(
            note => note.scientific() !== this.chosenNote.scientific()
          )
        )
        this.chosenNote = n

        const dur = sample(['2n', '2n', '2n', '2n', '4n', '4n', '8n'])
        console.log({ note: n.scientific(), dur })

        this.droneSynth.triggerAttackRelease(
          n.scientific(),
          dur,
          Tone.now(),
          random(0.6, 1)
        )
      }, this.pulse)
      this.loop.start(0)
    },

    initSound() {
      Tone.Transport.bpm.value = 60

      // reverb
      const reverb = new Tone.Reverb()
      reverb.decay = 3.5
      reverb.preDelay = 0.01
      reverb.generate()

      // root synth
      // this.rootSynth = new Tone.Oscillator({
      //   type: 'sine',
      //   frequency: this.root,
      //   detune: 0,
      //   phase: 0,
      //   partials: [],
      //   partialCount: 0,
      // }).toMaster()

      this.rootSynth = new Tone.Synth({
        oscillator: {
          type: 'sine',
          frequency: this.root,
          detune: 0,
          phase: 0,
          partials: [],
          partialCount: 0,
        },
        envelope: {
          attack: 0.005,
          decay: 0.1,
          sustain: 0.3,
          release: 1,
        },
      }).toMaster()

      // this.rootSynth = new Tone.OmniOscillator(this.root, 'amsine').chain(
      //   Tone.Master
      // )
      // this.rootSynth.set({
      //   harmonicity: 0.5,
      //   oscillator: {
      //     type: 'sine',
      //   },
      //   envelope: {
      //     attack: 0.01,
      //     decay: 0.01,
      //     sustain: 1,
      //     release: 0.5,
      //   },
      //   modulation: {
      //     type: 'sine',
      //   },
      //   modulationEnvelope: {
      //     attack: 0.5,
      //     decay: 0,
      //     sustain: 1,
      //     release: 0.5,
      //   },
      //   // portamento: Tone.Time('8n'),
      // })

      // drone synth
      this.droneSynth = new Tone.PolySynth(6, Tone.Synth).chain(
        // reverb,
        Tone.Master
      )
      this.droneSynth.set({
        oscillator: {
          type: 'sine',
        },
        envelope: {
          attack: 0.25,
          decay: 0.5,
          sustain: 0.1,
          release: 0.1,
        },
      })

      // improv synth
      this.improvSynth = new Tone.PolySynth(6, Tone.Synth).chain(
        reverb,
        Tone.Master
      )
      this.improvSynth.set({
        oscillator: {
          type: 'sine',
        },
        envelope: {
          attack: 0.25,
          decay: 0.5,
          sustain: 0.1,
          release: 0.1,
        },
      })

      window.rootSynth = this.rootSynth
      window.droneSynth = this.droneSynth
      window.improvSynth = this.improvSynth
    },

    setRandomConfiguration() {
      // generate random configuration
      this.knobs.forEach(knob => {
        if (knob.id === 'volume') return
        const r = Math.floor(Math.random() * knob.options.length)
        let y = map(r, 0, knob.options.length - 1, this.minY, this.maxY)
        this.setKnobPosition(knob, knob.x, y)
      })

      const volumeKnob = this.knobs.find(knob => knob.id === 'volume')
      this.setKnobPosition(
        volumeKnob,
        volumeKnob.x,
        this.middleY - this.halfThird / 2
      )
    },

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
      const amp = Math.max(0, map(y, this.maxY, 0, 0, 1))
      const idx = Math.floor(map(x, 0, this.width, 0, this.improvNotes.length))
      const n = this.improvNotes[idx]

      console.log('improv:', n.scientific(), '@', amp)

      this.improvSynth.triggerAttackRelease(
        n.scientific(),
        this.improvNoteDuration,
        Tone.now(),
        amp
      )

      // animate circle
      Vue.nextTick(() => {
        TweenLite.to(`#${id}`, this.improvNoteDuration.toSeconds(), {
          attr: {
            r: 50,
          },
          opacity: 0,
          ease: Linear.easeNone,
          onComplete: () => {
            console.log('finished', id)
            // const idx = this.improvCircles.find(circle => circle.id === id)
            // this.improvCircles.splice(idx, 1)
            this.improvCircles = this.improvCircles.filter(
              circle => circle.id !== id
            )
          },
        })
      })
    },

    resize() {
      const vh = window.innerHeight * 0.01
      document.documentElement.style.setProperty('--vh', `${vh}px`)

      this.previousWidth = this.width
      this.previousHeight = this.height
      this.width = this.$el.clientWidth
      this.height = this.$el.clientHeight
      this.updateKnobsPositions()
    },

    initKnobs() {
      this.knobs.forEach(knob => {
        const id = `#${knob.id}`
        const that = this
        knob.draggable = Draggable.create(id, {
          type: 'y',
          cursor: 'pointer',
          onDrag() {
            that.setKnobPosition(knob, this.x, this.y)
          },
        })[0]
      })
    },

    updateKnobsPositions() {
      this.knobs.forEach((knob, idx) => {
        const div = this.width / this.knobs.length
        const x = div / 2 + div * idx
        const y = map(knob.y, 0, this.previousHeight, 0, this.height)
        this.setKnobPosition(knob, x, y)
        knob.draggable.applyBounds({ minY: this.minY, maxY: this.maxY })
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

      if (!this.playing) {
        Tone.Master.volume.rampTo(-120, 0.5)
        Tone.Transport.stop()
        this.loop.stop()
        return
      }

      Tone.Master.volume.value = -Infinity
      Tone.Master.volume.rampTo(this.volume, 0.5)
      // this.rootSynth.start()
      this.rootSynth.triggerAttack(this.root)
      Tone.Transport.start()
      this.loop.start()
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

html {
  // Small devices (landscape phones, 576px and up)
  @media (min-width: 576px) {
    text {
      font-size: 13px;
    }
  }

  // Medium devices (tablets, 768px and up)
  @media (min-width: 768px) {
    text {
      font-size: 14px;
    }
  }

  // Large devices (desktops, 992px and up)
  @media (min-width: 992px) {
    text {
      font-size: 15px;
    }
  }

  // Extra large devices (large desktops, 1200px and up)
  @media (min-width: 1200px) {
    text {
      font-size: 16px;
    }
  }
}

#app {
  font-family: 'Raleway', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.wrapper {
  position: absolute;
  width: 100%;
  height: calc(var(--vh, 1vh) * 100);
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
    font-size: 1.2em;
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

* {
  -webkit-tap-highlight-color: transparent !important;
  &:focus {
    outline: none !important;
  }
}
</style>
