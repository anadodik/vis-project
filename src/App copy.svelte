<script>
	import * as d3 from 'd3';
	import { mapboxgl, key } from './mapboxgl.js';
	import {onMount} from 'svelte';
	
	let width = "100%";
	let height = "100vh";
	
	let data = [];
	let geography;
	let municipalities = [];
	let map;

	onMount(async function() {
		data = await d3.csv('housing_census_municipalities_compliance.csv', (d) => ({
			...d,
			area: +d.land_area_sq_miles_2020,
			density: +d.density_2020
		}));

		geography = await d3.json("ma_municipalities.geojson", (g) => (g));
		createMap(geography);
		createDots(data);
	});

	function createMap(data) {
        map = new mapboxgl.Map({
          container: "map", // container id
          style: "mapbox://styles/mapbox/light-v9", // stylesheet location
          zoom: 5 // starting zoom
        });

        map.on("viewreset", render);
        map.on("move", render);
        map.on("moveend", render);

        // Optional: Modify map with d3
        d3.selectAll(".mapboxgl-canvas")
          .style("opacity", 1)
          .style("position", "absolute")
          .style("z-index", 1);
        return data;
      }

	  function createDots(data) {
        var container = map.getCanvasContainer();

        var svg = d3
          .select(container)
          .append("svg")
          .attr("width", "100%")
          .attr("height", "2000")
          // Ensure d3 layer in front of map
          .style("position", "absolute")
          .style("z-index", 10);

        let dots = svg
          .selectAll("circle")
          .data(data.features)
          .enter()
          .append("circle")
          .attr("class", "circle")
          .attr("r", 5)
          .style("opacity", 0.7)
          .style("fill", "#ff3636");
      }

	  function render() {
        d3.selectAll(".circle")
          .attr("cx", function(d) {
            return project(d.geometry.coordinates).x;
          })
          .attr("cy", function(d) {
            return project(d.geometry.coordinates).y;
          });
      }

	  function project(d) {
        return map.project(new mapboxgl.LngLat(d[0], d[1]));
      }

	
	// $: projection = d3.geoAlbersUsa(); // .fitSize([width, height], geography);
	// $: pathGenerator = d3.geoPath(projection);
	// $: if (geography) municipalities = geography.features.map(feature => {
	// 	return {
	// 		...feature,
	// 		path: pathGenerator(feature)
	// 	}
	// });
	// $: console.log(geography);

	
	
	// $: xScale = d3.scaleBand()
	// 	.domain(data.map(d => d.municipal))
	// 	.range([margin.left, width - margin.right]);
	
	// $: yScale = d3.scaleLinear()
	// 		.domain([0, d3.max(data, d => d.area)])
	// 		.range([height - margin.bottom, margin.top])

	// $: colorScale = d3.scaleOrdinal(d3.schemeTableau10)
	// 		.domain(data.map(d => d.complies_with_zoning));
	
	// let xAxis;
	// let yAxis;
	
	// $: {
	// 	d3.select(yAxis).call(d3.axisLeft(yScale));
		
	// 	d3.select(xAxis)
	// 			.call(d3.axisBottom(xScale))
	// 			.selectAll('.tick > text')
	// 				.attr('y', 0)
	// 				.attr('dy', '0.35em')
	// 				.attr('dx', '-1em')
	// 				.attr('text-anchor', 'end')
	// 				.attr('transform', 'rotate(-90)');
	// }
</script>

<div class="maps-container" bind:clientWidth={width} bind:clientHeight={height}>
<div id="map"></div>
<svg {width} {height}>
	{#each municipalities as municipality}
		<circle
			r="5"
			opacity="0.7"
			cx={project(municipality.geometry.coordinates).x}
			cy={project(municipality.geometry.coordinates).y}
			/>
	{/each}
	<!-- {#each data as d, i}
		<rect class="bar"
					x={xScale(d.municipal)}
					y={yScale(d.area)}
					width={xScale.bandwidth()}
					height={yScale(0) - yScale(d.area)}
					fill={colorScale(d.complies_with_zoning)} />
	{/each} -->
	
	<!-- <g transform="translate(0, {height - margin.bottom})"
		 bind:this={xAxis} />
	
	<g transform="translate({margin.left}, 0)"
		 bind:this={yAxis} /> -->
</svg>
</div>

<style>
	svg {
		border: 2px dashed goldenrod;	
		/* position: absolute; */
		z-index: 1;
	}
	
	.bar {
		stroke: white;
	}

	#map {
		/* position: absolute; */
		top: 0;
		/* bottom: 0; */
		width: 100%;
		z-index: 10;
	}
</style>