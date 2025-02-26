<template lang="pug">
alcor-container.p-3.w-100.chart-container-inner
  .d-flex.justify-content-between
    el-radio-group.custom-radio.p-1.bg-base.br-4(
      v-model='activeTab',
      size='small'
    )
      el-radio-button.pointer(v-for="{ label, value } in tabs" :label='value') {{ label }}

    el-radio-group.custom-radio.p-1.bg-base.br-4(
      v-model='activeTime',
      size='small'
    )
      el-radio-button.pointer(v-for="{ label, value } in times" :label='value') {{ label }}

  //- .p-absolute
    .d-flex.gap-6.align-items-center.p-relative.t-15
      .indicator.primary
      .fs-20 Swap $2.5 B
      .indicator.secondary
      .fs-20 Spot $2.5 B
  .header(v-if="sortedA && sortedB")
    .pair-container
      PairIcons(
        :token1="sortedA"
        :token2="sortedB"
        size="18"
      )
      .name-container
        .names {{ tokenA.symbol }}/{{ tokenB.symbol }}
    .both-prices(v-if="price && charts.length")
      .item
        TokenImage(:src="$tokenLogo()" height="15")
        span.text.muted.ml-1 1 {{ tokenB.symbol }} = {{ (1 / price).toFixed(8) }} {{ tokenA.symbol }}
      .item
        TokenImage(:src="$tokenLogo()" height="15")
        span.text.muted.ml-1 1 {{ tokenA.symbol }} = {{ price }} {{ tokenB.symbol }}
    .price-container(v-if="price && charts.length")
      .price {{ price }}
      //- .change()
      //-   i(:class="`el-icon-caret-${false? 'bottom': 'top'}`" v-if="true")
      //-   span.text 10%
  component(
    :is="activeTab"
    :series="series"
    height="400px"
    :color="activeTab === 'Tvl' ? '#723de4' : undefined"
    style="min-height: 400px"
    :events="chartEvents"
    :tooltipFormatter="activeTab === 'Tvl' ? (v) => `$${v}`: undefined"
  )
</template>

<script>
import JSBI from 'jsbi'
import { mapGetters, mapState } from 'vuex'

import { Price, Q128 } from '@alcorexchange/alcor-swap-sdk'

import Line from '~/components/charts/Line.vue'
import StackedColumns from '~/components/charts/StackedColumns'
import StepLine from '~/components/charts/StepLine'
import AlcorContainer from '~/components/AlcorContainer'
import PairIcons from '~/components/PairIcons'
import TokenImage from '~/components/elements/TokenImage'
import { constructPoolInstance } from '~/utils/amm'

