
<script setup lang="ts">

import videoJs from 'video.js'
import 'video.js/dist/video-js.min.css'
import { getUser } from '@/utilies/user'
import { getContest } from '@/utilies/contests'
import { postMilestones } from '@/utilies/milestones'

const rVideoPlayer = ref<Element>()

const fieleMap = ref() // 球場element
const timeline = ref() // 球序圖element
const fieldMapWidth = ref() // 球場響應寬度
const contentData = ref() // 球賽資料
const player1 = ref()
const player2 = ref()
const playerSelected = ref() // 選取中球員Id
const mirro = ref(false) // 是否鏡像換場 true 為鏡向
const videoTiming = ref()
const editing = ref(false)
const editType = ref('')
let video: { on: (arg0: string, arg1: () => void) => void; currentTime: (arg0?: number | undefined) => number }
watch(rVideoPlayer, () => {
  video = videoJs(rVideoPlayer.value, {
    aspectRatio: '16:9',
    // bigPlayButton: false,
    // textTrackDisplay: false,
    // posterImage: false,
    // errorDisplay: false,
    muted: true,
    controlBar: {
      // - 禁用全螢幕項目
      fullscreenToggle: false,
      inFullscreenClass: 'vjs-fullscreen-disabled',
      // - 禁用子母畫面功能
      pictureInPictureToggle: false
    }
    // ...其他配置参数
  })

  video.on('timeupdate', () => {
    videoTiming.value = contentData.value.start_time + video.currentTime()
    fieleMap.value.createChart()
    timeline.value.resetChart(video.currentTime())
  })
})

const ballType = [
  { value: 'KICK', label: 'KICK', color: '#46FCC5' },
  { value: 'SLICE', label: 'SLICE', color: '#FFBC35' },
  { value: 'FLAT', label: 'FLAT', color: '#951D1D' },
  { value: 'OTHER', label: 'OTHER', color: '#F8F5FA' }
]
// - 等比自適應撐高
const autoHeight = computed(() => {
  return `min-height: ${fieldMapWidth.value * 3 / 4 / 838 * 388 + 100}px`
})

const apiGetUser = async (id:string) => {
  const { data } = await getUser(id)
  return data
}

const defaultPlayerSettings = () => {
  if (player1.value && player2.value) {
    playerSelected.value = player2.value.user_id
    return
  }
  if (player2.value && !player1.value) {
    playerSelected.value = player2.value.user_id
    return
  }
  if (player1.value && !player2.value) {
    playerSelected.value = player1.value.user_id
  }
}
const selectedPlayer1Style = computed(() => {
  const selected = player1.value && player1.value.user_id === playerSelected.value
  if (selected) return 'font-bold text-3xl text-white'
  return 'font-bold text-3xl text-green-1'
})

const selectedPlayer2Style = computed(() => {
  const selected = player2.value && player2.value.user_id === playerSelected.value
  if (selected) return 'font-bold text-3xl text-white'
  return 'font-bold text-3xl text-green-1'
})

const apiGetContest = async () => {
  const id = window.location.pathname.replace('/admin/video/', '')
  const { data } = await getContest(id, { detail: true })
  editType.value = ''
  contentData.value = data
  if (!videoTiming.value) videoTiming.value = contentData.value.start_time
  if (data.player1_id) player1.value = await apiGetUser(data.player1_id)
  if (data.player2_id) player2.value = await apiGetUser(data.player2_id)
  timeline.value.resetChart()
  fieleMap.value.createChart()
  defaultPlayerSettings()
}

const selectPoint = (index:number, time: number) => {
  const timeing = time - contentData.value.start_time
  video.currentTime(timeing)
  videoTiming.value = contentData.value.start_time + video.currentTime()
  nextTick(() => {
    fieleMap.value.createChart()
    fieleMap.value.highLightPoint(index)
  })
}

const videoTitle = computed(() => {
  const player1 = contentData.value?.player1_name || ''
  const player2 = contentData.value?.player2_name || ''
  const subTitle = player1 + player2
  return `${subTitle} ${contentData.value?.title}`
})

const changeField = async () => {
  const contestId = contentData.value.contest_id
  const title = videoTiming.value + contentData.value.title
  const type = 'change'
  const timestamp = videoTiming.value
  try {
    await postMilestones({
      contest_id: contestId,
      type,
      title,
      timestamp
    })
    apiGetContest()
  } catch
  (err) {
    console.error(err, 'Post Milestones failed')
  }
}

const editTypeSwitch = (type:string) => {
  editType.value = type
  fieleMap.value.toggleMapEventListener(type)
  editing.value = true
}
onBeforeMount(async () => {
  apiGetContest()
})

