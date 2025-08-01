<script setup lang="ts">
import type CodeMirror from 'codemirror'
import { nextTick, onMounted, toRefs, useTemplateRef, watchEffect } from 'vue'
import { guessCodemirrowMode, syncEditorScrolls, syncScrollListeners, useCodeMirror } from '~/composables/codemirror'
import { settings } from '~/state/settings'
import { calculateDiffWithWorker } from '~/worker/diff'

const props = defineProps<{
  from: string
  to: string
  oneColumn: boolean
  diff: boolean
}>()

const { from, to } = toRefs(props)

const fromEl = useTemplateRef('fromEl')
const toEl = useTemplateRef('toEl')

let cm1: CodeMirror.Editor
let cm2: CodeMirror.Editor

onMounted(() => {
  cm1 = useCodeMirror(
    fromEl,
    from,
    {
      mode: 'javascript',
      readOnly: true,
      lineNumbers: true,
    },
  )

  cm2 = useCodeMirror(
    toEl,
    to,
    {
      mode: 'javascript',
      readOnly: true,
      lineNumbers: true,
    },
  )

  syncScrollListeners(cm1, cm2)

  watchEffect(() => {
    cm1.setOption('lineWrapping', settings.value.codeviewerLineWrap)
    cm2.setOption('lineWrapping', settings.value.codeviewerLineWrap)
  })

  watchEffect(async () => {
    cm1.getWrapperElement().style.display = props.oneColumn ? 'none' : ''
    if (!props.oneColumn) {
      await nextTick()
      // Force sync to current scroll
      cm1.refresh()
      syncEditorScrolls(cm2, cm1)
    }
  })

  watchEffect(async () => {
    const l = from.value
    const r = to.value
    const diffEnabled = props.diff

    cm1.setOption('mode', guessCodemirrowMode(l))
    cm2.setOption('mode', guessCodemirrowMode(r))

    await nextTick()

    cm1.startOperation()
    cm2.startOperation()

    // clean up marks
    cm1.getAllMarks().forEach(i => i.clear())
    cm2.getAllMarks().forEach(i => i.clear())
    for (let i = 0; i < cm1.lineCount() + 2; i++)
      cm1.removeLineClass(i, 'background', 'diff-removed')
    for (let i = 0; i < cm2.lineCount() + 2; i++)
      cm2.removeLineClass(i, 'background', 'diff-added')

    if (diffEnabled && from.value) {
      const changes = await calculateDiffWithWorker(l, r)

      const addedLines = new Set()
      const removedLines = new Set()

      let indexL = 0
      let indexR = 0
      changes.forEach(([type, change]) => {
        if (type === 1) {
          const start = cm2.posFromIndex(indexR)
          indexR += change.length
          const end = cm2.posFromIndex(indexR)
          cm2.markText(start, end, { className: 'diff-added-inline' })
          for (let i = start.line; i <= end.line; i++) addedLines.add(i)
        }
        else if (type === -1) {
          const start = cm1.posFromIndex(indexL)
          indexL += change.length
          const end = cm1.posFromIndex(indexL)
          cm1.markText(start, end, { className: 'diff-removed-inline' })
          for (let i = start.line; i <= end.line; i++) removedLines.add(i)
        }
        else {
          indexL += change.length
          indexR += change.length
        }
      })

      Array.from(removedLines).forEach(i =>
        cm1.addLineClass(i, 'background', 'diff-removed'),
      )
      Array.from(addedLines).forEach(i =>
        cm2.addLineClass(i, 'background', 'diff-added'),
      )
    }

    cm1.endOperation()
    cm2.endOperation()
  })
})

function _onUpdate(size: number) {
  // Refresh sizes
  cm1?.refresh()
  cm2?.refresh()
  if (props.oneColumn)
    return
  settings.value.codeviewerDiffPanelSize = size
}
</script>

<template>
  <div h-full w-full of-auto :class="oneColumn ? 'flex' : 'grid grid-cols-2'" style="overscroll-behavior: contain">
    <div v-show="!oneColumn" ref="fromEl" h-inherit />
    <div ref="toEl" h-inherit :class="oneColumn ? 'w-full' : ''" />
  </div>
</template>

<style lang="postcss">
.diff-added {
  --at-apply: bg-green-400/15;
}
.diff-removed {
  --at-apply: bg-red-400/15;
}
.diff-added-inline {
  --at-apply: bg-green-400/30;
}
.diff-removed-inline {
  --at-apply: bg-red-400/30;
}
</style>
