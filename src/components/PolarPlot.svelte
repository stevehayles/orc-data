<svelte:options accessors />

<script>
import { scaleLinear } from 'd3-scale';
import { symbol, symbolCircle } from 'd3-shape';

import VppCurves from './VppCurves.svelte';
import { DEG2RAD } from '../util.js';
export let boats = [];

let windowInnerHeight, windowInnerWidth;
let container;
let tooltip = null;
let width = 300;
$: if (windowInnerWidth && container) {
    width = container.offsetWidth;
}
let height = 700;
$: if (windowInnerHeight && windowInnerWidth && container) {
    height = Math.min(width * 1.8, windowInnerHeight - 60);
}
$: radius = Math.min(height / 1.8 - 20, width) - 25;
$: rScale.range([0, radius]);

// Scale for the r axis, mapping SOG to plot coordinates
$: rScale = scaleLinear().domain([0, 15]).range([0, radius]);

const sogs = [2, 4, 6, 8, 10, 12, 14, 16, 18, 20];
const maxSogLabel = 20;
const angles = [0, 45, 52, 60, 75, 90, 110, 120, 135, 150, 165];

let highlight = undefined;
export const hover = (_newHighlight) => {
    highlight = _newHighlight;
};
$: container?.offsetWidth;
$: centerY = radius * 0.65;

function onMouseMove(event) {
    const xScreen = event.offsetX;
    const yScreen = event.offsetY;

    const x = xScreen - 10;
    const y = yScreen - centerY;

    const r = Math.sqrt(x * x + y * y);

    const maxTooltipR = Math.max(
        radius,
        ...boats.flatMap((boat) =>
            boat
                ? boat.vpp.angles.flatMap((angle) =>
                      boat.vpp[angle].map((sog) => rScale(sog))
                  )
                : []
        )
    );

    if (r > maxTooltipR * 1.05) {
        tooltip = null;
        return;
    }

    const angleRad = Math.atan2(x, -y);
    const twa = ((angleRad * 180) / Math.PI + 360) % 360;

    if (twa > 180) {
        tooltip = null;
        return;
    }

    const mouseSog = rScale.invert(r);
    const tws = nearestTwsAtMouse(boats[0], twa, mouseSog);

    tooltip = {
        xScreen,
        yScreen,
        twa,
        tws
    };
}

function getBoatValue(boat, twa, sog) {
    if (!boat) return null;

    const vpp = boat.vpp;
    const angles = vpp.angles;
    const speeds = vpp.speeds;

    // --- find surrounding angles
    let a1 = angles[0], a2 = angles[angles.length - 1];
    for (let i = 0; i < angles.length - 1; i++) {
        if (twa >= angles[i] && twa <= angles[i + 1]) {
            a1 = angles[i];
            a2 = angles[i + 1];
            break;
        }
    }

    const t = (twa - a1) / (a2 - a1 || 1);

    // --- find surrounding wind speeds
    let s1i = 0, s2i = speeds.length - 1;
    for (let i = 0; i < speeds.length - 1; i++) {
        if (sog >= speeds[i] && sog <= speeds[i + 1]) {
            s1i = i;
            s2i = i + 1;
            break;
        }
    }

    const s1 = speeds[s1i];
    const s2 = speeds[s2i];
    const u = (sog - s1) / (s2 - s1 || 1);

    // --- values at 4 corners
    const v11 = vpp[a1]?.[s1i];
    const v12 = vpp[a1]?.[s2i];
    const v21 = vpp[a2]?.[s1i];
    const v22 = vpp[a2]?.[s2i];

    if ([v11, v12, v21, v22].some(v => v == null)) return null;

    // --- bilinear interpolation
    const v1 = v11 + u * (v12 - v11);
    const v2 = v21 + u * (v22 - v21);

    return v1 + t * (v2 - v1);
}

function getBoatName(boat) {
    return boat?.name || "—";
}

function getTwsIndex(boat, sog) {
    const speeds = boat?.vpp?.speeds;
    if (!speeds) return 0;

    return speeds.reduce((best, s, i) =>
        Math.abs(s - sog) < Math.abs(speeds[best] - sog) ? i : best,
        0
    );
}

