<script setup lang="ts">
import { useRoute } from 'vitepress'
import { nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

interface Tick {
	id: string
	level: number
	top: number
	pos: number
}

const route = useRoute()
const enabled = ref(false)
const scrollable = ref(false)
const progress = ref(0)
const label = ref('top')
const ticks = ref<Tick[]>([])
const active = ref(-1)
const awake = ref(false)

let maxScroll = 0
let raf = 0
let sleepTimer = 0
let resizeObserver: ResizeObserver | undefined

const clamp = (n: number, min = 0, max = 1) => Math.min(max, Math.max(min, n))

function wake() {
	awake.value = true
	window.clearTimeout(sleepTimer)
	sleepTimer = window.setTimeout(() => {
		awake.value = false
	}, 1400)
}

function collect() {
	const root = document.documentElement
	maxScroll = Math.max(0, root.scrollHeight - window.innerHeight)
	scrollable.value = maxScroll > 120

	const seen = new Set<string>()
	const found: Tick[] = []
	document.querySelectorAll<HTMLElement>('h2[id], h3[id]').forEach((h) => {
		if (h.closest('nav, aside, header, footer') || seen.has(h.id)) return
		seen.add(h.id)
		const top = h.getBoundingClientRect().top + window.scrollY
		found.push({
			id: h.id,
			level: h.tagName === 'H2' ? 2 : 3,
			top,
			pos: maxScroll ? clamp((top - 24) / maxScroll) : 0,
		})
	})
	ticks.value = found
}

function update() {
	raf = 0
	const y = window.scrollY
	progress.value = maxScroll ? clamp(y / maxScroll) : 0
	const pct = Math.round(progress.value * 100)
	label.value = pct <= 0 ? 'top' : pct >= 100 ? 'end' : `${pct}%`

	let idx = -1
	ticks.value.forEach((t, i) => {
		if (t.top - 140 <= y) idx = i
	})
	active.value = idx
}

function onScroll() {
	wake()
	if (!raf) raf = requestAnimationFrame(update)
}

function refresh() {
	collect()
	update()
}

onMounted(() => {
	enabled.value = window.matchMedia(
		'(hover: hover) and (pointer: fine)',
	).matches
	if (!enabled.value) return

	refresh()
	window.addEventListener('scroll', onScroll, { passive: true })
	window.addEventListener('resize', refresh)
	resizeObserver = new ResizeObserver(refresh)
	resizeObserver.observe(document.body)
})

onBeforeUnmount(() => {
	window.removeEventListener('scroll', onScroll)
	window.removeEventListener('resize', refresh)
	resizeObserver?.disconnect()
	cancelAnimationFrame(raf)
	window.clearTimeout(sleepTimer)
})

watch(
	() => route.path,
	() => nextTick(() => setTimeout(refresh, 120)),
)
</script>

<template>
	<div
		v-if="enabled && scrollable"
		class="scroll-rail"
		:class="{ awake }"
		:style="{ '--p': progress }"
		aria-hidden="true"
	>
		<span class="track"></span>
		<span class="fill"></span>

		<span
			v-for="(t, i) in ticks"
			:key="t.id"
			class="tick"
			:class="{ passed: i <= active, current: i === active, minor: t.level === 3 }"
			:style="{ '--t': t.pos }"
		></span>

		<span class="knob"></span>
		<span class="thumb">{{ label }}</span>
	</div>
</template>

<style>
pre,
table {
	scrollbar-width: thin;
	scrollbar-color: var(--rule) transparent;
}
</style>

<style scoped>
.scroll-rail {
	--thumb: 26px;
	position: fixed;
	top: 20px;
	/* Inset from the edge so it never sits under an overlay scrollbar (macOS etc.) */
	right: 14px;
	bottom: 20px;
	z-index: 60;
	width: 22px;
	pointer-events: none;
	user-select: none;
	opacity: 0.55;
	transition: opacity 0.25s;
}

.scroll-rail.awake {
	opacity: 1;
}

.track,
.fill {
	position: absolute;
	left: 50%;
	width: 4px;
	margin-left: -2px;
	background-size: 4px 8px;
	background-repeat: repeat-y;
}

.track {
	top: 0;
	bottom: 0;
	background-image: radial-gradient(circle at 50% 50%, var(--rule) 1.25px, transparent 1.75px);
}

.fill {
	top: 0;
	height: calc(var(--p) * (100% - var(--thumb)) + var(--thumb) / 2);
	background-image: radial-gradient(circle at 50% 50%, var(--accent) 1.25px, transparent 1.75px);
}

.tick {
	position: absolute;
	left: 50%;
	top: calc(var(--t) * (100% - var(--thumb)) + var(--thumb) / 2);
	width: 7px;
	height: 7px;
	border-radius: 50%;
	background: var(--muted);
	box-shadow: 0 0 0 2px var(--bg);
	transform: translate(-50%, -50%);
	transition: background 0.15s, box-shadow 0.15s;
}

.tick.minor {
	width: 5px;
	height: 5px;
}

.tick.passed {
	background: var(--accent);
}

.tick.current {
	box-shadow: 0 0 0 2px var(--bg), 0 0 0 4px var(--bubble);
}

.knob {
	position: absolute;
	left: 50%;
	top: calc(var(--p) * (100% - var(--thumb)) + var(--thumb) / 2);
	width: 11px;
	height: 11px;
	border-radius: 50%;
	background: var(--accent);
	box-shadow: 0 0 0 3px var(--bg);
	transform: translate(-50%, -50%);
}

.thumb {
	position: absolute;
	right: 20px;
	top: calc(var(--p) * (100% - var(--thumb)));
	min-width: 48px;
	height: var(--thumb);
	padding: 0 10px;
	border-radius: 6px 6px 0 6px;
	background: var(--bubble);
	color: var(--bubbletext);
	font-family: var(--font-mono);
	font-size: 0.75rem;
	line-height: var(--thumb);
	text-align: center;
	font-variant-numeric: tabular-nums;
	opacity: 0;
	transition: opacity 0.25s;
}

.awake .thumb {
	opacity: 1;
}

@media (max-width: 640px) {
	.thumb {
		display: none;
	}
}

@media (prefers-reduced-motion: reduce) {
	.scroll-rail,
	.thumb,
	.tick {
		transition: none;
	}
}
</style>
