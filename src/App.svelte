<script lang="ts">
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


		let query = {
			"query": "query Dataset {elspotprices(where: {HourUTC: {_gte: \"2022-02-27\", _lt: \"2022-03-05\"}PriceArea: {_eq: \"DK2\"}} order_by: {HourUTC: desc} offset: 0){HourUTC HourDK PriceArea SpotPriceDKK SpotPriceEUR }}"
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

	const elspot_query = (async() => {
		const response = await postToEnergidataservice();
		return await JSON.stringify(response)
	})();
</script>

<main>
	<h1>{name}</h1>
	{#await elspot_query}
		<p>...waiting</p>
	{:then timeseries }
		<p>{timeseries}</p>
	{:catch error}
		<p>An error occurred!</p>
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