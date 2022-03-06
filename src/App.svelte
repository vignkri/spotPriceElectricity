<script lang="ts">

	import { onMount } from 'svelte';

	export let name: string;

	type Record = {
			HourUTC: Date;
			HourDK: Date;
			PriceArea: string;
			SpotPriceDKK: number;
			SpotPriceEUR: number;
		};

	type ElspotPriceObject = {
		elspotprices: Array<Record>
	};

	type ResponseObject = {
		data: ElspotPriceObject
	};

	async function postToEnergidataservice() {

		let url: string = 'https://data-api.energidataservice.dk/v1/graphql';

		// Get the data for given date
		const today: Date = new Date();
		let days: number = 86400000 //number of milliseconds in a day
		const seven_days_ago: Date = new Date( today.getTime() - (7 * days));

		const today_str: string = today.toISOString().split("T")[0];
		const past_str: string = seven_days_ago.toISOString().split("T")[0];

		let query = {
			"query": "query Dataset {elspotprices(where: {HourUTC: {_gte: \"" + past_str + "\", _lte: \"" +  today_str + "\"}PriceArea: {_eq: \"DK2\"}} order_by: {HourUTC: desc} offset: 0){HourUTC HourDK PriceArea SpotPriceDKK SpotPriceEUR }}"
		};

		const response = await fetch(url, {
			method: "POST",
			mode: "cors",
			cache: "no-cache",
			credentials: "same-origin",
			headers: {
				'Content-Type': "application/json",
			},
			body: JSON.stringify(query)
		});

		const dataset: Promise<ResponseObject> = response.json();
		let data_array = (await dataset).data.elspotprices;
		return await data_array
	}

	const elspot_query: Promise<string> = (async() => {
		const response = await postToEnergidataservice();

		const output = response.map(record => {

			let current_date: Date = new Date(record.HourDK);
			let spotPriceInDKKForkWH: number = record.SpotPriceDKK / 1000;

			return {
				isoDate: current_date.toISOString().split("T")[0],
				hourOfDay: current_date.toISOString().split("T")[1].split("+")[0],
				spotPriceDKKkWh: spotPriceInDKKForkWH,
			}
		})
		
		return await JSON.stringify(output)
	})();

</script>

<main>
	<h1>{name}</h1>
	<p>
		The price you pay for electricity is the spot price + taxes and tariffs. If the spot prices are high, the price you pay for electricity goes up. If the prices are low, then you pay much lower prices in total. Ideally, the price should be low when you use your most expensive appliance i.e., washing machine, dishwasher, ovens, electric vehicles.
	</p>
	{#await elspot_query}
		<p>...waiting</p>
	{:then timeseries }
		<p>{timeseries}</p>
	{:catch error}
		<p>An error occurred, unable to fetch data</p>
	{/await}
</main>

<style>
	main {
		text-align: center;
		padding: 1em;
		max-width: 240px;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 4em;
		font-weight: 100;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>