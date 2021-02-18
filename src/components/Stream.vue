<template>
  <div class="stream">
    <div class="alert alert-warning text-center mb-1 font-weight-bold" v-if="!connected">
      Connecting
    </div>
    <div class="alert alert-success text-center mb-1 font-weight-bold" v-if="connected">
      Connected
    </div>
    <div class="alert alert-danger text-center mb-1 font-weight-bold" v-if="error">
      <b>Error:</b>
      <pre>{{ error }}</pre>
    </div>
    <div v-for="ledger, index in ledgers" v-bind:key="ledger" class="card mt-3">
      <h1 class="h4 card-header text-white py-1" :class="{
        'bg-primary': index === 0,
        'bg-secondary': index !== 0
      }"><code class="text-white">{{ ledger }}</code></h1>
      <div class="card-body px-0 pt-1 pb-0">
        <ul class="node-list mt-2 pb-0 mb-0">
          <li v-for="node in nodesPerLedger[ledger]" v-bind:key="node.master_key" class="shadow-sm" :class="{
            'bg-primary': index === 0,
            'bg-secondary': index !== 0
          }">
            <img :src="nodeIcons[node.master_key]" width="50" class="icon" />
            <small><code class="text-white">
              {{ node.master_key.slice(0, 5) }}...{{ node.master_key.slice(-5) }}</code>
            </small>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import hashicon from 'hashicon'

export const options = {
  streamEndpoint: 'wss://validations.zaphod.ee',
  historyPerNode: 5,
  historyLedgerCount: 10
}

export default {
  name: 'stream',
  data () {
    return {
      connection: null,
      connected: false,
      error: null,
      nodes: {},
      nodeIcons: {}
    }
  },
  computed: {
    ledgers () {
      return Object.keys(this.nodes)
        .reduce((a, n) => {
          a = a.concat(this.nodes[n].map(l => {
            return Number(l.ledger_index)
          }))
          return a
        }, [])
        .sort()
        .filter((value, index, self) => {
          return self.indexOf(value) === index
        })
        .reverse()
        .filter(l => {
          return Object.values(this.nodes).map(n => n[0]).filter(n => n.ledger_index === String(l)).length > 0
        })
        .slice(0, options.historyLedgerCount)
    },
    nodesPerLedger () {
      return this.ledgers.reduce((a, ledger) => {
        const nodes = Object.values(this.nodes).map(n => n[0]).filter(n => n.ledger_index === String(ledger))
        Object.assign(a, { [String(ledger)]: nodes })
        return a
      }, {})
    }
  },
  methods: {
    onMessage (msg) {
      try {
        const data = JSON.parse(msg)
        if (data.master_key) {
          if (Object.keys(this.nodes).indexOf(data.master_key) < 0) {
            // Add node while triggering reactivity
            this.$set(this.nodes, data.master_key, [])
            this.nodeIcons[data.master_key] = hashicon(data.master_key).toDataURL()
          }
          // Add new record on top
          this.nodes[data.master_key].unshift(data)
          // Slice history per node
          if (this.nodes[data.master_key].length > options.historyPerNode) {
            this.nodes[data.master_key] = this.nodes[data.master_key].slice(0, options.historyPerNode)
          }
        }
      } catch (e) {
        this.error = e
      }
    },
    connectIfRequired () {
      if (this.connection === null) {
        this.connection = new WebSocket(options.streamEndpoint)

        this.connection.onerror = e => {
          console.log('WebSocket Error', e)
          this.error = e
        }
        this.connection.onclose = c => {
          console.log('WebSocket closed', c)
          this.connected = false
        }
        this.connection.onopen = _ => {
          console.log('WebSocket ready')
          this.connected = true
        }

        this.connection.onmessage = m => this.onMessage(m.data)
      }
    }
  },
  mounted () {
    this.connectIfRequired()
  }
}
</script>

<style scoped lang="scss">
  ul.node-list {
    li {
      display: inline-block;
      margin: 3px;
      padding: 4px;
      padding-right: 8px;
      padding-left: 30px;
      height: 50px;
      margin-right: 40px;
      margin-bottom: 10px;
      border-top-right-radius: 6px;
      border-bottom-right-radius: 6px;

      >img.icon {
        position: absolute;
        margin-left: -55px;
        margin-top: -4px;
      }
    }
  }
</style>