// 掛載dom 監聽resize設置圖表重繪機制
onMounted(() => {
  fieldMapWidth.value = window.innerWidth
  window.onresize = function () {
    fieldMapWidth.value = window.innerWidth
    fieleMap.value.setChartResize()
    timeline.value.setChartResize()
  }
})

</script>

<template>
  <header class="h-[60px] lg:h-[86px] w-full flex flex-row justify-between items-center bg-green-1 text-white text-2xl lg:text-4xl px-6">
    <span>{{ videoTitle }}</span>
    <span class="text-4xl lg:text-7xl">{{ contentData?.drop_points.length }}</span>
  </header>
  <div
    v-if="contentData"
    class="flex flex-col items-center bg-green-3 h-[calc(100%-60px)] lg:h-[calc(100%-86px)] overflow-y-auto"
  >
    <div class="flex flex-col items-center bg-green-3 h-full w-3/4">
      <video
        id="my-video"
        ref="rVideoPlayer"
        class="video-js vjs-big-play-centered"
        controls
        controlsList="nofullscreen"
        preload="auto"
      >
        <source
          :src="contentData.video_url"
          type="application/x-mpegURL"
        >
      </video>
      <div class="w-full min-h-20">
        <TennisTimeline
          ref="timeline"
          :end-time="contentData.end_time"
          :start-time="contentData.start_time"
          :drop-points="contentData.drop_points"
          :content-data="contentData"
          :ball-type="ballType"
          @on-click="selectPoint"
        />
      </div>
      <!-- To fix 這裡開始應該要被抽離成組件 -->
      <div class="flex justify-between w-full pt-8 border-dashed border-gray-3 border-t-2 items-center">
        <div
          :class="mirro ? 'flex-row-reverse' : 'flex-row'"
          class="flex gap-20 items-center"
        >
          <div
            :class="player2 && playerSelected === player2.user_id ? 'px-7 py-3 bg-green-1 rounded-xl':'cursor-pointer'"
            @click="playerSelected = player2.user_id"
          >
            <span
              :class="selectedPlayer2Style"
            >{{ player2 ? `${player2.username}` : '' }}</span>
          </div>
          <span class="text-gray-3 text-4xl">VS</span>
          <div
            :class="player1 && playerSelected === player1.user_id ? 'px-7 py-3 bg-green-1 rounded-xl':'cursor-pointer'"
            @click="playerSelected = player1.user_id"
          >
            <span
              :class="selectedPlayer1Style"
            >{{ player1 ? `${player1.username}` : '' }}</span>
          </div>
        </div>
        <div class="flex flex-row items-center gap-4">
          <div
            class="h-[61px] w-[118px] rounded-[48px] flex flex-row items-center justify-center cursor-pointer gap-3"
            :class="editType === 'edit' ? 'bg-black text-white' : 'bg-white'"
            @click="editTypeSwitch('edit')"
          >
            <BaseSvgIcon
              name="edit"
              :class="editType === 'edit' ? 'fill-white' : 'fill-gray-4'"
              class="w-[24px] h-[24px] "
            />
            <span class="text-xl">Tag</span>
          </div>
          <div
            class="h-[61px] w-[118px] rounded-[48px] flex flex-row items-center justify-center cursor-pointer gap-3"
            :class="editType === 'add' ? 'bg-black text-white' : 'bg-white'"
            @click="editTypeSwitch('add')"
          >
            <BaseSvgIcon
              name="plus"
              :class="editType === 'add' ? 'fill-white' : 'fill-gray-4'"
              class="w-[24px] h-[24px] fill-gray-4"
            />
            <span class="text-xl">Tag</span>
          </div>
          <div
            class="h-[61px] w-[118px] rounded-[48px] flex flex-row items-center justify-center cursor-pointer gap-3"
            :class="editType === 'remove' ? 'bg-black text-white' : 'bg-white'"
            @click="editTypeSwitch('remove')"
          >
            <span class="text-xl">Tag</span>
            <BaseSvgIcon
              name="close"
              :class="editType === 'remove' ? 'fill-white' : 'fill-gray-4'"
              class="w-[24px] h-[24px] fill-gray-4"
            />
          </div>
          <Button
            theme="secondary"
            class="w-[264px] h-[61px] text-4xl"
            @click="changeField"
          >
            Change Field
          </Button>
        </div>
      </div>
      <!-- 到這 -->
      <TennisFieldMap
        ref="fieleMap"
        :contest-data="contentData"
        :video-timing="videoTiming"
        :player-id="playerSelected"
        :ball-type="ballType"
        :contest-id="contentData.contest_id"
        :drop-points="contentData.drop_points"
        :style="autoHeight"
        class="w-3/4"
        @on-cancel-edit="editing = false;editType = ''"
        @on-reload="apiGetContest"
      />
      <div class="w-full min-h-48" />
    </div>
  </div>
</template>
