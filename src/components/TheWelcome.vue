<script setup lang="ts">

import { ref, onMounted } from 'vue'
import { Timeline } from 'vue-timeline-chart'
import 'vue-timeline-chart/style.css'
import jsonData from '@/data/example.json'

const exampleData = ref(jsonData)

const timelineEvents = ref([])
const timeline = ref(null);

const inputCode = ref('')
const minT = ref('')
const maxT = ref('')
const groups = ref([])


const viewport = ref({ start: 400000, end: 700000 });
const totalRange = ref({ start: 0, end: 800000 });

let isDraggingMapViewport = false;
let previousDragTimePos = 0;

  function handleViewportDrag ({ time, event, item }) {
    if (event.type === 'pointerdown') {
      if (item?.id !== 'selection') {
        return;
      }

      isDraggingMapViewport = true;
      previousDragTimePos = time;
    }
    else if (event.type === 'pointermove') {
      if (!isDraggingMapViewport) {
        return;
      }

      const delta = time - previousDragTimePos;
      const length = viewport.value.end - viewport.value.start;
      if (delta < 0) {
        viewport.value.start = Math.max(viewport.value.start + delta, totalRange.value.start);
        viewport.value.end = viewport.value.start + length;
      }
      else {
        viewport.value.end = Math.min(viewport.value.end + delta, totalRange.value.end);
        viewport.value.start = viewport.value.end - length;
      }
      previousDragTimePos = time;
    }
  }

    window.addEventListener('pointerup', () => {
    isDraggingMapViewport = false;
  }, { capture: true });

  function onMapWheel (event: WheelEvent) {
    timeline.value?.onWheel(event);
  }

const error = ref('')
const mouseHoverPosition = ref(null);
function onMousemoveTimeline ({ time }) {
    mouseHoverPosition.value = time;
  }
function onMouseleaveTimeline () {
    mouseHoverPosition.value = null;
  }
minT.value = 0
maxT.value = 10
onMounted(() => {})


const example = () => {
  inputCode.value = JSON.stringify(exampleData.value, null, 2)
  processJSON();
}
const processJSON = () => {
  error.value = ''
  timelineEvents.value = [] // clear previous events

  error.value = ''
  try {
    const data = JSON.parse(inputCode.value)
    if (!Array.isArray(data)) {
      throw new Error('JSON must be an array of objects')
    }
    const events = []
    const speakerSet = new Set()
    let minTimestamp = Infinity
    let maxTimestamp = -Infinity

    data.forEach(item => {
      if (
        item.recordType?.subType === 'FULL_SPEAKING_EVENT' &&
        item.payload
      ) {
        const payload = JSON.parse(item.payload)
        const speaker = payload.speakerName

        if (Array.isArray(payload.speakingFragments)) {
          payload.speakingFragments.forEach(frag => {
            const start = frag.startTimeMillis
            const end = frag.stopTimeMillis

            // Track min/max
            if (start < minTimestamp){
              minTimestamp = start
            } 
            if (end > maxTimestamp){
              maxTimestamp = end
            }
            speakerSet.add(speaker)
            events.push({
              group: speaker,
              start,
              end,
              type: 'range'
            })
          })
        }
      }
    })
    events.sort((a, b) => a.start - b.start)

    groups.value = Array.from(speakerSet).map(speaker => ({
      id: speaker,
      label: speaker
    }))

    timelineEvents.value = events
    console.log('Parsed timeline events:', events, maxTimestamp, minTimestamp)
    minT.value = minTimestamp
    maxT.value = maxTimestamp
    viewport.value = {
      start: minTimestamp,
      end: minTimestamp + 60000
    }
    totalRange.value = {
      start: minTimestamp,
      end: maxTimestamp
    }

  } catch (err) {
    error.value = 'Invalid JSON: ' + err.message
  }

}

</script>

<template>
  <div class="p-4">
    <textarea v-model="inputCode" rows="10" placeholder="Paste your code here..."></textarea>
    <br>
    <button @click="processJSON" class="">
      Parse
    </button>
    <button @click="example" class="">
      Example
    </button>

  </div>
  <Timeline
    :items="[...timelineEvents, { id: 'selection', type: 'background', start: viewport.start, end: viewport.end }]"
    :groups="groups"
    :viewportMin="totalRange.start"
    :viewportMax="totalRange.end"
    :initialViewportStart="totalRange.start"
    :initialViewportEnd="totalRange.end"
    :minViewportDuration="totalRange.end - totalRange.start"
    class="map"
    @pointermove="handleViewportDrag"
    @pointerdown="handleViewportDrag"
    @wheel="onMapWheel"
  />
   <Timeline
    ref="timeline"
    :groups="groups"
    :items="timelineEvents"
    :viewportMin="totalRange.start"
    :viewportMax="totalRange.end"
    :initialViewportStart="viewport.start"
    :initialViewportEnd="viewport.end"
    @mousemoveTimeline="onMousemoveTimeline"
    @mouseleaveTimeline="onMouseleaveTimeline"
    @changeViewport="viewport = $event"
    class="map"
  >

  </Timeline>
    {{ mouseHoverPosition ? new Date(mouseHoverPosition).toLocaleString() : '' }}


    <br>


</template>

<style lang="scss" scoped>
  .map {
    --group-items-height: .5em;
    --group-border-top: 0;
    --label-padding: 0;
    --group-padding-top: .1em;
    --group-padding-bottom: .1em;

    :deep(.group:first-of-type) {
      padding-top: 1rem;
    }

    :deep(.group:nth-of-type(3)) {
      padding-bottom: 1rem;
    }

    :deep(.background)  {
      --item-background: color-mix(in srgb, currentcolor, transparent 90%);

      cursor: pointer;
      z-index: 1;
    }

    :deep(.item)  {
      pointer-events: none;
      border-radius: 0px;
      height: 0.85em;
    }
  }
</style>