function interpolateAtTwaForTws(boat, twa, twsIndex) {
    if (!boat) return null;

    const vpp = boat.vpp;
    const angles = vpp.angles;

    if (twa <= angles[0]) return vpp[angles[0]]?.[twsIndex] ?? null;
    if (twa >= angles[angles.length - 1]) return vpp[angles[angles.length - 1]]?.[twsIndex] ?? null;

    for (let i = 0; i < angles.length - 1; i++) {
        const a1 = angles[i];
        const a2 = angles[i + 1];

        if (twa >= a1 && twa <= a2) {
            const v1 = vpp[a1]?.[twsIndex];
            const v2 = vpp[a2]?.[twsIndex];

            if (v1 == null || v2 == null) return null;

            const t = (twa - a1) / (a2 - a1);
            return v1 + t * (v2 - v1);
        }
    }
    return null;
}

function nearestTwsAtMouse(boat, twa, mouseSog) {
    if (!boat) return null;

    const speeds = boat.vpp.speeds;

    let bestTws = null;
    let bestError = Infinity;

    for (let i = 0; i < speeds.length; i++) {
        const sog = interpolateAtTwaForTws(boat, twa, i);
        if (sog == null) continue;

        const error = Math.abs(sog - mouseSog);

        if (error < bestError) {
            bestError = error;
            bestTws = speeds[i];
        }
    }

    return bestTws;
}

function getBoatValueAtTws(boat, twa, tws) {
    if (!boat || tws == null) return null;

    const idx = boat.vpp.speeds.indexOf(tws);
    if (idx < 0) return null;

    return interpolateAtTwaForTws(boat, twa, idx);
}
</script>

<svelte:window bind:innerHeight={windowInnerHeight} bind:innerWidth={windowInnerWidth} />
<div bind:this={container}>
    <svg {width} {height}
        on:mousemove={onMouseMove}
        on:mouseleave={() => tooltip = null}>

        <g transform="translate(10, {centerY})">
            <!-- Speed rings -->
            {#each sogs as sog}
                <g class="r axis sog-{sog}">
                    <circle r={rScale(sog)}></circle>
                    {#if sog <= maxSogLabel}
                        <text y={-rScale(sog) - 2} transform="rotate(25)" text-anchor="middle">
                            {sog} kts
                        </text>
                    {/if}
                </g>
            {/each}
            <!-- Course lines -->
            {#each angles as angle}
                <g class="a axis" transform="rotate({angle - 90})">
                    <line x1={rScale(0)} x2={rScale(15)} />
                    <text class="xlabel" x={rScale(10) + 5} y={0} text-anchor="start" alignment-baseline="middle">
                        {angle}°
                    </text>
                </g>
            {/each}
            {#each boats as boat, index}
                {#if boat}
                    <VppCurves vpp={boat.vpp} {index} {rScale} />
                {/if}
            {/each}
            {#if highlight}
                <path
                    class="highlight tws-{highlight.tws}"
                    d={symbol(symbolCircle, 80)()}
                    transform="translate({rScale(highlight.sog) * Math.sin(highlight.cog * DEG2RAD)}, {rScale(
                        highlight.sog,
                    ) * -Math.cos(highlight.cog * DEG2RAD)})"
                    transition="400ms"
                    stroke-width="1" />
            {/if}
        </g>
        
        {#if tooltip}
            {@const a = getBoatValueAtTws(boats[0], tooltip.twa, tooltip.tws)}
            {@const b = getBoatValueAtTws(boats[1], tooltip.twa, tooltip.tws)}
            {@const delta = (a != null && b != null) ? (a - b) : null}

            <g transform="translate({tooltip.xScreen + 15}, {tooltip.yScreen - 75})">
                <rect width="210" height="95" fill="white" stroke="black" rx="6" />

                <text x="8" y="16" font-weight="bold">
                    {tooltip.twa.toFixed(0)}°
                    {#if tooltip.tws != null}
                        @ {tooltip.tws} kts
                    {/if}
                </text>

                <text x="8" y="36" fill="steelblue">
                    {boats[0]?.name || 'Boat A'}:
                    {a != null ? `${a.toFixed(2)} kts` : '–'}
                </text>

                <text x="8" y="56" fill="crimson">
                    {boats[1]?.name || 'Boat B'}:
                    {b != null ? `${b.toFixed(2)} kts` : '–'}
                </text>

                {#if delta != null}
                    <text x="8" y="76"
                        fill={delta > 0 ? 'green' : delta < 0 ? 'red' : 'black'}
                        font-weight="bold">
                        Δ: {delta > 0 ? '+' : ''}{delta.toFixed(2)} kts
                    </text>
                {/if}
            </g>
        {/if}
    </svg>
</div>
