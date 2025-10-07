<script setup lang="ts">
import type { DraggableEvent, SortableEvent } from 'vue-draggable-plus'
import { useCssVar, useElementSize } from '@vueuse/core'
import { computed, inject, ref, onMounted, watch, nextTick } from 'vue'
import { VueDraggable } from 'vue-draggable-plus'
import useElementGrabbing from '~/composables/useElementGrabbing'
import { LineContextKey } from '~/utils/symbols'

const {
  fluid = false,
} = defineProps<{
  fluid?: boolean
}>()

const el = ref()
const line = ref()
const sizeFactor = computed(() => Number.parseInt(useCssVar('--base-size', el).value ?? '1'))
const { width: branchLength } = useElementSize(line)

const branch = defineModel<Branch>({ required: true })
const emphasize = ref(false)
const { grab, release } = useElementGrabbing((event) => {
  emphasize.value = ['STOP', 'SPACER'].includes(event.type ?? '')
})

const elements = computed({
  get: () => branch.value.$branch.elements ?? [],
  set: val => branch.value.$branch.elements = val,
})

const lineContext = inject<LineContext>(LineContextKey)!

const elementSpacing = computed(() => `${branch.value.$branch.elementSpacing}em`)
const leftMargin = computed(() => `${branch.value.$branch.marginLeft || 0}em`)
const rightMargin = computed(() => `${branch.value.$branch.marginRight || 0}em`)

const color = computed(() => lineContext?.color.value ?? '#000000')
const lineWidth = computed(() => lineContext.lineThickness.value)
const lineOffset = computed(() => sizeFactor.value * lineWidth.value * 16 / 2)

// --- Closed adjacent stops handling ---------------------------------------
// Pixel padding inside each dot so the grey segment ne touche pas exactement l'intérieur du cercle.
// Mettre à 0 pour coller au bord interne.
const CLOSED_SEGMENT_INNER_PADDING = 0 // plus de retrait interne
const CLOSED_SEGMENT_EXPAND = 1 // px d'expansion vers l'extérieur pour garantir aucune lacune
interface ClosedSegment { from: number; to: number }
const closedSegments = ref<ClosedSegment[]>([])

function computeClosedSegments() {
  closedSegments.value = []
  const branchEl: HTMLElement | null = el.value as HTMLElement | null
  const lineEl: HTMLElement | null = (branchEl?.querySelector('.line') as HTMLElement | null)
  if (!branchEl || !lineEl) return

  const lineRect = lineEl.getBoundingClientRect()
  const domStops: HTMLElement[] = Array.from(branchEl.querySelectorAll('.branch-elements > .stop-wrapper')) as HTMLElement[]
  if (domStops.length === 0) return

  // Compute center + approximate dot radius from first dot element
  let dotRadius = 6 // fallback px
  const firstDot = domStops[0]?.querySelector('.dot') as HTMLElement | null
  if (firstDot) {
    const r = firstDot.getBoundingClientRect()
    dotRadius = Math.min(r.width, r.height) / 2
  }

  const dotBoxes = domStops.map(stopEl => {
    const dot = stopEl.querySelector('.dot') as HTMLElement | null
    const target = dot ?? stopEl
    const rect = target.getBoundingClientRect()
    return {
      left: rect.left - lineRect.left,
      right: rect.right - lineRect.left,
    }
  })

  const rawSegments: ClosedSegment[] = []
  const elementsModel = elements.value
  // Nouvelle logique: pour chaque stop fermé, on crée un segment gris vers chaque voisin couvrant
  // TOUT l'espace entre les deux stops (bord interne à bord interne). Ainsi un stop fermé relié à un stop
  // ouvert produit un segment gris complet jusqu'à l'autre stop.
  for (let i = 0; i < elementsModel.length; i++) {
    const current = elementsModel[i] as any
    if (!(current?.$stop) || !current.$stop.closed) continue
    const box = dotBoxes[i]
    if (!box) continue

    // Segment vers la gauche (jusqu'au bord interne du précédent stop)
    if (i > 0) {
      const prevBox = dotBoxes[i - 1]
      if (prevBox) {
        // Bord interne précédent -> bord interne courant
        let from = prevBox.right
        let to = box.left
        if (to > from) {
          from += CLOSED_SEGMENT_INNER_PADDING - CLOSED_SEGMENT_EXPAND
          to -= CLOSED_SEGMENT_INNER_PADDING - CLOSED_SEGMENT_EXPAND
          if (to > from) rawSegments.push({ from, to })
        }
      }
    }

    // Segment vers la droite (jusqu'au bord interne du prochain stop)
    if (i < elementsModel.length - 1) {
      const nextEl = elementsModel[i + 1] as any
      const nextBox = dotBoxes[i + 1]
      if (nextBox) {
        let from = box.right
        let to = nextBox.left
        if (to > from) {
          from += CLOSED_SEGMENT_INNER_PADDING - CLOSED_SEGMENT_EXPAND
          to -= CLOSED_SEGMENT_INNER_PADDING - CLOSED_SEGMENT_EXPAND
          if (to > from) rawSegments.push({ from, to })
        }
      }
    }
  }

  // Fusion des segments qui se touchent / se chevauchent (cas de deux stops fermés adjacents :
  // leurs segments se rejoignent automatiquement, recréant un segment continu comme avant).
  rawSegments.sort((a, b) => a.from - b.from)
  for (const seg of rawSegments) {
    const last = closedSegments.value[closedSegments.value.length - 1]
    if (!last || seg.from > last.to + 1) {
      closedSegments.value.push({ ...seg })
    } else {
      last.to = Math.max(last.to, seg.to)
    }
  }
}

