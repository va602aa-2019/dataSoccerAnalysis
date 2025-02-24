<template>
  <div>
    <b-row>
      <div id="app">
        <div class="filter-info">
          <ul v-if="tempNames" class="name-list">
            <li
              v-for="(item, index) in tempNames"
              :key="index"
              :style="{ color: colorScale(item.group) }"
              @mouseover="activeItem = item;"
              @mouseleave="activeItem = null;"
              @click="pushCounterElement(item.id)"
            >
              {{ decodeUnicodeName(item.name) }}
            </li>
          </ul>
        </div>

        <div class="parallel-container">
          <!-- SVG Entry -->
          <parallel-coord
            class="coords"
            v-if="dataset"
            :dataset="dataset"
            :column-data="columnData"
            :filtered.sync="allFiltered"
            :filtered-sample.sync="filteredSample"
            :render-count="$data.$sampleAmount"
            :active-item="activeItem"
            :color="getColor"
            :line-style="getLineStyle"
          />
        </div>
      </div>
    </b-row>
    <b-row>
      <b-col style="text-align: center;">
        <h3>Data values of {{decodeUnicodeName(this.nameOfThePlayer)}}</h3>
      </b-col>
    </b-row>
    <b-row style="margin-left: 22%;">
      <counter :measure="'Pass'" :value="this.valuePass"></counter>
      <counter :measure="'Head Pass'" :value="this.valueHeadPass"></counter>
      <counter :measure="'Air Duel'" :value="this.valueAirDuel"></counter>
      <counter :measure="'Foul'" :value="this.valueFoul"></counter>
      <counter :measure="'Dribbling'" :value="this.valueDribbling"></counter>
      <counter :measure="'Corner'" :value="this.valueCorner"></counter>
      <counter :measure="'Cross'" :value="this.valueCross"></counter>
      <counter :measure="'Kick'" :value="this.valueKick"></counter>
      <counter :measure="'Defense'" :value="this.valueDefense"></counter>
      <counter :measure="'Acceleration'" :value="this.valueAcceleration"></counter>
      <counter :measure="'Goal'" :value="this.valueGoal"></counter>
      <counter :measure="'Score'" :value="this.valueScore"></counter>
    </b-row>
    <b-row>
      <b-col>
        <bubble :playerSelected ="this.tempNames2"></bubble>
      </b-col>
    </b-row>
    <b-row>
      <b-col>
        <h3>Visualization Description</h3>
        <div class="description" style="font-size: 18px">
          <p>This visualization, based on measure selected, has the scope to help the D.S. manager to check what are the football player that have a certain skills.
            In particular this is for market reasons, because every D.S. has the possibility to check what are the players that the team need.</p>
          <p>In the first part of the visualization there is the table in which we have all the possible measure over which we can filter players,
            with a list that let the viewer see who are the player in that specific measures's ranges. Clicking over a name inside the list will render on card on bottom the exact values
            of the player in all the measures.</p>
          <p>In the second part of visualization there is also the bubble chart which display all players in different size of bubbles based on player score (precomputed).
            This has the aim to improve in a visual way the research of the best player inside all the other obtained by the list.
            In the visualization there are 4 kinds of colours based on role: <p style="color:#1E8361">GoalKeeper</p> <p style="color:#BF5F00">Defense</p> <p style="color:#736AB3"> Midfielder</p> <p style="color:#D42884">Forward</p>
          <p>The data are precomputed inside Anaconda, for each player has been extracted all the metrics available
            inside the visualization, then an interpolation has been applied in order to have all the metrics between values 0 and 100.</p>

        </div>
      </b-col>
    </b-row>
  </div>
</template>

<script>
import ParallelCoord from './components/ParallelCoord/index.vue'
import bubble from './components/bubble'
import counter from './components/counter'
import * as d3 from 'd3'
import _ from 'lodash'

