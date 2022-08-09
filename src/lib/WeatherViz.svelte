<script>
  import { range } from 'd3-array'
  import { scaleLinear } from 'd3-scale'
  import { interpolateSpectral } from 'd3-scale-chromatic'
  import Spline from 'cubic-spline'
  import SunCalc from 'SunCalc'

  export let data
  export let latitude
  export let longitude

  $: console.log(data)

  $: indices = range(data.hourly.time.length)

  $: times = new Spline(
    indices,
    data.hourly.time.map(d => new Date(d)).valueOf()
  )

  $: windDirection = new Spline(
    indices,
    data.hourly.winddirection_10m.reduce((acc, cur, i) => {
      if (i === 0) {
        acc.push(cur > 180 ? cur - 360 : cur)
      } else {
        const prev = acc[i - 1]

        if (cur - prev > 180) {
          acc.push(cur - 360)
        } else {
          acc.push(cur)
        }
      }

      return acc
    }, [])
  )

  $: windSpeed = new Spline(indices, data.hourly.windspeed_10m)

  $: temperature = new Spline(indices, data.hourly.temperature_2m)

  const margin = {
    top: 40,
    right: 40,
    bottom: 40,
    left: 40
  }

  const width = 800
  const height = 800

  const w = width - margin.left - margin.right
  const h = height - margin.top - margin.bottom

  const center = [w / 2, h / 2]

  let index = 0

  const BEAUFORT = [2, 6, 12, 20, 29, 39, 50, 62, 75, 89, 103, 118]

  const temp = scaleLinear().domain([-20, 0, 40]).range([1, 0.5, 0]).clamp(true)
  const b = scaleLinear()
    .domain([0, 120])
    .range([50, w / 2 - 20])
  let x = scaleLinear()
    .domain([0, width])
    .range([0, data.hourly.time.length - 1 || 0])

  const trails = range(0, 4, 0.2)

  const closest = goal =>
    BEAUFORT.reduce((prev, cur) =>
      Math.abs(cur - goal) < Math.abs(prev - goal) ? cur : prev
    )

  $: localTime = () => {
    var date = new Date(new Date(times.at(index)))
    const userTimezoneOffset = date.getTimezoneOffset() * 60000
    return new Date(date.getTime() - userTimezoneOffset)
  }

  $: sunPosition = SunCalc.getPosition(localTime(), latitude, longitude)

  const azimuth = scaleLinear().domain([-Math.PI, Math.PI]).range([w, 0])
  const altitude = scaleLinear()
    .domain([0, Math.PI / 2])
    .range([0, 1])
</script>

