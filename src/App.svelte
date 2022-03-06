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
		const seven_days_ago: Date = new Date( today.getTime() - (7 * days));

		const today_str: string = today.toISOString().split("T")[0];
		const past_str: string = seven_days_ago.toISOString().split("T")[0];

		let query = {
			"query": "query Dataset {elspotprices(where: {HourUTC: {_gte: \"" + past_str + "\", _lte: \"" +  today_str + "\"}PriceArea: {_eq: \"" + filter + "\"}} order_by: {HourUTC: desc} offset: 0){HourUTC HourDK PriceArea SpotPriceDKK SpotPriceEUR }}"
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
			let spotPriceInDKKForkWH: number = record.SpotPriceDKK / 1000;

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
			marginRight: 30,
			marginBottom: 30,
			marginLeft: 40
		};

		const width = 800;
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
		<h1>{name}</h1>
		<p>
			The price you pay for electricity is the spot price + taxes and tariffs. If the spot prices are high, the price you pay for electricity goes up. If the prices are low, then you pay much lower prices in total. Ideally, the price should be low when you use your most expensive appliance i.e., washing machine, dishwasher, ovens, electric vehicles.
		</p>
	</div>
	<hr>
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
		width: 100%;
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