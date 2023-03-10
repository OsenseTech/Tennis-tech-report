<script setup lang="ts">
import * as echarts from 'echarts/core'
import {
  GeoComponent,
  ToolboxComponent,
  TooltipComponent,
  LegendComponent
} from 'echarts/components'
import { ScatterChart } from 'echarts/charts'
import { UniversalTransition } from 'echarts/features'
import { CanvasRenderer } from 'echarts/renderers'
import { postDropPoints, deleteDropPoints, patchDropPoints } from '@/utilies/dropPoints'
import svg from '@/assets/images/2d-court.svg'
import Modal from './common/Modal.vue'
const addPointModal = ref(false)
const deleteAskmodal = ref(false)
const position = ref([])
const editType = ref<string>('')
const pointId = ref('')
let pointSelect = 0
const formData = ref({
  shot: 'Not Tag',
  status: 'In',
  type: 'Not Tag'
})
const addPointOption = {
  shot: ['Not Tag', 'First Shot', 'Second Shot'],
  status: ['In', 'Faults'],
  type: ['Not Tag', 'KICK', 'SLICE', 'FLAT', 'OTHER']
}
const props = defineProps(
  {
    dropPoints: {
      type: Array,
      default: () => []
    },
    ballType: {
      type: Array,
      default: () => []
    },
    playerId: {
      type: String,
      default: () => ''
    },
    videoTiming: {
      type: Number,
      default: () => 0
    },
    contestId: {
      type: String,
      default: () => ''
    },
    contestData: {
      type: Object,
      default: () => ({ })
    },
    mirroToLeft: {
      type: Boolean,
      default: false
    }

  }
)

const emit = defineEmits(['addPoint', 'resize', 'on-reload', 'on-cancel-edit'])

echarts.use([
  ToolboxComponent,
  TooltipComponent,
  GeoComponent,
  LegendComponent,
  ScatterChart,
  CanvasRenderer,
  UniversalTransition
])
const field = ref()
const canEdit = ref(false)

// api 取回地圖長寬上限
const maxWidth = 838
const maxHeight = 388
let fileterStatus = 'All'
const option = {
  geo: {
    map: 'field_map',
    roam: false, // 允許縮放
    left: '3%', // 固定位置等比縮放
    top: '13%',
    bottom: '13%',
    right: '3%',
    selectedMode: 'single',
    itemStyle: {
      color: '#fff'
    },
    select: {
      itemStyle: {
        color: 'green'
      },
      label: {
        show: false,
        textBorderColor: '#fff',
        textBorderWidth: 2
      }
    }
  },

  grid: {
    left: '0',
    right: '0'
    // bottom: '0%'
    // containLabel: true
  },
  tooltip: {
    showDelay: 0,
    formatter: function (params: {
      seriesName:string
      name:string
      value: Array<number>
    }) {
      if (params.value.length > 1) {
        return (
          '球員 : ' + params.value[4]

        )
      } else {
        return (
          params.seriesName +
          ' :<br/>' +
          params.name +
          ' : ' +
          params.value +
          'kg '
        )
      }
    },
    axisPointer: {
      show: true,
      type: 'cross',
      lineStyle: {
        type: 'dashed',
        width: 1
      }
    }
  },
  series: [
    {
      selectedMode: false, // 選取模式
      animation: false, // 關閉渲染動畫避免影片更新卡頓
      coordinateSystem: 'geo',
      type: 'scatter',
      data: [],
      label: {
        show: true,
        formatter: function (params: {value:string[]}) {
          const rule = (
            element: any
          ) => element.drop_point_id === params.value[2] as string
          const index = props.dropPoints.findIndex(rule) + 1
          return index === 0 ? '' : index
        }
      },
      itemStyle: {
        color: function (params: {data:Array<number>}) {
          return params.data[3]
        }
      },
      emphasis: {
        itemStyle: {
          color: undefined,
          borderColor: 'white',
          borderWidth: 2
        }
      },
      symbolSize: 22,
      select: {
        itemStyle: {
          // color: 'green',
          borderColor: 'white',
          borderWidth: 2
        }
      }
    }
  ]
}
const fieldMapSvg = async () => {
  await fetch(svg).then(response => response.text()).then(svgText => {
    echarts.registerMap('field_map', { svg: svgText })
  })
}
let fieldChart: echarts.ECharts