export default {
  name: 'VisualizzazioneGiocatori',
  components: {
    ParallelCoord,
    bubble,
    counter
  },
  filters: {
    asString (object) {
      return Object.entries(object)
        .map(v => {
          return `${v[0]}: ${v[1]}`
        })
        .join('\n')
    }
  },
  data () {
    return {
      groupNames: [],
      nameOfThePlayer: ' ',
      tempNames2: [],
      namesForCounter: [],
      $sampleAmount: 100,
      dataset: null,
      ignoredDimensions: ['name', 'id', 'group', 'score'],
      allFiltered: null,
      filteredSample: null,
      activeItem: null,
      valueGoal: 0,
      valuePass: 0,
      valueHeadPass: 0,
      valueAirDuel: 0,
      valueFoul: 0,
      valueDribbling: 0,
      valueCorner: 0,
      valueCross: 0,
      valueKick: 0,
      valueDefense: 0,
      valueAcceleration: 0,
      valueScore: 0,
      testScale: d3
        .scaleLinear()
        .domain([1, 100])
        .range([50, 800])
    }
  },
  mounted () {
    // window.scaler = this.testScale
  },
  beforeCreate () {
    d3.csv(
      './static/mod.csv'
    ).then(d => {
      // filter and set numbers
      const groups = []
      const cleaned = d.reduce((prev, cur) => {
        // return entries forced to numbers
        const entries = Object.entries(cur).map(v => {
          const f = parseFloat(v[1])
          const isNumber = !isNaN(f)
          return [v[0], isNumber ? f : v[1]]
        })

        const hasAllValues = entries.every(v => {
          return v[1] != null
        })

        if (hasAllValues) {
          if (!groups.includes(cur.group)) groups.push(cur.group)
          prev.push(
            entries.reduce((p, curv) => {
              const f = parseFloat(curv[1])
              if (!isNaN(f)) {
                p[curv[0]] = f
              } else {
                p[curv[0]] = curv[1]
              }
              return p
            }, {})
          )
        }

        return prev
      }, [])

      this.groupNames = Object.freeze(groups)
      this.dataset = cleaned
    })
  },
  computed: {
    /**
     * An array of dimension key names
     */
    columnKeys () {
      if (this.dataset) {
        return Object.keys(this.dataset[0]).filter(v => {
          return !this.ignoredDimensions.includes(v)
        })
      }
    },

    colorScale () {
      return d3
        .scaleOrdinal()
        .domain(this.groupNames)
        .range([...d3.schemeDark2, '#fff'])
    },

    /**
     * Creates columns to pass to the parallel coord
     */
    columnData () {
      if (this.columnKeys) {
        return this.columnKeys.map(v => {
          return {
            name: v
          }
        })
      }
    },

    tempNames () {
      const v = _.sampleSize(this.filteredSample, 100)
      v.sort((a, b) => {
        if (a.group < b.group) {
          return -1
        } else {
          return 1
        }
      })
      // manage data for bubble chart
      // eslint-disable-next-line vue/no-side-effects-in-computed-properties
      this.tempNames2 = v
      // eslint-disable-next-line vue/no-side-effects-in-computed-properties
      this.namesForCounter = v
      if (this.tempNames2.length !== 0) {
        this.prepareListBubbleChart(this.tempNames2)
      }
      return v
    }
  },
  methods: {
    getColor (item) {
      return this.colorScale(item.group)
    },
    getLineStyle (item) {
      const sameGroup = this.activeItem && this.activeItem.group === item.group
      return {
        transition: 'opacity 300ms ease',
        opacity: sameGroup ? 1 : this.activeItem ? 0.15 : 0.55,
        strokeWidth: sameGroup ? 5 : 2
      }
    },
    decodeUnicodeName: (str) => str
      .replace(/\\u(....)/g, (match, p1) => String.fromCharCode(parseInt(p1, 16)))
      .replace(/\\(\d{3})/g, (match, p1) => String.fromCharCode(parseInt(p1, 8))),
    // bubble chart preparation
    prepareListBubbleChart (array) {
      let selectElements = []
      const colors = {
        Goalkeeper: '#1E8361',
        Defender: '#BF5F00',
        Midfielder: '#736AB3',
        Forward: '#D42884'
      }
      array.forEach((element, index, array) => {
        selectElements.push({
          name: this.decodeUnicodeName(element.name),
          amount: element.score,
          color: colors[element.group]})
        // var score = 0
      })
      this.tempNames2 = selectElements
    },
    pushCounterElement (id) {
      let vm = this
      console.log(id)
      this.namesForCounter.forEach(function (player) {
        if (id === player.id) {
          console.log(player)
          vm.valueAcceleration = player.acceleration
          vm.valueAirDuel = player.airduel
          vm.valueCorner = player.corner
          vm.valueCross = player.cross
          vm.valueDefense = player.defense
          vm.valueDribbling = player.dribbling
          vm.valueFoul = player.foul
          vm.valueGoal = player.goal
          vm.valueHeadPass = player.headpass
          vm.valueKick = player.kick
          vm.valuePass = player.pass
          vm.valueScore = player.score
          vm.nameOfThePlayer = player.name
          console.log(player.pass)
        }
      })
      console.log(this['valuePass'])
    }
  }
}
</script>

<style scoped>
  .parallel-container {
    padding-top: 50px;
    padding-left: 25px;

  @media screen and (max-width: 1000px) {
    padding-bottom: 20px;
  }
  }

  .coords {
    width: 100%;
    height: 100%;
    background-color: #b4bdc2;
    overflow: visible;

  /deep/ text {
    user-select: none;
  }
  }

  rect.overlay {
  // fill: rgba(255, 255, 255, 0.14);
  }

  .filter-info {
    max-height: 100%;
    overflow: auto;
    color: black;
    padding: 10px;
    will-change: contents;
    background-color: white;
    border: 2px solid #17a2b8;

  @media screen and (max-width: 1000px) {
    order: 1;
  }
  }

  .name-list {
    list-style: none;
    padding: 0;
    margin: 0;
    font-size: 10px;

  > li {
    cursor: pointer;
    padding-bottom: 10px;

    &:hover,
    &:active,
    &:focus {
       color: #17a2b8;
     }
    }
  }

  ::-webkit-scrollbar {
    width: 4px;
  }

  ::-webkit-scrollbar-track {
    background-color: rgba(#fff, 0.3);
  }

  ::-webkit-scrollbar-thumb {
    background-color: #17a2b8;
  }
  .description{
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
  }
</style>
