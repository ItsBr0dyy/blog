<script setup lang="ts">
import { nextTick, onBeforeUnmount, onMounted, ref, watch } from "vue";
import { useRoute } from "vitepress";

interface Tick {
	id: string;
	text: string;
	level: number;
	top: number;
	pos: number;
}

const THUMB = 26;

const route = useRoute();
const rail = ref<HTMLElement | null>(null);
const enabled = ref(false);
const scrollable = ref(false);
const progress = ref(0);
const label = ref("top");
const ticks = ref<Tick[]>([]);
const active = ref(-1);
const awake = ref(false);
const dragging = ref(false);

let hovering: boolean = false;
let maxScroll = 0;
let raf = 0;
let sleepTimer = 0;
let resizeObserver: ResizeObserver | undefined;

const clamp = (n: number, min = 0, max = 1) => Math.min(max, Math.max(min, n));

function wake() {
	awake.value = true;
	window.clearTimeout(sleepTimer);
	sleepTimer = window.setTimeout(() => {
		if (!dragging.value && !hovering) awake.value = false;
	}, 1400);
}

function collect() {
	const root = document.documentElement;
	maxScroll = Math.max(0, root.scrollHeight - window.innerHeight);
	scrollable.value = maxScroll > 120;

	const seen = new Set<string>();
	const found: Tick[] = [];
	document.querySelectorAll<HTMLElement>("h2[id], h3[id]").forEach((h) => {
		if (h.closest("nav, aside, header, footer") || seen.has(h.id)) return;
		seen.add(h.id);
		const top = h.getBoundingClientRect().top + window.scrollY;
		found.push({
			id: h.id,
			text: (h.textContent || "").replace(/[#\u200b]+$/, "").trim(),
			level: h.tagName === "H2" ? 2 : 3,
			top,
			pos: maxScroll ? clamp((top - 24) / maxScroll) : 0,
		});
	});
	ticks.value = found;
}

function update() {
	raf = 0;
	const y = window.scrollY;
	progress.value = maxScroll ? clamp(y / maxScroll) : 0;
	const pct = Math.round(progress.value * 100);
	label.value = pct <= 0 ? "top" : pct >= 100 ? "end" : `${pct}%`;

	let idx = -1;
	ticks.value.forEach((t, i) => {
		if (t.top - 140 <= y) idx = i;
	});
	active.value = idx;
}

function onScroll() {
	wake();
	if (!raf) raf = requestAnimationFrame(update);
}

function refresh() {
	collect();
	update();
}

function seek(e: PointerEvent) {
	if (!rail.value) return;
	const r = rail.value.getBoundingClientRect();
	const ratio = clamp((e.clientY - r.top - THUMB / 2) / (r.height - THUMB));
	window.scrollTo({ top: ratio * maxScroll, behavior: "instant" as ScrollBehavior });
}

function onDown(e: PointerEvent) {
	if ((e.target as HTMLElement).closest(".tick")) return;
	dragging.value = true;
	rail.value?.setPointerCapture(e.pointerId);
	seek(e);
	wake();
}

function onMove(e: PointerEvent) {
	if (dragging.value) seek(e);
}

function onUp(e: PointerEvent) {
	dragging.value = false;
	rail.value?.releasePointerCapture?.(e.pointerId);
	wake();
}

function go(t: Tick) {
	const reduce = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
	window.scrollTo({ top: t.top - 24, behavior: reduce ? "auto" : "smooth" });
}

function onEnter() {
	hovering = true;
	wake();
}

function onLeave() {
	hovering = false;
	wake();
}

onMounted(() => {
	enabled.value = window.matchMedia("(hover: hover) and (pointer: fine)").matches;
	if (!enabled.value) return;

	document.documentElement.classList.add("has-rail");
	refresh();
	window.addEventListener("scroll", onScroll, { passive: true });
	window.addEventListener("resize", refresh);
	resizeObserver = new ResizeObserver(refresh);
	resizeObserver.observe(document.body);
});

onBeforeUnmount(() => {
	document.documentElement.classList.remove("has-rail");
	window.removeEventListener("scroll", onScroll);
	window.removeEventListener("resize", refresh);
	resizeObserver?.disconnect();
	cancelAnimationFrame(raf);
	window.clearTimeout(sleepTimer);
});

watch(
	() => route.path,
	() => nextTick(() => setTimeout(refresh, 120)),
);
</script>

<template>
	<nav
		v-if="enabled && scrollable"
		ref="rail"
		class="scroll-rail"
		:class="{ awake, dragging }"
		:style="{ '--p': progress }"
		aria-label="Page sections"
		@pointerdown="onDown"
		@pointermove="onMove"
		@pointerup="onUp"
		@pointercancel="onUp"
        @pointerenter="onEnter"
        @pointerleave="onLeave"
	>
		<span class="track"></span>
		<span class="fill"></span>

		<button
			v-for="(t, i) in ticks"
			:key="t.id"
			type="button"
			class="tick"
			:class="{ passed: i <= active, current: i === active, minor: t.level === 3 }"
			:style="{ '--t': t.pos }"
			:aria-label="`Jump to ${t.text}`"
			@click="go(t)"
		>
			<span class="tick-label">{{ t.text }}</span>
		</button>

		<span class="knob"></span>
		<span class="thumb" aria-hidden="true">{{ label }}</span>
	</nav>
</template>

<style>
html.has-rail {
	scrollbar-width: none;
}

html.has-rail::-webkit-scrollbar {
	display: none;
}

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
	right: 4px;
	bottom: 20px;
	z-index: 60;
	width: 22px;
	cursor: pointer;
	touch-action: none;
	user-select: none;
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
	left: 0;
	top: calc(var(--t) * (100% - var(--thumb)) + var(--thumb) / 2 - 8px);
	width: 100%;
	height: 16px;
	padding: 0;
	border: 0;
	background: none;
	cursor: pointer;
}

.tick::before {
	content: "";
	position: absolute;
	top: 50%;
	left: 50%;
	width: 7px;
	height: 7px;
	border-radius: 50%;
	background: var(--muted);
	box-shadow: 0 0 0 2px var(--bg);
	transform: translate(-50%, -50%);
	transition: transform 0.15s, background 0.15s;
}

.tick.minor::before {
	width: 5px;
	height: 5px;
}

.tick.passed::before {
	background: var(--accent);
}

.tick.current::before {
	box-shadow: 0 0 0 2px var(--bg), 0 0 0 4px var(--bubble);
}

.tick:hover::before,
.tick:focus-visible::before {
	transform: translate(-50%, -50%) scale(1.5);
}

.tick:focus-visible {
	outline: 2px solid var(--accent);
	outline-offset: -2px;
}

.tick-label {
	position: absolute;
	top: 50%;
	right: calc(100% + 4px);
	max-width: 240px;
	padding: 4px 10px;
	border-radius: 6px 6px 0 6px;
	background: var(--bubble);
	color: var(--bubbletext);
	font-family: var(--font-sans);
	font-size: 0.8125rem;
	font-weight: 400;
	line-height: 1.4;
	text-align: left;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
	pointer-events: none;
	opacity: 0;
	transform: translateY(-50%);
	z-index: 2;
}

.tick:hover .tick-label,
.tick:focus-visible .tick-label {
	opacity: 1;
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
	transition: transform 0.15s;
	pointer-events: none;
}

.dragging .knob {
	transform: translate(-50%, -50%) scale(1.35);
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
	pointer-events: none;
	opacity: 0;
	transition: opacity 0.25s;
}

.awake .thumb {
	opacity: 1;
}

.scroll-rail {
	opacity: 0.55;
	transition: opacity 0.25s;
}

.scroll-rail.awake {
	opacity: 1;
}

@media (max-width: 640px) {
	.thumb,
	.tick-label {
		display: none;
	}
}

@media (prefers-reduced-motion: reduce) {
	.scroll-rail,
	.thumb,
	.knob,
	.tick::before {
		transition: none;
	}
}
</style>