// --- Pointillé personnalisé (cercles) -------------------------------------
// Rayon des points gris basé sur l'épaisseur réduite utilisée auparavant
// Facteur 1.25 pour points plus gros
const dotRadius = computed(() => (Math.max(0.4, lineWidth.value * .5) * 8) / 2 * 1.25)
// Espacement cible entre centres ~ 1.8 * diamètre pour bien aérer
function dotCount(seg: { from: number; to: number }) {
  const span = Math.min(branchLength.value, seg.to) - Math.max(0, seg.from)
  if (span <= 0) return 0
  // espacement basé sur diamètre * 1.7 (légèrement plus serré car points plus gros)
  const step = dotRadius.value * 2 * 1.7
  return Math.max(1, Math.floor(span / step))
}
function dotX(seg: { from: number; to: number }, n: number) {
  const start = Math.max(0, seg.from)
  const end = Math.min(branchLength.value, seg.to)
  const count = dotCount(seg)
  if (count <= 1) return (start + end) / 2
  return start + (end - start) * (n - 0.5) / count
}

// Recompute on mount, when branch length changes, or when elements / their closed state change
onMounted(() => {
  nextTick(() => computeClosedSegments())
})

watch([branchLength, elements, () => elements.value.map(e => (e as any)?.$stop?.closed).join(',')], () => {
  nextTick(() => computeClosedSegments())
})

/* Simply because the lib is muffin broken */
function moveOut(event: DraggableEvent<BranchElement>) {
  const el = event.from
  for (let i = 0; i < el.children.length; i++) {
    const child = el.children.item(i)!
    if (child.hasAttribute('data-id') && child.getAttribute('data-id') === event.data.id) {
      el.removeChild(child)
    }
  }
}
</script>

<template>
  <div
    ref="el"
    class="branch-wrapper" :class="{
      empty: elements?.length === 0,
      fluid,
      negativeLeftMargin: (branch.$branch.marginLeft ?? 0) < 0,
      negativeRightMargin: (branch.$branch.marginRight ?? 0) < 0,
      positiveLeftMargin: (branch.$branch.marginLeft ?? 0) > 0,
      positiveRightMargin: (branch.$branch.marginRight ?? 0) > 0,
    }"
  >
    <VueDraggable
      v-model="elements"
      :animation="150"
      class="branch-elements open"
      :class="{ emphasize }"
      group="branchElements"
      ghost-class="branch-element-ghost"
      :swap-threshold=".75"
      handle=".branch-element-handle"
      @remove="(e: SortableEvent) => moveOut(e as DraggableEvent<BranchElement>)"
      @start="grab('STOP')"
      @end="release()"
    >
      <BranchElement
        v-for="(element, i) in elements"
        :key="element.id"
        v-model="elements[i]"
        :data-id="element.id"
        :reverse="branch.$branch.invertedElements"
      />
    </VueDraggable>
    <div ref="line" class="line">
      <svg width="100%" :height="`${lineWidth}em`" overflow="visible">
        <g :transform="`translate(0 ${lineOffset})`">
          <SvgLine
            :path="`M 0 0 L ${branchLength} 0`"
            :color="color"
            :line-width="lineWidth"
            :striped="lineContext.lineStyle.value === 'STRIPED'"
          />
          <!-- Grey thinner overlay segments between two consecutive closed stops -->
          <!-- Masquage (stroke blanc pleine épaisseur) pour couvrir la ligne colorée -->
          <template v-for="(seg, i) in closedSegments" :key="`closed-${i}-${seg.from}-${seg.to}`">
            <!-- Masque blanc pleine épaisseur -->
            <SvgLine
              :path="`M ${Math.max(0, seg.from - 1)} 0 L ${Math.min(branchLength, seg.to + 1)} 0`"
              color="#ffffff"
              :line-width="lineWidth"
              :striped="false"
            />
            <!-- Suite de points gris -->
            <g v-if="(Math.min(branchLength, seg.to) - Math.max(0, seg.from)) > 0">
              <!-- Calcul simple: on définit un pas cible basé sur l'épaisseur -->
              <template v-for="n in dotCount(seg)" :key="`dot-${i}-${n}`">
                <circle :cx="dotX(seg, n)" cy="0" :r="dotRadius" fill="#9ca3af" />
              </template>
            </g>
          </template>
        </g>
      </svg>
    </div>
  </div>