// - 取得點位資料
const getSelectPosition = (event:{
  event: {
    offsetX: number
    offsetY: number
  }
}) => {
  // - 點擊空白處觸發事件 取得對應點位
  // if (!event.target) {
  const pointInPixel = [event.event.offsetX, event.event.offsetY]

  position.value = fieldChart.convertFromPixel({ seriesIndex: 0 }, pointInPixel) as never

  // - position.value[0] 為 X軸 position.value[1]為 Y軸
  // - 檢查點選範圍是否為球場範圍內
  if (
    position.value[0] < 0 ||
      position.value[1] < 0 ||
      position.value[0] > maxWidth ||
      position.value[1] > maxHeight
  ) return

  fieldChart.getZr().off('click', getSelectPosition)
  canEdit.value = false
  formData.value = {
    shot: 'Not Tag',
    status: 'In',
    type: 'Not Tag'
  }
  addPointModal.value = true
}

const toggleMapEventListener = (type: 'add' | 'remove'| 'edit') => {
  editType.value = type

  canEdit.value = true
  if (type !== 'add') return
  fieldChart.getZr().on('click', getSelectPosition)

  // canEdit.value = !canEdit.value
}

const dropPointClick = (params: any) => {
  if (params.data[2].length < 1) return
  pointId.value = params.data[2]
  if (editType.value === 'remove' && canEdit.value) {
    // pointId.value = params.data[2]
    deleteAskmodal.value = true
    return
  }
  if (editType.value === 'edit' && canEdit.value) {
    // pointId.value = params.data[2]
    const pointData = props.dropPoints.find(element => {
      const point = JSON.parse(JSON.stringify(element)).drop_point_id
      return point === params.data[2]
    })
    const initForm = JSON.parse(JSON.stringify(pointData))

    formData.value = {
      shot: initForm.shot,
      status: initForm.status,
      type: initForm.type
    }
    addPointModal.value = true
    // pointId.value = params.data[2]
    // deleteAskmodal.value = true
  }
}

const deletePoint = async () => {
  try {
    await deleteDropPoints(pointId.value)
    await emit('on-reload')
    canEdit.value = false
    deleteAskmodal.value = false
    emit('on-cancel-edit')
  } catch (e) {
    console.error('delete point failed')
  }
}

const setChartResize = () => {
  fieldChart.resize()
}

const createChart = () => {
  option.series[0].data = []
  const dropPoints = JSON.parse(JSON.stringify(props.dropPoints))

  dropPoints.forEach((
    element: {
      position: any,
      type: string,
      timestamp: number,
      drop_point_id:string
      status: string
      player_id: string
    },
    index: any
  ) => {
    // be refactor filter rule its sucks
    if (element.timestamp <= props.videoTiming + 1) {
      if (fileterStatus === 'All') {
        // console.log(element.timestamp)
        const ballType:any = props.ballType.find(
          (type:any) => element.type === type.value
        )
        option.series[0].data.push([
          mirroLeftX(element.position[0]), //  Ｘ軸決定位置
          element.position[1], // y軸
          element.drop_point_id, // 點位ID
          ballType.color,
          playerName(element.player_id),
          index
        ] as never // 球種
        )
      } else {
        if (element.status === fileterStatus) {
          const ballType:any = props.ballType.find(
            (type:any) => element.type === type.value
          )
          option.series[0].data.push([
            mirroLeftX(element.position[0]), //  Ｘ軸決定位置
            element.position[1], // y軸
            element.drop_point_id, // 點位ID
            ballType.color,
            playerName(element.player_id),
            index
          ] as never // 球種
          )
        }
      }
    }
  })
  playerPositionSet()
  if (fieldChart) {
    console.log(option)
    option && fieldChart.setOption(option)
  }
}

const playerPositionSet = () => {
  console.log(JSON.parse(JSON.stringify(props.contestData.positions)))
  const color1 = '#FFBC35'
  const color2 = '#569A45'
  props.contestData.positions.forEach((
    positionData: {
      timestamp: number;
      position: any[];
      player_id: string
    }) => {
    if (positionData.timestamp <= props.videoTiming + 1) {
      let x = positionData.position[0]
      const y = positionData.position[1]

      x = x > maxWidth / 2 ? maxWidth + 10 : -10
      option.series[0].data.push([
        x, //  Ｘ軸決定位置
        y, // y軸
        '', // 點位ID
        props.contestData.player1_id === positionData.player_id ? color1 : color2,
        playerName(positionData.player_id),
        '2'
      ] as never
      )
    }
  })
}

const playerName = (id:string) => {
  const player1Id = props.contestData.player1_id
  const player2Id = props.contestData.player2_id
  if (id === player1Id) return props.contestData.player1_name
  if (id === player2Id) return props.contestData.player2_name
}

