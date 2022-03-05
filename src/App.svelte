<script lang="ts">
	export let name: string;

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

		return await response.json
	}

	postToEnergidataservice()
		.then(data => {console.log(data)});

	const fetchImage = (async () => {
		const response = await fetch('https://dog.ceo/api/breeds/image/random')
    return await response.json()
	})()
</script>

<main>
	<h1>{name}</h1>
	{#await fetchImage}
		<p>...waiting</p>
	{:then dog_data }
		<!-- svelte-ignore a11y-img-redundant-alt -->
		<img src={dog_data.message} alt="Dog image" />
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