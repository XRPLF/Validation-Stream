<template>
  <div class="stream">
    <div class="alert alert-warning text-center mb-1 font-weight-bold" v-if="!connected">
      Connecting
    </div>
    <div class="alert alert-success text-center mb-1 font-weight-bold" v-if="connected">
      <div class="d-inline-block px-2">
        <b>Connected</b> - Last ledger: <b>{{ lastLedgerIndex }}</b>
      </div>
      <div class="d-inline-block px-2">
        <input type="checkbox" id="hashicons" v-model="hashicon" />
        <label for="hashicons" class="px-1">Show Hashicons</label>
      </div>
    </div>
    <div class="alert alert-danger text-center mb-1 font-weight-bold" v-if="error">
      <b>Error:</b>
      <pre>{{ error }}</pre>
    </div>
    <div :class="{
      'card shadow-sm': hashicon
    }">
      <div :class="{
        'card-body': hashicon
      }">
        <ul class="node-list mt-4 pb-0 mb-0" :class="{
          'hashicon': hashicon
        }">
          <li v-for="node in Object.keys(nodes)" v-bind:key="node" class="border" :class="{
            'border-muted is-behind': nodes[node][0].ledger_index !== prevLedgerIndex && nodes[node][0].ledger_index !== lastLedgerIndex,
            'border-primary': nodes[node][0].ledger_index === lastLedgerIndex,
            'border-secondary is-prev': nodes[node][0].ledger_index === prevLedgerIndex,
            'bg-primary is-first': node === firstNode
          }">
            <img v-if="hashicon" :src="nodeIcons[node]" width="35" class="icon" />
            <i v-if="node !== firstNode" :class="{
              'bg-primary': nodes[node][0].ledger_index === lastLedgerIndex,
              'bg-light': nodes[node][0].ledger_index !== lastLedgerIndex
            }" class="shadow-sm last"></i>
            <i v-if="node !== firstNode" :class="{
              'bg-secondary': nodes[node][0].ledger_index === prevLedgerIndex,
              'bg-light': nodes[node][0].ledger_index !== prevLedgerIndex
            }" class="shadow-sm prev"></i>
            <i v-if="node !== firstNode" :class="{
              'bg-danger': nodes[node][0].ledger_index !== prevLedgerIndex && nodes[node][0].ledger_index !== lastLedgerIndex,
              'bg-light': nodes[node][0].ledger_index === prevLedgerIndex || nodes[node][0].ledger_index === lastLedgerIndex
            }" class="shadow-sm old"></i>
            <small><code :class="{
              'text-primary':node !== firstNode && nodes[node][0].ledger_index === lastLedgerIndex,
              'text-secondary':node !== firstNode && nodes[node][0].ledger_index === prevLedgerIndex,
              'text-danger':node !== firstNode && nodes[node][0].ledger_index !== prevLedgerIndex && nodes[node][0].ledger_index !== lastLedgerIndex,
              'text-white': node === firstNode
            }">
              {{ node.slice(0, 5) }}...{{ node.slice(-5) }}</code>
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
      nodeIcons: {},
      lastLedgerIndex: '',
      prevLedgerIndex: '',
      firstNode: '',
      hashicon: false
    }
  },
  computed: {
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
          if (typeof data.ledger_index !== 'undefined') {
            const ledgerIndex = Number(data.ledger_index)
            if (ledgerIndex > this.lastLedgerIndex) {
              this.prevLedgerIndex = String(this.lastLedgerIndex)
              this.lastLedgerIndex = String(ledgerIndex)
              this.firstNode = data.master_key
            }
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
    margin: 0;
    padding: 0;
    margin-left: 1em;

    &.hashicon {
      >li {
        height: 32px;
        border-top-right-radius: 7px;
        border-bottom-right-radius: 7px;
        border-radius: 0px;
        margin-right: 30px;
        padding-left: 25px;
        border-top-right-radius: 7px;
        border-bottom-right-radius: 7px;
      }
    }

    li {
      display: inline-block;
      position: relative;
      margin: 3px;
      padding: 4px;
      padding-right: 22px;
      padding-left: 9px;
      margin-right: 20px;
      margin-bottom: 10px;
      border-radius: 5px;
      transition: transform .1s ease-in, opacity .3s ease-in;

      // box-shadow: 0px 3px 7px -3px rgba(0, 0, 0, 0.4);
      transform: scale(1.01);

      &.is-first {
        box-shadow: 0px 4px 10px -2px rgba(0, 0, 0, 0.5);
        transform: scale(1.08);
      }

      &.is-prev {
        transform: scale(0.98);
      }

      &.is-behind {
        opacity: .3;
        transform: scale(.85);
      }

      >img.icon {
        position: absolute;
        margin-left: -43px;
        margin-top: -8px;
      }

      >i {
        position: absolute;
        display: block;
        right: 2px;
        top: 2px;
        width: 8px;
        border-radius: 8px;
        height: 8px;

        &.prev {
          top: 11px;
        }

        &.old {
          top: 20px;
        }
      }
    }
  }
</style>
