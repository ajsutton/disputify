<template>
  <v-container class="" fluid>
    
    <v-row>
      <v-col cols="12">
        <v-card >
          <v-card-title>Disputify</v-card-title>
          <v-card-actions>
            <v-slider label="Game Progress" v-model="maxClaim" :min="1" :max="allClaims.length" show-ticks="always" thumb-label="always" :step="1"></v-slider>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>
    <v-row>
      <v-col cols="12">
        <div ref="svg" id="svg"></div>
      </v-col>
    </v-row>
    <v-row>
      <v-col cols="12">
        <pre style="white-space: pre-wrap;">{{ graph }}</pre>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import { instance } from "@viz-js/viz";

/*
	ParentIndex uint32
	Countered   bool
	Claim       [32]byte
	Position    *big.Int
	Clock       *big.Int
*/

const maxDepth = 4
const maxNode = 31
const firstLeafNode = Math.pow(2, maxDepth)
const baseNodeData = [
  { position: 1, traceIndex: 15, attack: 2,  },
  { position: 2, traceIndex: 7, attack: 4, defend: 6, },
  { position: 3, },
  { position: 4, traceIndex: 3, attack: 8, defend: 10, },
  { position: 5 },
  { position: 6, traceIndex: 11, attack: 12, defend: 14, },
  { position: 7 },
  { position: 8, traceIndex: 1, attack: 16, defend: 18, },
  { position: 9 },
  { position: 10, traceIndex: 5, attack: 20, defend: 22, },
  { position: 11 },
  { position: 12, traceIndex: 9, attack: 24, defend: 26, },
  { position: 13 },
  { position: 14, traceIndex: 13, attack: 28, defend: 30, },
  { position: 15, },
  { position: 16, leaf: true, traceIndex: 0},
  { position: 17, leaf: true, },
  { position: 18, leaf: true, traceIndex: 2},
  { position: 19, leaf: true, },
  { position: 20, leaf: true, traceIndex: 4},
  { position: 21, leaf: true, },
  { position: 22, leaf: true, traceIndex: 6},
  { position: 23, leaf: true, },
  { position: 24, leaf: true, traceIndex: 8},
  { position: 25, leaf: true, },
  { position: 26, leaf: true, traceIndex: 10},
  { position: 27, leaf: true, },
  { position: 28, leaf: true, traceIndex: 12},
  { position: 29, leaf: true, },
  { position: 30, leaf: true, traceIndex: 14},
  { position: 31, leaf: true, },
]
export default {
    data: () => ({
      allClaims: [
        {ParentIndex: 0, Countered: true, Claim: "0xaaaaaa", Position: 1, Clock: 1500},
        {ParentIndex: 0, Countered: true, Claim: "0xbbbbbb", Position: 2, Clock: 1500},
        {ParentIndex: 1, Countered: true, Claim: "0xcccccc", Position: 6, Clock: 1500},
        {ParentIndex: 2, Countered: true, Claim: "0xdddddd", Position: 12, Clock: 1500},
        {ParentIndex: 3, Countered: false, Claim: "0xeeeeee", Position: 24, Clock: 1500},

        {ParentIndex: 0, Countered: true, Claim: "0xffffff", Position: 2, Clock: 1500},
        {ParentIndex: 5, Countered: false, Claim: "0xgggggg", Position: 4, Clock: 1500},
      ],
      maxClaim: 1,
      unusedConnections: `
8 -> 17[color=lightgrey]
4 -> 9[color=lightgrey]
9 -> 18[color=lightgrey]
9 -> 19[color=lightgrey]
2 -> 5[color=lightgrey]
5 -> 10[color=lightgrey]
10 -> 21[color=lightgrey]
5 -> 11[color=lightgrey]
11 -> 22[color=lightgrey]
11 -> 23[color=lightgrey]
1 -> 3[color=lightgrey]
3 -> 6[color=lightgrey]
12 -> 25[color=lightgrey]
6 -> 13[color=lightgrey]
13 -> 26[color=lightgrey]
13 -> 27[color=lightgrey]
3 -> 7[color=lightgrey]
7 -> 14[color=lightgrey]
14 -> 29[color=lightgrey]
7 -> 15[color=lightgrey]
15 -> 30[color=lightgrey]
15 -> 31[color=lightgrey]
`,
    }),
    computed: {
      claims ()  {
        return this.allClaims.slice(0, this.maxClaim)
      },
      graph() {
        return `
          digraph {
            // textNodes
            ${this.textNodes}

            // nodes
            ${this.nodes}

            // unusedConnections
            ${this.unusedConnections}

            // connections
            ${this.connections}
          }
        `
      },
      nodeData() {
        // Create copy of base node data and pre-populate all nodes
        const nodes = baseNodeData.map(data => Object.assign({}, data, {
          idx: data.position - 1,
          label: data.position,
          unused: data.traceIndex === undefined,
        }))
        nodes.forEach(node => {
          node.claims = []
          if (node.attack !== undefined) {
            nodes[node.attack - 1].accessedFrom = node.position
            nodes[node.attack - 1].accessType = 'Attack'
          }
          if (node.defend !== undefined) {
            nodes[node.defend - 1].accessedFrom = node.position
            nodes[node.defend - 1].accessType = 'Defend'
          }
        })
        this.claims.forEach(claim => {
          const node = nodes[claim.Position - 1]
          node.active = true
          node.countered = claim.Countered
          node.claims.push({value: claim.Claim, countered: claim.Countered})
        })
        return nodes
      },
      connections() {
        return this.nodeData.map(data => {
          if (data.accessedFrom !== undefined) {
            const isAttack = this.isAttack(data)
            var color = this.color(data)
            return `${data.accessedFrom}->${data.position}` + 
                    `[constraint=${isAttack}]` + 
                    `[color="${color}"][fontcolor="${color}"]` +
                    `[label=${data.active ? data.accessType : '""'}]`
          } else {
            return ""
          }
        }).join('\n')
      },
      nodes() {
        return this.nodeData.map(data => {
          var label = data.label
          if (data.active) {
            label += '\n' + data.claims.map(this.renderClaim).join('\n')
          }
          var node = `${data.position}[label="${label}"]`
          if (data.unused) {
            node += '[color=lightgrey][fontcolor=lightgrey]'
          } else if (data.active) {
            node += `[color="${this.color(data)}"][fontcolor="${this.color(data)}"]`
          }
          return node
        }).join('\n')
      },
      unusedNodes() {
        return baseNodeData
          .filter(data => data.unused)
          .map(data => `${data.position}[color=lightgrey][fontcolor=lightgrey]`)
          .join('\n')
      },

      textNodes() {
        return baseNodeData
          .filter(data => data.leaf)
          .map(data => `${data.position}->t${data.position - firstLeafNode}[style=dotted][arrowhead=none]` +
          `t${data.position - firstLeafNode}[shape=square][label="${data.position - firstLeafNode}"]`)
          .join('\n')
      }
    },
    mounted() {
      this.render()
    },
    watch: {
      async graph() {
        await this.render()
      }
    },

    methods: {
      isAttack(node) {
        return node.accessType === 'Attack'
      },
      color(node) {
          if (node.active) {
            return this.isAttack(node) ? 'red' : 'blue'
          } else {
            return this.isAttack(node) ? '#FF000022' : '#0000FF22'
          }
      },
      renderClaim(claim) {
        var value = claim.value
        if (value.startsWith('0x')) {
          value = value.substring('0x'.length)
        }
        if (value.length > 4) {
          value = value.substring(0,2) + '…' + value.substring(value.length - 2)
        }
        if (claim.countered) {
          value = '⚔️ ' + value
        } else {
          value = '👑 ' + value
        }
        return value
      },
      async render() {
        const viz = await instance()
        
        const el = document.getElementById('svg')
        while (el.lastChild) {
          el.removeChild(el.lastChild)
        }
        el.appendChild(viz.renderSVGElement(this.graph))
      }
    }
  }
</script>
