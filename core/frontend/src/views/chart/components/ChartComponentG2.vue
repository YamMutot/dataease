<template>
  <div
    ref="chartContainer"
    style="padding: 0;width: 100%;height: 100%;overflow: hidden;"
    :style="bg_class"
  >
    <view-track-bar
      ref="viewTrack"
      :track-menu="trackMenu"
      class="track-bar"
      :style="trackBarStyleTime"
      @trackClick="trackClick"
    />
    <span
      v-if="chart.type"
      v-show="title_show"
      ref="title"
      :class="titleIsRight"
      :style="title_class"
      style="cursor: default;display: block;"
    >
      <div style="padding:4px 4px 0;margin: 0;">
        <chart-title-update
          :title-class="title_class"
          :chart-info="chartInfo"
        />
        <title-remark
          v-if="remarkCfg.show"
          style="text-shadow: none!important;margin-left: 4px;"
          :remark-cfg="remarkCfg"
        />
      </div>
    </span>
    <div
      :id="chartId"
      style="width: 100%;overflow: hidden;"
      :style="{height:chartHeight}"
    />
  </div>
</template>

<script>
import { baseLiquid } from '@/views/chart/chart/liquid/liquid'
import { uuid } from 'vue-uuid'
import ViewTrackBar from '@/components/canvas/components/editor/ViewTrackBar'
import { getRemark, hexColorToRGBA } from '@/views/chart/chart/util'
import { baseBarOptionAntV, hBaseBarOptionAntV, baseBidirectionalBarOptionAntV, timeRangeBarOptionAntV } from '@/views/chart/chart/bar/bar_antv'
import { baseAreaOptionAntV, baseLineOptionAntV } from '@/views/chart/chart/line/line_antv'
import { basePieOptionAntV, basePieRoseOptionAntV } from '@/views/chart/chart/pie/pie_antv'
import { baseScatterOptionAntV } from '@/views/chart/chart/scatter/scatter_antv'
import { baseGaugeOptionAntV } from '@/views/chart/chart/gauge/gauge_antv'
import { baseFunnelOptionAntV } from '@/views/chart/chart/funnel/funnel_antv'
import { baseTreemapOptionAntV } from '@/views/chart/chart/treemap/treemap_antv'
import { baseRadarOptionAntV } from '@/views/chart/chart/radar/radar_antv'
import { baseWaterfallOptionAntV } from '@/views/chart/chart/waterfall/waterfall'
import { baseWordCloudOptionAntV } from '@/views/chart/chart/wordCloud/word_cloud'
import TitleRemark from '@/views/chart/view/TitleRemark'
import { CHART_CONT_FAMILY_MAP, DEFAULT_TITLE_STYLE } from '@/views/chart/chart/chart'
import { baseMixOptionAntV } from '@/views/chart/chart/mix/mix_antv'
import ChartTitleUpdate from './ChartTitleUpdate.vue'
import { equalsAny } from '@/utils/StringUtils'
import { mapState } from 'vuex'
import { baseFlowMapOption } from '@/views/chart/chart/map/map_antv'
import { clear } from 'size-sensor'
export default {
  name: 'ChartComponentG2',
  components: { TitleRemark, ViewTrackBar, ChartTitleUpdate },
  props: {
    chart: {
      type: Object,
      required: true
    },
    filter: {
      type: Object,
      required: false,
      default: function() {
        return {}
      }
    },
    trackMenu: {
      type: Array,
      required: false,
      default: function() {
        return ['drill']
      }
    },
    searchCount: {
      type: Number,
      required: false,
      default: 0
    },
    scale: {
      type: Number,
      required: false,
      default: 1
    }
  },
  data() {
    return {
      myChart: null,
      chartId: uuid.v1(),
      showTrackBar: true,
      trackBarStyle: {
        position: 'absolute',
        left: '0px',
        top: '0px'
      },
      pointParam: null,
      dynamicAreaCode: null,
      borderRadius: '0px',
      chartHeight: '100%',
      title_class: {
        margin: '0 0',
        width: '100%',
        fontSize: '18px',
        color: '#303133',
        textAlign: 'left',
        fontStyle: 'normal',
        fontWeight: 'normal',
        background: ''
      },
      title_show: true,
      antVRenderStatus: false,
      linkageActiveParam: null,
      linkageActiveHistory: false,
      remarkCfg: {
        show: false,
        content: ''
      },
      resizeTimer: null
    }
  },

  computed: {
    trackBarStyleTime() {
      return this.trackBarStyle
    },
    titleIsRight() {
      return this.title_class?.textAlign === 'right' && 'title-is-right'
    },
    bg_class() {
      return {
        borderRadius: this.borderRadius
      }
    },
    chartInfo() {
      const { id, title } = this.chart
      return { id, title }
    },
    ...mapState([
      'canvasStyleData'
    ])
  },
  watch: {
    chart: {
      handler(newVal, oldVla) {
        this.initTitle()
        this.calcHeightRightNow(this.drawView)
      },
      deep: true
    }
  },
  beforeDestroy() {
    if (this.myChart.container) {
      if (typeof this.myChart.container.getAttribute === 'function') {
        clear(this.myChart.container)
      }
    }
    this.myChart?.clear?.()
    this.myChart?.unbindSizeSensor?.()
    this.myChart?.unbind?.()
    this.myChart?.destroy?.()
    if (this.myChart) {
      for (const key in this.myChart.chart) {
        this.myChart.chart[key] = null
        this.$delete(this.myChart.chart, key)
      }
      for (const key in this.myChart) {
        this.myChart[key] = null
        this.$delete(this.myChart, key)
      }
    }
    for (const key in this.pointParam) {
      this.$delete(this.pointParam, key)
    }
    window.removeEventListener('resize', this.chartResize)
    this.myChart = null
  },
  mounted() {
    this.preDraw()
  },
  methods: {
    reDrawView() {
      this.linkageActiveHistory = false
      this.myChart.render()
    },
    linkageActivePre() {
      if (this.linkageActiveHistory) {
        this.reDrawView()
      }
      this.$nextTick(() => {
        this.linkageActive()
      })
    },
    linkageActive() {
      this.linkageActiveHistory = true
      this.myChart.setState('active', (param) => {
        if (Array.isArray(param)) {
          return false
        } else {
          if (this.checkSelected(param)) {
            return true
          }
        }
      })
      this.myChart.setState('inactive', (param) => {
        if (Array.isArray(param)) {
          return false
        } else {
          if (!this.checkSelected(param)) {
            return true
          }
        }
      })
    },
    clearLinkage() {
      this.linkageActiveHistory = false
      this.myChart?.setState('active', () => true, false)
      this.myChart?.setState('inactive', () => true, false)
    },
    checkSelected(param) {
      return (this.linkageActiveParam.name === param.name || (this.linkageActiveParam.name === 'NO_DATA' && !param.name)) &&
        (this.linkageActiveParam.category === param.category)
    },
    preDraw() {
      this.initTitle()
      this.calcHeightRightNow(this.drawView)
      window.addEventListener('resize', this.chartResize)
    },
    async drawView() {
      const chart = JSON.parse(JSON.stringify(this.chart))
      this.antVRenderStatus = true
      if (!chart.data || (!chart.data.data && !chart.data.series)) {
        chart.data = {
          data: [{}],
          series: [
            {
              data: [0]
            }
          ]
        }
      }
      if (chart.type === 'bar') {
        this.myChart = baseBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction, true, false)
      } else if (chart.type === 'bar-group') {
        this.myChart = baseBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction, true, false)
      } else if (equalsAny(chart.type, 'bar-stack', 'percentage-bar-stack')) {
        this.myChart = baseBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction, false, true)
      } else if (chart.type === 'bar-group-stack') {
        this.myChart = baseBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction, true, true)
      } else if (chart.type === 'bar-horizontal') {
        this.myChart = hBaseBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction, true, false)
      } else if (equalsAny(chart.type, 'bar-stack-horizontal', 'percentage-bar-stack-horizontal')) {
        this.myChart = hBaseBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction, false, true)
      } else if (equalsAny(chart.type, 'bar-time-range')) {
        this.myChart = timeRangeBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'line') {
        this.myChart = baseLineOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'area') {
        this.myChart = baseAreaOptionAntV(this.myChart, this.chartId, chart, this.antVAction, false)
      } else if (chart.type === 'line-stack') {
        this.myChart = baseAreaOptionAntV(this.myChart, this.chartId, chart, this.antVAction, true)
      } else if (chart.type === 'scatter') {
        this.myChart = baseScatterOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'radar') {
        this.myChart = baseRadarOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'gauge') {
        this.myChart = baseGaugeOptionAntV(this.myChart, this.chartId, chart, this.antVAction, this.scale)
      } else if (chart.type === 'pie' || chart.type === 'pie-donut') {
        this.myChart = basePieOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'pie-rose' || chart.type === 'pie-donut-rose') {
        this.myChart = basePieRoseOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'funnel') {
        this.myChart = baseFunnelOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'treemap') {
        this.myChart = baseTreemapOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'liquid') {
        this.myChart = baseLiquid(this.myChart, this.chartId, chart)
      } else if (chart.type === 'waterfall') {
        this.myChart = baseWaterfallOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'word-cloud') {
        this.myChart = baseWordCloudOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'chart-mix') {
        this.myChart = baseMixOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'flow-map') {
        this.myChart = await baseFlowMapOption(this.myChart, this.chartId, chart, this.antVAction)
      } else if (chart.type === 'bidirectional-bar') {
        this.myChart = baseBidirectionalBarOptionAntV(this.myChart, this.chartId, chart, this.antVAction)
      } else {
        if (this.myChart) {
          this.antVRenderStatus = false
          this.myChart.destroy()
        }
      }

      if (this.myChart && !equalsAny(chart.type, 'liquid', 'flow-map') && this.searchCount > 0) {
        this.myChart.options.animation = false
      }
      if (this.myChart?.options?.legend) {
        let pageNavigatorInactiveFill, pageNavigatorFill
        if (this.canvasStyleData.panel.themeColor === 'dark') {
          pageNavigatorFill = '#ffffff'
          pageNavigatorInactiveFill = '#8c8c8c'
        } else {
          pageNavigatorFill = '#000000'
          pageNavigatorInactiveFill = '#8c8c8c'
        }
        this.myChart.options.legend['pageNavigator'] = {
          marker: {
            style: {
              inactiveFill: pageNavigatorInactiveFill, // 不能点击的颜色
              fill: pageNavigatorFill // 正常的颜色
            }
          }
        }
      }

      if (this.antVRenderStatus) {
        this.myChart.render()
        if (this.linkageActiveHistory) {
          this.linkageActive()
        }
      }
      this.setBackGroundBorder()
    },
    antVAction(param) {
      switch (this.chart.type) {
        case 'treemap':
          this.pointParam = param.data.data
          break
        case 'word-cloud':
          this.pointParam = {
            data: param.data.data.datum
          }
          break
        default:
          this.pointParam = param.data
          break
      }
      this.linkageActiveParam = {
        category: this.pointParam.data.category ? this.pointParam.data.category : 'NO_DATA',
        name: this.pointParam.data.name ? this.pointParam.data.name : 'NO_DATA',
        group: this.pointParam.data.group ? this.pointParam.data.group : 'NO_DATA'
      }
      if (this.trackMenu.length < 2) { // 只有一个事件直接调用
        this.trackClick(this.trackMenu[0])
      } else { // 视图关联多个事件
        this.trackBarStyle.left = param.x + 'px'
        this.trackBarStyle.top = (param.y + 10) + 'px'
        this.$refs.viewTrack.trackButtonClick()
      }
    },
    setBackGroundBorder() {
      if (this.chart.customStyle) {
        const customStyle = JSON.parse(this.chart.customStyle)
        if (customStyle.background) {
          this.borderRadius = (customStyle.background.borderRadius || 0) + 'px'
        }
      }
    },
    chartResize() {
      this.resizeTimer && clearTimeout(this.resizeTimer)
      this.resizeTimer = setTimeout(() => {
        this.calcHeightRightNow()
      }, 100)
    },
    trackClick(trackAction) {
      const param = this.pointParam
      if (!param || !param.data || !param.data.dimensionList) {
        // 地图提示没有关联字段 其他没有维度信息的 直接返回
        if (this.chart.type === 'map') {
          this.$warning(this.$t('panel.no_drill_field'))
        }
        return
      }
      let quotaList = this.pointParam.data.quotaList
      if (this.chart.type === 'bar-time-range') {
        quotaList = this.pointParam.data.dimensionList
      } else {
        quotaList[0]['value'] = this.pointParam.data.value
      }
      const linkageParam = {
        option: 'linkage',
        name: this.pointParam.data.name,
        viewId: this.chart.id,
        dimensionList: this.pointParam.data.dimensionList,
        quotaList: quotaList,
        category: this.pointParam.data.category,
        group: this.pointParam.data.group
      }
      const jumpParam = {
        option: 'jump',
        name: this.pointParam.data.name,
        viewId: this.chart.id,
        dimensionList: this.pointParam.data.dimensionList,
        quotaList: quotaList,
        category: this.pointParam.data.category,
        group: this.pointParam.data.group
      }

      if (this.chart.type === 'scatter' && this.chart.render === 'antv') {
        const xAxis = JSON.parse(this.chart.xaxis)
        if (xAxis && xAxis[0] && xAxis[0].groupType === 'q') {
          linkageParam.scatterSpecial = true
          linkageParam.scatterSpecialData = this.pointParam.data

          jumpParam.scatterSpecial = true
          jumpParam.scatterSpecialData = this.pointParam.data
        }
      }

      switch (trackAction) {
        case 'drill':
          this.$emit('onChartClick', this.pointParam)
          break
        case 'linkage':
          this.linkageActivePre()
          this.$store.commit('addViewTrackFilter', linkageParam)
          break
        case 'jump':
          this.$emit('onJumpClick', jumpParam)
          break
        default:
          break
      }
    },

    initTitle() {
      if (this.chart.customStyle) {
        const customStyle = JSON.parse(this.chart.customStyle)
        if (customStyle.text) {
          this.title_show = customStyle.text.show
          this.title_class.fontSize = customStyle.text.fontSize + 'px'
          this.title_class.color = customStyle.text.color
          this.title_class.textAlign = customStyle.text.hPosition
          this.title_class.fontStyle = customStyle.text.isItalic ? 'italic' : 'normal'
          this.title_class.fontWeight = customStyle.text.isBolder ? 'bold' : 'normal'

          this.title_class.fontFamily = customStyle.text.fontFamily ? CHART_CONT_FAMILY_MAP[customStyle.text.fontFamily] : DEFAULT_TITLE_STYLE.fontFamily
          this.title_class.letterSpacing = (customStyle.text.letterSpace ? customStyle.text.letterSpace : DEFAULT_TITLE_STYLE.letterSpace) + 'px'
          this.title_class.textShadow = customStyle.text.fontShadow ? '2px 2px 4px' : 'none'
        }
        if (customStyle.background) {
          this.title_class.background = hexColorToRGBA(customStyle.background.color, customStyle.background.alpha)
          this.borderRadius = (customStyle.background.borderRadius || 0) + 'px'
        }
        if (this.chart.type === 'flow-map') {
          this.title_class.zIndex = 4
          this.title_class.position = 'absolute'
        }
      }
      this.initRemark()
    },

    calcHeightRightNow(callback) {
      this.$nextTick(() => {
        if (this.$refs.chartContainer) {
          const { offsetHeight } = this.$refs.chartContainer
          let titleHeight = 0
          if (this.$refs.title) {
            titleHeight = this.$refs.title.offsetHeight
          }
          this.chartHeight = (offsetHeight - titleHeight) + 'px'
          this.$nextTick(() => callback?.())
        }
      })
    },
    initRemark() {
      this.remarkCfg = getRemark(this.chart)
    }
  }
}
</script>