const mirroLeftX = (x:number) => {
  if (props.mirroToLeft) {
    // 距離中心點
    const center = maxWidth / 2
    // 距離䦆對值
    const gap = x - center
    if (x > maxWidth / 2) return center - gap
  }
  return x
}

const savePoint = async () => {
  if (editType.value === 'add') {
    newPoint()
  } else {
    editPoint()
  }
  emit('on-cancel-edit')
}

const editPoint = async () => {
  const requsetBody = {
    ...formData.value
  }

  try {
    await patchDropPoints(pointId.value, requsetBody)
    await emit('on-reload')
    canEdit.value = false
    addPointModal.value = false
  } catch (e) {
    console.error('edit point failed')
  }
}
const newPoint = async () => {
  const dropPosition = JSON.parse(JSON.stringify(position.value))
  dropPosition[0] = dropPosition[0].toFixed(0)
  dropPosition[1] = dropPosition[1].toFixed(1)
  const requsetBody = {
    ...formData.value,
    player_id: props.playerId,
    timestamp: props.videoTiming,
    position: dropPosition,
    contest_id: props.contestId
  }
  try {
    const res = await postDropPoints(requsetBody)
    addPointModal.value = false
    emit('on-reload')
    emit('on-cancel-edit')
  } catch (err) {
    throw new Error('Add new Point failed')
  }
}
const highLightPoint = (originIndex:never) => {
  // 取消前一個被選取
  let highlightIndex = -1
  option.series[0].data.forEach((elment, index) => {
    if (elment[5] === originIndex) highlightIndex = index
  })

  fieldChart.dispatchAction({
    type: 'downplay',
    dataIndex: pointSelect
  })
  fieldChart.dispatchAction({
    type: 'highlight',
    dataIndex: highlightIndex
  })
  pointSelect = highlightIndex
}

const filterPoint = (key:string) => {
  fileterStatus = key
  createChart()
}
onBeforeMount(() => {
  createChart()
})
onMounted(async () => {
  const chartDom = field.value as unknown as HTMLElement
  await fieldMapSvg()
  fieldChart = echarts.init(chartDom)
  console.log(option)
  option && fieldChart.setOption(option)
  fieldChart.on('click', dropPointClick)
})
defineExpose({ setChartResize, toggleMapEventListener, createChart, highLightPoint })
</script>

<template>
  <div class="flex flex-col items-center">
    <div
      ref="field"
      class="w-full h-full"
    />
    <FieldMapFilter @on-change="filterPoint" />
    <Modal
      v-model:value="addPointModal"
      :showmask="false"
      :mask-close="false"
      title="落點編輯"
      @on-close="addPointModal = false;emit('on-cancel-edit'); canEdit = false"
    >
      <p>球序：</p>
      <div class="flex flex-row gap-3 px-2 mt-5 mb-9">
        <div
          v-for="(select, index) in addPointOption.shot"
          :key="index"
        >
          <input
            v-model="formData.shot"
            type="radio"
            :value="select"
          >
          <label class="mr-[6px] ml-1"> {{ select }} </label>
        </div>
      </div>
      <p>落點：</p>
      <div class="flex flex-row gap-3 px-2 mt-5 mb-9">
        <div
          v-for="(select, index) in addPointOption.status"
          :key="index"
        >
          <input
            v-model="formData.status"
            type="radio"
            :value="select"
          >
          <label class="mr-[6px] ml-1"> {{ select }} </label>
        </div>
      </div>
      <p>球種：</p>
      <div class="flex flex-row flex-wrap gap-3 px-2 mt-5 mb-9 w-[327px] justify-between bg-gray-1 pr-[90px]">
        <div
          v-for="(select, index) in addPointOption.type"
          :key="index"
        >
          <input
            v-model="formData.type"
            type="radio"
            :value="select"
          >
          <label class="mr-[6px] ml-1"> {{ select }} </label>
        </div>
      </div>
      <Button
        class="h-14 text-[24px]"
        @click="savePoint"
      >
        Save
      </Button>
    </Modal>
    <Modal
      v-model:value="deleteAskmodal"
      title="刪除落點紀錄"
      content="提醒您，落點刪除後將無哪復原。"
      :confirm-mode="true"
      @on-close="deleteAskmodal = false;canEdit = false;emit('on-cancel-edit')"
      @cancel="deleteAskmodal = false;canEdit = false;emit('on-cancel-edit')"
      @confirm="deletePoint"
    />
  </div>
</template>