<svg {width} {height}>
  <defs>
    <marker
      id="arrow"
      viewBox="0 0 10 10"
      refX="8"
      refY="5"
      markerWidth="4"
      markerHeight="4"
      orient="auto-start-reverse"
    >
      <path d="M 0 0 L 10 5 L 0 10 z" fill="deepskyblue" />
    </marker>
  </defs>
  <!-- <rect
    {width}
    {height}
    fill={sunPosition.altitude < 0 ? 'transparent' : 'deepskyblue'}
    opacity={0.1}
  /> -->
  <g transform="translate({margin.left}, {margin.right})">
    <g transform="translate({center[0]}, {center[1]})">
      <!-- <circle
        r={50 + windSpeed.at(index) * 10}
        fill="none"
        stroke="#fff"
        stroke-width="1"
      /> -->

      {#each BEAUFORT as beaufort, i}
        {#if BEAUFORT.indexOf(closest(windSpeed.at(index))) >= i}
          <circle
            r={b(beaufort)}
            fill="none"
            stroke="#444"
            stroke-dasharray="2 2"
          />
          {#if BEAUFORT.indexOf(closest(windSpeed.at(index))) === i}
            <text
              y={-b(beaufort)}
              stroke-width={5}
              style="text-anchor: middle; dominant-baseline: middle; font-size: 11px; fill: var(--background-color); stroke: var(--background-color);"
              >{i + 1} bft.</text
            >
            <text
              y={-b(beaufort)}
              fill={BEAUFORT.indexOf(closest(windSpeed.at(index))) >= i
                ? '#fff'
                : '#777'}
              style="text-anchor: middle; dominant-baseline: middle; font-size: 11px;"
              >{i + 1} bft.</text
            >
          {/if}
        {/if}}
      {/each}

      {#each trails as j, k}
        <!-- <circle
          r={50 + windSpeed.at(index + j) * 10}
          fill="none"
          stroke={j === 0
            ? 'orange'
            : windSpeed.at(index + trails[k]) >
              windSpeed.at(index + trails[k - 1])
            ? 'red'
            : 'skyblue'}
          opacity={1 - j * 0.5}
        /> -->

        {#if index + j <= data.hourly.time.length - 1}
          {#if k === 0}
            <line
              x1={Math.cos(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(0)}
              y1={Math.sin(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(0)}
              x2={Math.cos(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(windSpeed.at(index + j))}
              y2={Math.sin(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(windSpeed.at(index + j))}
              stroke={j === 0 ? 'deepskyblue' : 'white'}
              stroke-width={j === 0 ? 4 : 2 - j * 2}
              stroke-linecap="round"
              opacity={j % 1 === 0 ? 0.9 : 0.8 - j * 0.2}
              marker-start="url(#arrow)"
            />
          {/if}

          <circle
            r={50 - 2 * k}
            fill={interpolateSpectral(
              temp(
                temperature.at(
                  Math.min(
                    data.hourly.time.length - 1,
                    index + trails[trails.length - k - 1]
                  )
                )
              )
            )}
            stroke="#242424"
            stroke-width={0.1}
          />

          <!-- {#if j % 1 === 0}
            <text
              dy={-5}
              x={Math.cos(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(windSpeed.at(index + j))}
              y={Math.sin(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(windSpeed.at(index + j))}
              style="text-anchor: middle; font-size: 11px;"
              fill={k === 0 ? 'deepskyblue' : '#fff'}
            >
              {#if k > 0}+0{j}:00{:else}now{/if}</text
            >
          {/if} -->

          {#if k > 0}
            <line
              x1={Math.cos(
                -Math.PI / 2 +
                  windDirection.at(index + trails[k - 1]) * (Math.PI / 180)
              ) * b(windSpeed.at(index + trails[k - 1]))}
              y1={Math.sin(
                -Math.PI / 2 +
                  windDirection.at(index + trails[k - 1]) * (Math.PI / 180)
              ) * b(windSpeed.at(index + trails[k - 1]))}
              x2={Math.cos(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(windSpeed.at(index + j))}
              y2={Math.sin(
                -Math.PI / 2 + windDirection.at(index + j) * (Math.PI / 180)
              ) * b(windSpeed.at(index + j))}
              stroke={'white'}
              stroke-width={2}
              opacity={0.8 - j * 0.2}
            />
          {/if}
        {/if}
      {/each}

      <text
        style="text-anchor: middle; dominant-baseline: middle; font-size: 20px; font-weight: bold;"
        >{(Math.round(temperature.at(index) * 10) / 10).toFixed(1)}&deg;C</text
      >

      <circle r={w / 2} fill="none" stroke="#fff" stroke-dasharray="2 2" />
      {#if sunPosition.altitude >= 0}
        <line
          x2={Math.cos(sunPosition.azimuth + Math.PI / 2) *
            altitude(Math.PI / 2 - sunPosition.altitude) *
            w *
            0.5}
          y2={Math.sin(sunPosition.azimuth + Math.PI / 2) *
            altitude(Math.PI / 2 - sunPosition.altitude) *
            w *
            0.5}
          stroke="orange"
          stroke-width={120}
          stroke-linecap="round"
          opacity={0.2}
        />
      {/if}

      <circle
        fill={sunPosition.altitude <= 0 ? 'transparent' : 'yellow'}
        stroke={sunPosition.altitude < 0 ? '#666' : 'none'}
        r={10}
        cx={Math.cos(sunPosition.azimuth + Math.PI / 2) *
          altitude(Math.PI / 2 - sunPosition.altitude) *
          w *
          0.5}
        cy={Math.sin(sunPosition.azimuth + Math.PI / 2) *
          altitude(Math.PI / 2 - sunPosition.altitude) *
          w *
          0.5}
      />

      {#each ['N', 'NE', 'E', 'SE', 'S', 'SW', 'W', 'NW'] as dir, i}
        <text
          x={Math.cos((i / 8) * Math.PI * 2 - Math.PI / 2) * (0.5 * w + 20)}
          y={Math.sin((i / 8) * Math.PI * 2 - Math.PI / 2) * (0.5 * w + 20)}
          style="font-weight: bold; font-size: {i % 2 === 0
            ? 24
            : 16}px;  fill: white; text-anchor: middle; dominant-baseline: middle;"
          >{dir}</text
        >
      {/each}
    </g>

    <!-- <g
      transform="translate({azimuth(sunPosition.azimuth)}, {altitude(
        sunPosition.altitude
      )})"
    >
      <circle
        r={10}
        fill={sunPosition.altitude >= 0 ? 'yellow' : 'transparent'}
        stroke={sunPosition.altitude < 0 ? 'yellow' : 'none'}
      />
    </g> -->
  </g>

  <text
    x={width / 2}
    y={height - 60}
    style="text-anchor: middle; font-size: 16px; fill: white;"
    >{new Date(times.at(index)).toLocaleDateString()}</text
  >
  <text
    x={width / 2}
    y={height - 30}
    style="text-anchor: middle; font-size: 16px; fill: white;"
    >{new Date(times.at(index)).toLocaleTimeString()}</text
  >

  <rect
    {width}
    {height}
    fill="transparent"
    on:mousemove={e => (index = x(e.offsetX))}
  />
</svg>
<div>
  <input
    type="range"
    min={0}
    max={data.hourly.time.length - 1 || 0}
    bind:value={index}
    step={0.1}
    style="width: 800px;"
  />
</div>

<style>
  svg {
    border: 1px solid #444;
  }
</style>
