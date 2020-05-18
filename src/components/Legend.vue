<template>
<div class="trading-vue-legend"
     v-bind:style="calc_style">
    <div v-if="grid_id === 0"
         class="trading-vue-ohlcv"
        :style = "{ 'max-width': common.width + 'px' }">
        <span class="t-vue-title"
             :style="{ color: common.colors.colorTitle }">
              {{common.title_txt}}
        </span>
        O<span class="t-vue-lspan" >{{ohlcv[0]}}</span>
        H<span class="t-vue-lspan" >{{ohlcv[1]}}</span>
        L<span class="t-vue-lspan" >{{ohlcv[2]}}</span>
        C<span class="t-vue-lspan" >{{ohlcv[3]}}</span>
        V<span class="t-vue-lspan" >{{ohlcv[4]}}</span>
    </div>
    <div class="t-vue-indicators">
        <div class="t-vue-ind" v-for="ind in this.indicators" v-bind:id="ind.id">
            <span class="t-vue-iname">{{ind.name}}</span>
            <button-group
                v-bind:buttons="common.buttons"
                v-bind:ov_id="ind.id"
                v-bind:grid_id="grid_id"
                v-bind:index="ind.index"
                v-bind:tv_id="common.tv_id"
                v-bind:display="ind.v"
                v-on:legend-button-click="button_click">
            </button-group>
            <span class="t-vue-ivalues" v-if="ind.v">
                <span class="t-vue-lspan t-vue-ivalue"
                      v-for="v in ind.values" :style="{ color: v.color }">
                    {{v.value}}
                </span>
            </span>
            <span v-if="ind.unk" class="t-vue-unknown">
                (Unknown type)
            </span>
        </div>
    </div>
</div>
</template>
<script>

import ButtonGroup from './ButtonGroup.vue'

export default {
    // If this component is called ChartLegend, then why is it named Legend.vue? It should be named as ChartLegend.
    // A ChartLegend consists of a list of ButtonGroup?
    name: 'ChartLegend',

    // TODO: What are all these props? What are typical values for these props?
    props: [
        // HN: This seems to have
        //      common.layout.grids[id]
        //      common.layout.grids[id].height
        //      common.layout.grids[id].offset
        //      common.layout.grids[id].prec
        //      common.data -> seems to be whole data.json structure??
        'common',
        // HN: This seems to have
        //      values.ohlcv   -> ohlcv value at the current cursor
        'values',

        // HN: This seems to be some kind of Id in the this.$props.common.layout.grids[id]
        'grid_id',
        'meta_props'
    ],
    components: { ButtonGroup },
    computed: {
        ohlcv() {
            if (!this.$props.values || !this.$props.values.ohlcv) {
                return Array(6).fill('n/a')
            }
            const prec = this.layout.prec

            let volumeDisplay = 'n/a'
            if (this.$props.values.ohlcv[5]) {
                let volume = this.$props.values.ohlcv[5]
                if (volume > 1000*1000*1000) {
                    volumeDisplay = (volume / 1000000000.0).toFixed(2) + " B"
                } else if (volume > 100*1000*1000) {
                        volumeDisplay = (volume / 1000000.0).toFixed(0) + " M"
                } else if (volume > 10*1000*1000) {
                    volumeDisplay = (volume / 1000000.0).toFixed(1) + " M"
                } else if (volume > 1000*1000) {
                    volumeDisplay = (volume / 1000000.0).toFixed(2) + " M"
                } else if (volume > 100*1000) {
                    volumeDisplay = (volume / 1000.0).toFixed(0) + " K"
                } else if (volume > 10*1000) {
                    volumeDisplay = (volume / 1000.0).toFixed(1) + " K"
                } else if (volume > 1000) {
                    volumeDisplay = (volume / 1000.0).toFixed(2) + " K"
                } else {
                    volumeDisplay = volume.toFixed(0)
                }
            }

            return [
                this.$props.values.ohlcv[1].toFixed(prec),
                this.$props.values.ohlcv[2].toFixed(prec),
                this.$props.values.ohlcv[3].toFixed(prec),
                this.$props.values.ohlcv[4].toFixed(prec),
                volumeDisplay
            ]
        },
        indicators() {
            const values = this.$props.values
            var types = {}
            return this.json_data.filter(
                x => x.settings.legend !== false && !x.main
            ).map(x => {
                if (!(x.type in types)) types[x.type] = 0
                const id = x.type + `_${types[x.type]++}`   // HN: This seems to generated id = XSpline_1
                return {
                    v: 'display' in x.settings ? x.settings.display : true,
                    name: x.name || id,     // HN: Example: EMA200
                    index: this.json_data.indexOf(x),   // an index in the list of iindicators
                    id: id,                 // HN: XPline_1
                    values: values ? this.format(id, values) : this.n_a(1),
                    unk: !(id in (this.$props.meta_props || {}))
                }
            })
        },
        calc_style() {
            let top = this.layout.height > 150 ? 10 : 5
            return {
                top: `${this.layout.offset + top}px`,
            }
        },
        layout() {
            const id = this.$props.grid_id
            return this.$props.common.layout.grids[id]
        },
        json_data() {
            return this.$props.common.data
        }
    },
    methods: {
        format(id, values) {
            let meta = this.$props.meta_props[id] || {}
            // Matches Overlay.data_colors with the data values
            // (see Spline.vue)
            if (!values[id]) return this.n_a(1)

            // Custom formatter
            if (meta.legend) return meta.legend(values[id])

            return values[id].slice(1).map((x, i) => {
                const cs = meta.data_colors ? meta.data_colors() : []
                if (typeof x == 'number') {
                    // Show 8 digits for small values
                    x = x.toFixed(x > 0.001 ? 4 : 8)
                }
                return {
                    value: x,
                    color: cs ? cs[i] : undefined
                }
            })
        },
        n_a(len) {
            return Array(len).fill({ value: 'n/a' })
        },
        button_click(event) {
            this.$emit('legend-button-click', event)
        }
    }
}
</script>
<style>
.trading-vue-legend {
    position: relative;
    z-index: 100;
    font-size: 1.25em;
    margin-left: 10px;
    pointer-events: none;
}
.trading-vue-ohlcv {
    pointer-events: none;
    margin-bottom: 0.5em;
}
.t-vue-lspan {
    font-variant-numeric: tabular-nums;
    font-weight: 100;
    font-size: 0.95em;
    color: #999999; /* TODO: move => params */
    margin-left: 0.1em;
    margin-right: 0.2em;
}
.t-vue-title {
    margin-right: 0.25em;
    font-size: 1.45em;
    font-weight: 200;
}
.t-vue-ind {
    margin-left: 0.2em;
    margin-bottom: 0.5em;
    font-weight: 200;
    font-size: 1.0em;
}
.t-vue-ivalue {
    margin-left: 0.5em;
}
.t-vue-unknown {
    color: #999999; /* TODO: move => params */
}
</style>
