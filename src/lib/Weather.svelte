<script>
  import WeatherViz from './WeatherViz.svelte'
  import { json } from 'd3-fetch'
  import SunCalc from 'SunCalc'

  let data = undefined

  // lochem
  // const latitude = 52.1614
  // const longitude = 6.4156

  // rijswijk
  const latitude = 52.03634
  const longitude = 4.32501

  // const longitude = -31.178
  // const latitude = 58.894

  // madrid
  // const latitude = 40.4168
  // const longitude = -3.7038

  const startDate = '2022-08-09'
  const endDate = '2022-08-10'

  let times = SunCalc.getTimes(new Date(startDate), latitude, longitude)
  let position = SunCalc.getPosition(times.sunrise, latitude, longitude)

  $: console.log(times)
  $: console.log(position)

  const URL = `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&hourly=temperature_2m,precipitation,cloudcover,windspeed_10m,winddirection_10m&daily=sunrise,sunset&timezone=Europe%2FBerlin&start_date=${startDate}&end_date=${endDate}`
</script>

{#await json(URL)}
  Loading...
{:then data}
  <WeatherViz {data} {latitude} {longitude} />
{:catch error}
  {console.error(error)}
{/await}
