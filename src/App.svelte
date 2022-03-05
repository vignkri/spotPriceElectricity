<script lang="ts">
	export let name: string;

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