<script setup lang="ts">
import * as echarts from 'echarts'
import { EChartOption } from 'echarts'
import { onMounted } from 'vue'

const timeline = ref()
const props = defineProps(
  {
    contentData: {
      type: Object,
      default: () => ({})
    },
    dropPoints: {
      type: Array,
      default: () => []
    },
    endTime: {
      type: Number,
      default: 0
    },
    startTime: {
      type: Number,
      default: 0
    }
  }
)
const emit = defineEmits(['onClick'])
const mixTimeing = props.endTime - props.startTime
// console.log(mixTimeing)
const option:any = {
  xAxis: {
    type: 'value',
    min: 0,
    max: mixTimeing,
    show: false
  },
  yAxis: {
    splitNumber: 0.5,
    show: false
  },

  toolbox: {
    show: false,
    feature: {
      restore: {},
      saveAsImage: {}
    }
  },
  grid: {
    left: '8%',
    right: '10%'
  },
  dataZoom: [{
    type: 'inside'
  },
  {
    type: 'slider',
    // preventDefaultMouseMove: true,
    handleStyle: {
      borderWidth: 4,
      borderColor: '#569A45',
      color: '#569A45'
    },
    moveHandleSize: 10,
    moveHandleStyle: {
      color: '#212531'
    },
    textStyle: {
      fontStyle: 'oblique'
    },
    backgroundColor: '#212531',
    borderColor: '#212531',
    fillerColor: 'rgba(248, 245, 250, 0.2)'
  }
  ],
  color: ['#1DC9FE', '#F6A120'],
  tooltip: {
    // -  should be any
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    formatter: function (player:any) {
      return '時間：' + Number.parseFloat(player.value[0]).toFixed(2) + ' 秒' +
      '<br />' +
      '球員：' + players[player.value[2]]
    },
    show: true,
    trigger: 'item',
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    position: function (point: any[]) {
      // 固定在顶部
      return [point[0], '10%']
    }
  },
  series: [
    {
      symbolSize: 20,
      data: [],
      type: 'scatter',
      itemStyle: {
        color: function (params: {data:Array<number>}) {
          return params.data[3]
        }
      },
      label: {
        show: true,
        formatter: function (params: {dataIndex:number}) {
          return params.dataIndex + 1
        }
      }
    }
  ]
}

let timelineChart: echarts.EChartsType
watch(props.dropPoints, () => {
  resetChart()
})
const players:{
  [key:string]:string
} = {}
const resetChart = (time?:number) => {
  // 由於timestamp與video js 可能產生之 js 與py 計算溢位誤差之校正
  if (time && option.xAxis && time >= 0.2 && time <= mixTimeing - 0.2) {
    option.xAxis.min = time - 10
    option.xAxis.max = time + 10
  } else if (option.xAxis) {
    option.xAxis.min = 0
    option.xAxis.max = mixTimeing
  }
  players[props.contentData.player1_id] = props.contentData.player1_name
  players[props.contentData.player2_id] = props.contentData.player2_name
  if (!option.series) return
  const dropPoints = JSON.parse(JSON.stringify(props.dropPoints))
  const chartData: unknown[] = []
  dropPoints.forEach((point:any) => {
    const time = point.timestamp - props.startTime
    const color = point.player_id === props.contentData.player1_id ? '#FFBC35' : '#569A45'
    // fix colors
    const temp = [time, 1, point.player_id, color, point.drop_point_id, point.timestamp]
    chartData.push(temp)
  })
  option.series[0].data = chartData as []
  option && timelineChart.setOption(option)
}

const setChartResize = () => {
  timelineChart.resize()
}

onMounted(() => {
  const chartDom = timeline.value as unknown as HTMLElement
  timelineChart = echarts.init(chartDom)
  option && timelineChart.setOption(option)
  resetChart()
  timelineChart.on('click', function (params: {
    dataIndex:string
    data: any;
  }) {
    emit('onClick', params.dataIndex, params.data[5])
  })
})
defineExpose({ setChartResize, resetChart })
</script>

<template>
  <div
    ref="timeline"
    class="w-full h-[100px]"
    @mousedown.stop
    @touchstart.stop
  />
</template>
