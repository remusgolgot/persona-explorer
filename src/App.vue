<template>
  <main
    :class="[
      nightMode ? 'theme-dark' : 'theme-light',
      'bg-theme-page-background text-theme-text-content min-h-screen font-sans xl:pt-8'
    ]"
  >
    <AppHeader />

    <RouterView />

    <AppFooter />
  </main>
</template>

<script type="text/ecmascript-6">
import AppHeader from '@/components/header/AppHeader'
import AppFooter from '@/components/AppFooter'
import { BlockchainService, CryptoCompareService, DelegateService, NodeService } from '@/services'
import { mapGetters } from 'vuex'
import moment from 'moment'

import '@/styles/style.css'

export default {
  components: { AppHeader, AppFooter },

  data: () => ({
    currencyTimer: null,
    networkTimer: null
  }),

  computed: {
    ...mapGetters('currency', { currencyName: 'name' }),
    ...mapGetters('ui', ['language', 'locale', 'nightMode']),
    ...mapGetters('network', ['token'])
  },

  async created () {
    const network = require(`../networks/${process.env.EXPLORER_CONFIG}`)

    this.$store.dispatch(
      'ui/setNightMode',
      localStorage.getItem('nightMode') || ((network.alias === 'Development'))
    )

    this.$store.dispatch('network/setDefaults', network.defaults)

    this.$store.dispatch('network/setServer', network.server)
    this.$store.dispatch('network/setAlias', network.alias)
    this.$store.dispatch('network/setActiveDelegates', network.activeDelegates)
    this.$store.dispatch('network/setRewardOffset', network.rewardOffset)
    this.$store.dispatch('network/setCurrencies', network.currencies)
    this.$store.dispatch('network/setKnownWallets', network.knownWallets)

    if (network.defaults.currency) {
      this.$store.dispatch(
        'currency/setName',
        localStorage.getItem('currencyName') || network.defaults.currency.name
      )

      this.$store.dispatch(
        'currency/setSymbol',
        localStorage.getItem('currencySymbol') || network.defaults.currency.symbol
      )
    }

    const response = await NodeService.config()
    this.$store.dispatch('network/setToken', response.token)
    this.$store.dispatch('network/setSymbol', response.symbol)
    this.$store.dispatch('network/setNethash', response.nethash)

    this.$store.dispatch(
      'ui/setLanguage',
      localStorage.getItem('language') || 'en-gb'
    )

    this.$store.dispatch(
      'ui/setLocale',
      localStorage.getItem('locale') || navigator.language || 'en-gb'
    )

    this.$store.dispatch(
      'ui/setPriceChart',
      localStorage.getItem('priceChart') || network.defaults.priceChart
    )

    this.$store.dispatch(
      'ui/setPriceChartPeriod',
      localStorage.getItem('priceChartPeriod') || network.defaults.priceChartPeriod
    )

    this.updateI18n()
    this.updateLocale()
    this.updateCurrencyRate()
    this.updateSupply()
    this.updateHeight()
    this.updateDelegates()
  },

  mounted () {
    this.prepareComponent()
  },

  beforeDestroy () {
    this.clearTimers()
  },

  methods: {
    prepareComponent () {
      this.initialiseTimers()
    },

    async updateCurrencyRate () {

        const rate = await CryptoCompareService.price(this.currencyName)
        this.$store.dispatch('currency/setRate', rate)

    },

    async updateSupply () {
      const supply = await BlockchainService.supply()
      this.$store.dispatch('network/setSupply', supply)
    },

    async updateHeight () {
      const height = await BlockchainService.height()
      this.$store.dispatch('network/setHeight', height)
    },

    async updateDelegates () {
      const delegates = await DelegateService.all()
      this.$store.dispatch('delegates/setDelegates', delegates)
    },

    updateI18n () {
      this.$i18n.locale = this.language
    },

    updateLocale () {
      moment.locale(this.locale)
    },

    initialiseTimers () {
      this.currencyTimer = setInterval(() => {
        this.updateCurrencyRate()
      }, 5 * 30 * 1000)

      this.networkTimer = setInterval(() => {
        this.updateSupply()
        this.updateHeight()
        this.updateDelegates()
      }, 8 * 1000)
    },

    clearTimers () {
      clearInterval(this.currencyTimer)
      clearInterval(this.networkTimer)
    }
  }
}
</script>