export default {
  components: {
    Price: Line,
    //Tvl: MultiLine,
    Tvl: Line,
    Depth: StepLine,
    Volume: StackedColumns,
    Fees: StackedColumns,
    AlcorContainer,
    PairIcons,
    TokenImage,
  },

  data: () => ({
    charts: [],
    activeTab: 'Price',
    activeTime: '7D',
    times: [
      { label: '1D', value: '24H' },
      { label: '7D', value: '7D' },
      { label: '30D', value: '30D' },
      { label: 'All', value: 'All' },
    ],
    price: undefined,
  }),

  computed: {
    tabs() {
      return [
        { label: this.$t('Price'), value: 'Price' },
        { label: this.$t('TVL'), value: 'Tvl' },
        { label: this.$t('Volume'), value: 'Volume' },
        // { label: 'Fees', value: 'Fees' },
        // { label: 'Depth', value: 'Depth' }
      ]
    },
    ...mapGetters('amm/swap', [
      'tokenA',
      'tokenB',
      'isSorted',
      'sortedA',
      'sortedB',
    ]),
    ...mapState('amm', ['pools']),

    series() {
      return this[`${this.activeTab}Series`]
    },

    TvlSeries() {
      const { sortedA, sortedB } = this

      let data = []

      if (sortedA && sortedB) {
        data = this.charts.map((c) => {
          return {
            x: c._id,
            y: (c.usdReserveA + c.usdReserveB).toFixed(0),
          }
        })
      }

      return [
        {
          name: 'TVL',
          data,
        },
      ]
    },

    PriceSeries() {
      const { sortedA, sortedB } = this

      let data = []

      if (sortedA && sortedB) {
        data = this.charts.map((c) => {
          const price = new Price(
            sortedA,
            sortedB,
            Q128,
            JSBI.multiply(JSBI.BigInt(c.price), JSBI.BigInt(c.price))
          )

          return {
            x: c._id,
            y: parseFloat(this.isSorted ? price.toSignificant() : price.invert().toSignificant()),
          }
        })
      }

      return [
        {
          name: 'Price',
          data,
        },
      ]
    },

    VolumeSeries() {
      const { sortedA, sortedB } = this

      let data = []

      if (sortedA && sortedB) {
        data = this.charts.map((c) => {
          return {
            x: c._id,
            y: c.volumeUSD,
          }
        })
      }

      return [
        {
          name: 'Volume',
          data,
        },
      ]
    },

    chartEvents() {
      if (this.activeTab !== 'Price') return {}
      return {
        mouseMove: (_, __, config) => {
          if (config.dataPointIndex < 0) return
          const data = this.series[0].data
          const y = data[config.dataPointIndex].y
          this.price = y
        },
        mouseLeave: () => {
          this.setCurrentPrice()
        },
      }
    },
  },

  watch: {
    activeTime() {
      this.fetchCharts()
    },

    tokenA(from, to) {
      if (from && to && from.id == to.id) return
      this.fetchCharts()
    },

    tokenB(from, to) {
      if (from && to && from.id == to.id) return
      this.fetchCharts()
    },
  },

  mounted() {
    // FIXME: setTimeout because I get Network Error on page reload
    setTimeout(() => {
      this.fetchCharts()
    }, 100)
  },

  methods: {
    async fetchCharts() {
      if (!this.tokenA || !this.tokenB) return

      try {
        const { data } = await this.$axios.get('/v2/swap/charts', {
          params: {
            period: this.activeTime,
            tokenA: this.tokenA.id,
            tokenB: this.tokenB.id,
          },
        })
        this.charts = data
        this.setCurrentPrice()
      } catch (e) {
        console.log('Getting Chart E', e)
      }
    },

    setCurrentPrice() {
      const series = this.PriceSeries[0].data
      if (!series) return
      const lastItem = series[series.length - 1]
      if (!lastItem) return
      this.price = lastItem.y
    },
  },
}
</script>

<style scoped lang="scss">
.chart-container-inner {
  padding: 8px 12px !important;
}
.custom-radio {
  display: flex !important;
  flex-wrap: nowrap;
  padding: 2px !important;
  gap: 2px;
  &::v-deep {
    .el-radio-button__inner {
      padding: 6px 8px !important;
      font-size: 14px !important;
    }
  }
}
.indicator {
  border-radius: 50%;
  width: 9px;
  height: 9px;
  &.primary {
    background-color: rgb(63, 81, 181);
  }
  &.secondary {
    background-color: rgb(3, 169, 244);
  }
}

.header {
  display: flex;
  flex-direction: column;
  padding-top: 8px;

  .pair-container {
    display: flex;
    align-items: center;

    .icons {
      position: relative;
      display: flex;
      height: 20px;
      width: 20px;
    }

    .name-container {
      padding-left: 10px;

      .names {
        font-size: 1.4rem;
        font-weight: bold;
      }

      display: flex;
      flex-direction: column;
    }
  }
}
.both-prices {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  margin-top: 8px;

  .item {
    padding: 2px 4px;
    display: flex;
    align-items: center;
    font-size: 0.9rem;
    border-radius: var(--radius);
    border: var(--border-1);

    .icon {
      width: 15px;
      height: 15px;
      border-radius: 50%;
      margin-right: 4px;
    }
  }
}

.price-container {
  display: flex;
  align-items: center;
  margin-bottom: 0px;

  .price {
    font-size: 1.2rem;
    font-weight: bold;
    margin-right: 4px;
  }

  .change {
    display: flex;
    align-items: center;
    color: var(--main-green);
    padding: 0 4px;

    &.isRed {
      color: var(--main-red);
    }

    &.isZero {
      color: var(--text-default);
    }
  }
}
</style>