</template>

<style lang="scss">
.element-ghost {
  opacity: .5;
}
</style>

<style scoped lang="scss">
.branch-wrapper {
  .debug & {
    outline: 1px solid orange;
    outline-offset: 1px;
  }

  position: relative;
  z-index: 2;

  &.fluid {
    flex-grow: 1;
  }

  &.empty {
    min-width: 3em;
  }

  &.negativeLeftMargin {
    margin-left: v-bind(leftMargin);
  }

  &.negativeRightMargin {
    margin-right: v-bind(rightMargin);
  }

  &.positiveLeftMargin {
    padding-left: v-bind(leftMargin);
  }

  &.positiveRightMargin {
    padding-right: v-bind(rightMargin);
  }

  .section-element + .section-element & {
     .line {
       padding-left: 0;
       clip: rect(auto, calc(v-bind(branchLength) * 1px + 1em), auto, 0);
     }
   }

  .section-element:not(:last-child) & {
    .line {
      padding-right: 0;
      clip: rect(auto, calc(v-bind(branchLength) * 1px + .5em), auto, calc(v-bind(lineWidth) * -.25em));
    }
  }

  // both
  .section-element:not(:last-child):has(+ .section-element) .section-element:not(:first-child) &,
  .section-element + .section-element .section-element:not(:last-child) &,
  .section-element + .section-element:not(:last-child) &
  {
    .line {
      padding: 0;
      clip: rect(auto, calc(v-bind(branchLength) * 1px), auto, 0);
    }
  }
}

.branch-elements {
  position: relative;
  min-height: 4em;
  display: flex;
  z-index: 10;
  flex-grow: 1;
  flex-direction: row;
  justify-content: space-evenly;
  align-items: center;
  gap: calc(v-bind(elementSpacing));
  pointer-events: fill;

  &:after {
    position: absolute;
    pointer-events: none;
    content: '';
    top: 50%;
    left: 0;
    right: 0;
    bottom: 0;
    min-height: 5em;
    transform: translateY(-50%);
    background-color: transparent;
    border: 2px dashed transparent;
    border-radius: .25em;
    padding: 0 2em;
    margin: 0 -2em;
    transition: background-color .2s ease, border-color .2s ease;
  }
}

.emphasize {
  --border-color: var(--p-blue-400);
  padding: 0 2em;
  margin: 0 -2em;

  &:after {
    background: color-mix(in srgb, var(--border-color), transparent 85%);
    border-color: var(--border-color);
    padding: 0;
    margin: 0;
  }
}

.line {
  position: absolute;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  width: 100%;
  // Fine‑tuned: small left overshoot but not enough to show the colored line inside the stop.
  padding: 0; 
  margin-left: calc(v-bind(lineWidth) * -.18em); // reduced overshoot
  width: calc(100% + calc(v-bind(lineWidth) * .18em));
  // Soft mask to prevent any colored stroke bleed into left stop; using inset shadow via clip-path fallback.
  /* If further precision needed, consider wrapping SVG in an extra div with overflow: hidden and left padding */
  z-index: -1;
}
</style>
