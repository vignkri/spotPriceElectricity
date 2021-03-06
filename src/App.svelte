<script lang="ts">

	import { onMount } from 'svelte';
	import * as d3 from 'd3';
import { svg_element } from 'svelte/internal';

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

	async function postToEnergidataservice(filter) {

		let url: string = 'https://data-api.energidataservice.dk/v1/graphql';

		// Get the data for given date
		const today: Date = new Date();
		let days: number = 86400000 //number of milliseconds in a day
		const tomorrow: Date = new Date( today.getTime() + ( 2 * days));
		const seven_days_ago: Date = new Date( today.getTime() - (7 * days));

		const tomorrow_str: string = tomorrow.toISOString().split("T")[0];
		const past_str: string = seven_days_ago.toISOString().split("T")[0];

		let query = {
			"query": "query Dataset {elspotprices(where: {HourUTC: {_gte: \"" + past_str + "\", _lte: \"" +  tomorrow_str + "\"}PriceArea: {_eq: \"" + filter + "\"}} order_by: {HourUTC: desc} offset: 0){ HourDK SpotPriceDKK SpotPriceEUR }}"
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

	let priceAreaFilter: string = "DK2";

	const elspot_query = (async() => {

		const response = await postToEnergidataservice(priceAreaFilter);

		const output = response.map(record => {

			let current_date: Date = new Date(record.HourDK);
			let spotPriceInDKKForkWH: number = (record.SpotPriceEUR * 7.44) / 1000;

			return {
				dates: current_date,
				spotPriceDKK: spotPriceInDKKForkWH,
			}
		})
		
		return await output
	})();


	let el;

	onMount(async() => {

		let data = await elspot_query;
		// TODO: Compute tomorrow's minimum period
		// TODO: Compute best time to do heavy loads
		const X = d3.map(data, d => d.dates);
		const Y = d3.map(data, d => d.spotPriceDKK);
		const I = d3.range(X.length);
		var defined = (d, i) => !isNaN(X[i]) && !isNaN(Y[i]);
		const D = d3.map(data, defined);

		// Compute domains
		const xDomain = d3.extent(X);
		const yDomain = [0, d3.max(Y)];

		let marginDef = {
			marginTop: 20,
			marginRight: 50,
			marginBottom: 30,
			marginLeft: 50
		};

		const width = window.innerWidth;
		const height = 400;

		const xRange = [marginDef.marginLeft, width - marginDef.marginRight];
		const yRange = [height - marginDef.marginBottom, marginDef.marginTop];

		const xScale = d3.scaleUtc(xDomain, xRange);
		const yScale = d3.scaleLinear(yDomain, yRange);
		const xAxis = d3.axisBottom(xScale).ticks(width / 80).tickSizeOuter(0);
		const yAxis = d3.axisLeft(yScale).ticks(height / 40);

		// Construct a line generator
		const line = d3.line()
        .defined(i => D[i])
        .curve(d3.curveStep)
        .x(i => xScale(X[i]))
        .y(i => yScale(Y[i]));


		const svg = d3.select("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto; height: intrinsic;");
  
		svg.append("g")
			.attr("transform", `translate(0,${height - marginDef.marginBottom})`)
			.call(xAxis);
	
		svg.append("g")
			.attr("transform", `translate(${marginDef.marginLeft},0)`)
			.call(yAxis)
			.call(g => g.select(".domain").remove())
			.call(g => g.selectAll(".tick line").clone()
				.attr("x2", width - marginDef.marginLeft - marginDef.marginRight)
				.attr("stroke-opacity", 0.1))
			.call(g => g.append("text")
				.attr("x", -marginDef.marginLeft)
				.attr("y", 10)
				.attr("fill", "currentColor")
				.attr("text-anchor", "start")
				.text("Spot Price DKK/kWh"));
	
		svg.append("path")
			.attr("fill", "none")
			.attr("stroke", "currentColor")
			.attr("stroke-width", 1.5)
			.attr("stroke-linecap", "round")
			.attr("stroke-linejoin", "round")
			.attr("stroke-opacity", 1)
			.attr("d", line(I));
	});

</script>

<main>
	<div id="title">
		<h1>DK2 {name}</h1>
		<p id="ref">
			Built by <a href="https://vigne.sh">Vignesh Krishnamoorthy</a>
		</p>
	</div>
	<svg></svg>
	<div bind:this={el} id="timeseries" class="svg">

	</div>
</main>

<style>
	main {
		text-align: left;
		padding: 1em;
		width: 100%;
		margin: 0 auto;
	}
	
	div#title {
		width: 98%;
		margin-bottom: 10px;
	}

	a#ref {
		margin-top: -10px;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 4em;
		font-weight: 100;
	}

	@media (min-width: 640px) {
		main {
			max-width: null;
		}
	}
</style>