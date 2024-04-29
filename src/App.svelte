<script>
	import { Map, LngLat } from "mapbox-gl";
	import "../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import Icon from 'fa-svelte'
	import { faXmarkSquare } from '@fortawesome/free-solid-svg-icons/faXmarkSquare'
	import { faXmark } from '@fortawesome/free-solid-svg-icons/faXmark'
	import { onMount, onDestroy } from "svelte";
	import * as d3 from "d3";
	import proj4 from "proj4";
	import * as turf from "@turf/turf";
	import MiniSearch from "minisearch";
	import { computePosition, autoPlacement, offset } from "@floating-ui/dom";
	
	let icon = faXmark;
	let map;
	let mapContainer;
	let mapViewChanged = 0;
	let hoveredProperties = null;
	let hoveredId = null;
	let hoveredData = null;
	$: if (hoveredProperties)
		hoveredData = dataLookup[hoveredProperties.muni_id];
	let cursor = { x: 0, y: 0 };
	let lng, lat, zoom;
	let tooltip;
	let tooltipPosition = {x: 0, y: 0};
	let minisearch;
	let data;
	let densityData;
	let dataLookup = {};
	let geography;
	$: visToggle = true;
	$: visFeature = "dwellingDensity";
	$: {
		console.log(visToggle);
		if (visToggle) {
			visFeature = "dwellingDensity";
		} else {
			visFeature =  "zonedDensity";
		}
	}
	let densityCmap = ["#d1eeea",1e-3,"#a8dbd9",1.129606099,"#85c4c9",1.217380321,"#68abb8",1.377882152,"#4f90a6",1.631587289,"#3b738f",3.396340725,"#2a5674"];
	let zonedDensityCmap = ["#d1eeea",1e-3,"#a8dbd9",1.993243243,"#85c4c9",3,"#68abb8",4.513858696,"#4f90a6",6.170743498,"#3b738f",51.51252103,"#2a5674"];
	$: features = [];
	$: centroids = [];
	let targetMuni;
	let sourceMuni;
	let counterfactualHousing = null;
	
	$: console.log(sourceMuni);
	$: {
		if (targetMuni && sourceMuni) {
			counterfactualHousing = Math.floor(dataLookup[targetMuni].area * dataLookup[sourceMuni].density) - dataLookup[targetMuni].pop;
		} else {
			counterfactualHousing = null;
		}
	}

	lng = -71.224518;
	lat = 42.213995;
	zoom = 10;

	onMount(async function () {
		const initialState = { lng: lng, lat: lat, zoom: zoom };
		const container = document.querySelector('.fullpage');
		const items = document.querySelectorAll('.section');

		container.addEventListener('wheel', (event) => {
		event.preventDefault();
		const delta = event.deltaY;

		container.scrollBy({
			top: delta,
			behavior: 'smooth'
		});
		});

		data = await d3.csv(
			"housing_census_municipalities_compliance.csv",
			(d) => ({
				...d,
				area_km: +d.land_area_sq_miles_2020 * 2.59,
				density: (+d.pop) / (+d.land_area_sq_miles_2020 * 2.59),
			}),
		);
		densityData = await d3.csv(
			"agg_density.csv", (d) => d
		);
		data.forEach((datum) => {
			dataLookup[datum.muni_id] = datum;
			densityData.forEach((densityDatum) => {
				if (datum.municipal !== densityDatum.municipal) {
					return;
				}
				dataLookup[datum.muni_id] = {
					...dataLookup[datum.muni_id],
					avg_zoned_density: densityDatum.avg_zoned_density,
					avg_actual_density: densityDatum.avg_actual_density,
				};
			});
		});
		// densityData.forEach((datum) => {
		// 	console.log(densityData)
		// 	dataLookup[datum.muni_id] = {
		// 		...dataLookup[datum.muni_id],
		// 		avg_zoned_density: datum.avg_zoned_density,
		// 		avg_actual_density: datum.avg_actual_density,
		// 	};
		// });
		// console.log(JSON.stringify(dataLookup));

		geography = await d3.json(
			"ma_municipalities.geojson",
			// "Existing_Bike_Network_2022.geojson",
			(g) => g,
		);
		features = geography.features;
		if (geography.crs.properties.name === "urn:ogc:def:crs:EPSG::26986") {
			proj4.defs(
				"EPSG:26986",
				'PROJCS["NAD83 / Massachusetts Mainland",GEOGCS["NAD83",DATUM["North_American_Datum_1983",SPHEROID["GRS 1980",6378137,298.257222101,AUTHORITY["EPSG","7019"]],AUTHORITY["EPSG","6269"]],PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],UNIT["degree",0.01745329251994328,AUTHORITY["EPSG","9122"]],AUTHORITY["EPSG","4269"]],UNIT["metre",1,AUTHORITY["EPSG","9001"]],PROJECTION["Lambert_Conformal_Conic_2SP"],PARAMETER["standard_parallel_1",42.68333333333333],PARAMETER["standard_parallel_2",41.71666666666667],PARAMETER["latitude_of_origin",41],PARAMETER["central_meridian",-71.5],PARAMETER["false_easting",200000],PARAMETER["false_northing",750000],AUTHORITY["EPSG","26986"],AXIS["X",EAST],AXIS["Y",NORTH]]',
			);
			let massProj = proj4.defs("EPSG:26986");
			let worldProj = proj4.defs("EPSG:4326");
			let proj = proj4(massProj, worldProj);

			features.forEach(function (feature) {
				var geom = feature.geometry;
				if (geom && geom.type === "MultiPolygon") {
					for (var i = 0; i < geom.coordinates.length; i++) {
						for (var j = 0; j < geom.coordinates[i].length; j++) {
							for (
								var k = 0;
								k < geom.coordinates[i][j].length;
								k++
							) {
								// console.log(geom.coordinates[i][j][k]);
								geom.coordinates[i][j][k] = proj.forward(
									geom.coordinates[i][j][k],
								);
							}
						}
					}
				} else if (geom && geom.type === "Polygon") {
					for (var i = 0; i < geom.coordinates.length; i++) {
						for (var j = 0; j < geom.coordinates[i].length; j++) {
							geom.coordinates[i][j] = proj.forward(
								geom.coordinates[i][j],
							);
						}
					}
				}
				let area = turf.area(feature.geometry) / (1000 * 1000);
				let pop = dataLookup[feature.properties.muni_id].pop;
				let density = pop / area;
				let dwellingDensity = dataLookup[feature.properties.muni_id].avg_actual_density;
				let zonedDensity = dataLookup[feature.properties.muni_id].avg_zoned_density;
				feature.properties = {
					...feature.properties,
					"area": +area,
					"pop": +pop,
					"density": +density,
					"dwellingDensity": +dwellingDensity,
					"zonedDensity": +zonedDensity,
				};
				dataLookup[feature.properties.muni_id]["area"] = area;
				dataLookup[feature.properties.muni_id]["pop"] = pop;
				dataLookup[feature.properties.muni_id]["density"] = density;
				dataLookup[feature.properties.muni_id]["geometry"] = feature.geometry;
			});
			features = features.sort((a,b) => (a.properties.municipal > b.properties.municipal) ? 1 : ((b.properties.municipal > a.properties.municipal) ? -1 : 0))
			minisearch = new MiniSearch({
				fields: ["municipal", "muni_id"],
				storeFields: ["municipal"],
			});
			let searchDocument = geography.features.map((feature) => ({
				...feature.properties,
				id: feature.properties.muni_id,
			}));
			minisearch.addAll(searchDocument);
			// console.log(JSON.stringify(searchDocument));
		}

		map = new Map({
			container: mapContainer,
			accessToken:
				"pk.eyJ1IjoidGhlZGl2dGFnZ3V5IiwiYSI6ImNpcWM4N3FlaDAxd2Nmd20xejdwdmVoNmwifQ.Toi-P83h-0tC_mj60h25rg",
			style: `mapbox://styles/mapbox/light-v9`,
			center: [initialState.lng, initialState.lat],
			zoom: initialState.zoom,
			transition: {
				"duration": 1200,
				"delay": 0
			}
		});

		map.on("load", () => {
			map.addSource("mass", {
				type: "geojson",
				data: geography,
				generateId: true,
			});

			map.addLayer({
				id: "mass-layer",
				type: "fill",
				source: "mass",
				layout: {},
				paint: {
					"fill-color": [
						"step",
						["get", "dwellingDensity"],
						"#d1eeea",0,"#a8dbd9",1.129606099,"#85c4c9",1.217380321,"#68abb8",1.377882152,"#4f90a6",1.631587289,"#3b738f",3.396340725,"#2a5674"
						// "#e4f1e1",10,"#b4d9cc",100,"#89c0b6",1000,"#63a6a0",3000,"#448c8a",6000,"#287274",9000,"#0d585f",
						// "#d1eeea",100,"#a8dbd9",500,"#85c4c9",1000,"#68abb8",3000,"#4f90a6",6000,"#3b738f",9000,"#2a5674"
						// "#d1eeea",0,"#a8dbd9",25,"#85c4c9",50,"#68abb8",75,"#4f90a6",100,"#3b738f",120,"#2a5674"
						
						// "#ffeda0",10,"#ffeda0",50,"#fed976",100,"#feb24c",500,"#fd8d3c",1000,"#fc4e2a",2000,"#e31a1c",3000,"hsl(348, 100%, 37%)",6000,"#bd0026"
					],
					"fill-opacity": [
						"case",
						["boolean", ["feature-state", "hover"], false],
						1,
						0.9,
					],
				},
			});

			map.addLayer({
				id: "outline",
				type: "line",
				source: "mass",
				layout: {},
				paint: {
					"line-color": "#000",
					"line-width": 0.5,
				},
			});
			map.fitBounds(turf.bbox(geography), {
				padding: { top: 10, bottom: 25, left: 15, right: 5 },
			});

			map.on("move", () => {
				mapViewChanged++;
				updateData();
			});

			map.on("mousemove", "mass-layer", (e) => {
				if (e.features.length > 0) {
					if (hoveredId !== null) {
						map.setFeatureState(
							{ source: "mass", id: hoveredId },
							{ hover: false },
						);
					}
					hoveredProperties = e.features[0].properties;
					hoveredId = e.features[0].id;
					map.setFeatureState(
						{ source: "mass", id: e.features[0].id },
						{ hover: true },
					);
				}
			});

			map.on("mouseleave", "mass-layer", () => {
				map.setFeatureState(
					{ source: "mass", id: hoveredId },
					{ hover: false },
				);
				// hoveredProperties = null;
				hoveredId = null;
			});

			features.forEach(function (feature) {
				let centroid = turf.centerOfMass(feature.geometry);
				// let centroid = turf.centerOfMass(turf.bboxPolygon(turf.bbox(feature.geometry)));
				centroid = {
					...centroid,
					properties: feature.properties,
				};
				centroids = [...centroids, centroid];
			});
			// console.log(centroids);
		});
	});

	function updateData() {
		zoom = map.getZoom();
		lng = map.getCenter().lng;
		lat = map.getCenter().lat;
	}

	onDestroy(() => {
		map.remove();
	});

	function projectCentroid(centroid) {
		let point = new LngLat(+centroid[0], +centroid[1]);
		let { x, y } = map.project(point);
		return { x: x, y: y };
	}

	function paintMap() {
		let cmap;
		if (visFeature == "dwellingDensity") {
			cmap = densityCmap
		} else {
			cmap = zonedDensityCmap;
		}
		if (sourceMuni && targetMuni) {
			map.style.stylesheet.layers.forEach(function(layer) {
				if (layer.type === 'symbol') {
					map.setLayoutProperty(layer.id,"visibility", "none");
				}
			});
			let geometry = turf.union(
				dataLookup[sourceMuni]["geometry"],
				dataLookup[targetMuni]["geometry"],
			);
			map.fitBounds(turf.bbox(geometry), {
				padding: { top: 100, bottom: 100, left: 100, right: 100},
				easing: (t) => {return t*t*t},
				duration: 1000,
			});
			map?.setPaintProperty(
				'mass-layer',
				'fill-color',
				[
					"step",
					[
						"case",
						["==", ["get", "muni_id"], targetMuni], ["get", visFeature], 
						["==", ["get", "muni_id"], sourceMuni], ["get", visFeature], 
						-1
					],
					"#DDDDDD",0,...cmap
				]
			);
			map?.setPaintProperty(
				"mass-layer",
				"fill-opacity",
				[
					"*", [
						"case",
						["boolean", ["feature-state", "hover"], false],
						1,
						0.9,
					], [
						"case",
						["==", ["get", "muni_id"], targetMuni], 1, 
						["==", ["get", "muni_id"], sourceMuni], 1, 
						0.5,
					]
				]
			);
		}
		else {
			map.style.stylesheet.layers.forEach(function(layer) {
				if (layer.type === 'symbol') {
					map.setLayoutProperty(layer.id, "visibility", "visible");
				}
			});
			map.fitBounds(turf.bbox(geography), {
				padding: { top: 10, bottom: 25, left: 15, right: 5 },
			});
			map?.setPaintProperty(
				"mass-layer",
				"fill-color",
				[
					"step",
					["get", visFeature],
					...cmap
				]
			);
			map?.setPaintProperty(
				"mass-layer",
				"fill-opacity",
				[
					"case",
					["boolean", ["feature-state", "hover"], false],
					1,
					0.9,
				]
			);
		}
	}
</script>

<head> </head>


<div class="fullpage">
	<div class="section">
		<div class="text-wrap">
			<h1 style="font-size: 6rem;">Policy ---> <mark>Reality</mark>
			</h1>
		</div>

	</div>

	<div class="section">
		<div class="text-wrap"  style="top: 35%">
			<h1 style="font-size: 3rem;">Massachusetts has the <mark>5th most expensive</mark> housing prices in the country.</h1>
			<h1>
				Nearly <mark>half</mark> of the state’s renters are <mark>rent-burdened</mark> while a
				<mark>quarter</mark> are <mark>severely rent-burdened</mark>.
			</h1>
			<h1 style="font-size: 1.6rem;">Our housing market is extremely saturated with <mark>rental vacancy rates at just 2.4%</mark>.
			</h1>
			<h1 style="font-size: 1.6rem;">There is estimated to be a shortage of <mark>125,000-200,000 housing units by 2030</mark>, with <mark>35,000-110,000 new
				units</mark> required just to meet current demand.
			</h1>

		</div>
		<img src="home.png" alt="home" style="position: absolute; bottom: 0; right: 0; width: 35%;">

	</div>
	<div class="section">
		<div class="text-wrap"  style="top: 50%">
			<h1 style="font-size: 1.85rem;">To mitigate the housing crisis, we need to build denser housing and use existing housing stock <mark>as efficiently as possible</mark>.
			</h1>
			<h1 style="font-size: 1.4rem;">
				One way to encourage this is to ensure that local laws allow for <mark>higher density housing*</mark>.
			</h1>

			<img src="home_2.png" alt="home" style="width: 70%; padding-top: 50px;">
			<h1 style="font-size: .7rem;">
				*Housing density can be defined as the average number of dwelling units per acre (du/ac).
			</h1>
		</div>

	</div>
	<div class="section">
		<div class="text-wrap" style="font-size: 1.5rem;">
			<h1>How might <mark>zoning</mark> allow for more <mark>density</mark>?</h1>
		</div>

	</div>
	<div class="section">
		<div class="text-wrap" style="top: 50%">
			<h1>One example of increasing neighborhood density is building an <mark>accessory dwelling unit (ADU)</mark>.
			</h1>
			<img src="houses.png" alt="home" style="width: 75%; padding-top: 20px; padding-bottom: 20px;">
			<h1 style="font-size: 1.3rem;">
				Sometimes called “granny flats”, ADU’s can introduce some <mark>gentle density</mark> into a neighborhood and
				often increase property values. They can allow for multi-generational living or even be rented out.
			</h1>
			<h1 style="font-size: 2.05rem;">ADU’s are often <mark>prohibited by local laws</mark>, but new laws to allow ADU’s have recently been adopted in
				places like Somerville!
			</h1>

		</div>
	</div>
	<div class="section">
		<div class="text-wrap" style="top: 50%">
			<h1 style="font-size: 1.5rem;">
				Another example of density-friendly adaptations are <mark>single family home conversions</mark>.
			</h1>
			<h1 style="font-size: 1.5rem;">
				This involves renovating a single house into multiple individual units.
			</h1>
			<img src="site_plan_condo.jpeg" alt="home" style="width: 65%;">
			<h1  style="font-size: 1rem; padding-top: 20px;">
				These homes often blend right in with the rest of the neighborhood, even preserving some historic homes.
			</h1>
			<h1 style="font-size: 1rem;">
				In an area that is zoned for single family homes only, this type of conversion would not be possible because it would exceed the allowable dwelling units per acre. 
			</h1>

		</div>
	</div>
	<div class="section">
		<div class="text-wrap" style="font-size: 1.5rem;">
			<h1>What do different levels of <mark>housing density</mark> look like in the Boston area?</h1>
		</div>

	</div>
	<div class="section">
		<div class="text-wrap" style="top: 50%;">
			<h1 style="font-size: 1.3rem;">
				<mark style="font-size: 2.2rem; color: #595959;">Milton</mark> is an example of <mark>low-density housing</mark> made up of <u>single-family</u> homes with <u>large yards</u>. 
			</h1>
			<img src="street1.png" alt="home" style="width: 90%; padding-top: 20px;">
			<h1 style="font-size: .9rem;  padding-top: 10px;">
				Average density: <u>1.2 du/ac</u>.
			</h1>

		</div>
	</div>

	<div class="section">
		<div class="text-wrap" style="top: 50%;">
			<h1 style="font-size: 1.3rem;">
				<mark style="font-size: 2.2rem; color: #595959;">Waltham</mark> is an example of <mark>medium-density housing</mark>, with a mix of <u>single-family</u> homes and <u>gentle density</u> like <u>duplexes</u>. 
			</h1>
			<img src="street2.png" alt="home" style="width: 90%; padding-top: 20px;">
			<h1 style="font-size: .9rem;  padding-top: 10px;">
				Average density: <u>2.2 du/ac</u>.
			</h1>

		</div>
	</div>


	<div class="section">
		<div class="text-wrap" style="top: 50%;">
			<h1 style="font-size: 1.3rem;">
				<mark style="font-size: 2.2rem; color: #595959;">Cambridge</mark> is an example of <mark>high-density housing</mark>, with a mix of <u>multi-family</u> homes, <u>apartments</u>, and <u>mixed-use buildings</u>. 
			</h1>
			<img src="street3.png" alt="home" style="width: 90%; padding-top: 20px;">
			<h1 style="font-size: .9rem;  padding-top: 10px;">
				Average density: <u>3.4 du/ac</u>.
			</h1>

		</div>
	</div>

	<div class="section">
		<div class="text-wrap" >
			<h1 style="font-size: 3rem;">How many more people could we house if we <mark>up-zoned</mark> Massachusetts?
			</h1>
		</div>

	</div>

	<div class="section">
		<div class="map-wrap">
			<div class="sidebar">
				If
				<select bind:value={targetMuni} on:change={paintMap}>
					{#each features as feature}
						<option value={feature.properties.muni_id}>{feature.properties.municipal}</option>
					{/each}
				</select>
				had the density of
				<select bind:value={sourceMuni} on:change={paintMap}>
					{#each features as feature}
						<option value={feature.properties.muni_id}>{feature.properties.municipal}</option>
					{/each}
				</select>
				{#if counterfactualHousing}
					{#if counterfactualHousing >= 0}
						we would <b>house {counterfactualHousing}</b> more people.
					{:else}
						we would <b>displace {-counterfactualHousing}</b> people.
					{/if}
					<button type="button" id="resetButton" on:click={()=>{targetMuni = null; sourceMuni = null; paintMap()}}>
						<Icon class="resetIcon" icon={icon}></Icon>
					</button>
				{:else}
				...
				{/if}
			</div>

			<!-- "#d1eeea",0,"#a8dbd9",5,"#85c4c9",10,"#68abb8",20,"#4f90a6",50,"#3b738f",100,"#2a5674" -->
			<div id="density-legend" class="legend">
				<label class="switch">
					<input type="checkbox" bind:checked={visToggle} on:change={paintMap}>
					<span class="slider round"></span>
				</label>
				{#if visFeature === "dwellingDensity"}
				<div on:outrostart={e => e.target.classList.add('isFading')}
					on:introstart={e => e.target.classList.remove('isFading')}>
					<h4>Density</h4> <p>(dwelling units per acre)</p>
					<div><span style="background-color: #a8dbd9"></span>0.00-1.13</div>
					<div><span style="background-color: #85c4c9"></span>1.13-1.22</div>
					<div><span style="background-color: #68abb8"></span>1.22-1.38</div>
					<div><span style="background-color: #4f90a6"></span>1.38-1.63</div>
					<div><span style="background-color: #3b738f"></span>1.63-3.40</div>
				</div>
				{:else}
				<div on:outrostart={e => e.target.classList.add('isFading')}
					on:introstart={e => e.target.classList.remove('isFading')}>
					<h4>Zoned Density</h4> <p>(dwelling units per acre)</p>
					<div><span style="background-color: #a8dbd9"></span>0.00-1.99</div>
					<div><span style="background-color: #85c4c9"></span>1.99-3.00</div>
					<div><span style="background-color: #68abb8"></span>3.00-4.51</div>
					<div><span style="background-color: #4f90a6"></span>4.51-6.17</div>
					<div><span style="background-color: #3b738f"></span>6.17-51.50</div>
				</div>
				{/if}
			</div>

			{#if dataLookup != {} && sourceMuni && targetMuni}
				<svg width="100%" height="100vh">
					<style>
						.text {
							font: 16px sans-serif;
						}
					</style>
						{#key mapViewChanged}
							{#each centroids as centroid}
								{#if centroid.properties.muni_id == sourceMuni || centroid.properties.muni_id == targetMuni}
									<text
										class="text"
										dominant-baseline="middle"
										text-anchor="middle"
										{...projectCentroid(centroid.geometry.coordinates)}
									>
										{centroid.properties.municipal}
									</text>
									{/if}
							{/each}
						{/key}
				</svg>
			{/if}

			<dl id="muni-tooltip" class="info tooltip" bind:this={tooltip} hidden={hoveredId === null}>
				{#if hoveredProperties && hoveredData}
					<dt>Municipality</dt>
					<dd>{hoveredProperties.municipal}</dd>

					<dt>Population</dt>
					<dd>{hoveredProperties.pop}</dd>

					<dt>Area</dt>
					<dd>{hoveredProperties.area.toFixed(2)}<sup>2</sup> </dd>

					<dt>Average Density</dt>
					<dd>{hoveredProperties.density.toFixed(2)}</dd>

					<dt>Zoned Density</dt>
					<dd>{hoveredProperties.zonedDensity.toFixed(2)}</dd>
				{/if}
			</dl>
			<div class="map" bind:this={mapContainer} />
		</div>

	</div>
</div>

<style>
	.map {
		position: absolute;
		width: 100%;
		height: 100%;
	}
	svg {
		background-color: rgb(35 55 75 / 2%);
		color: #fff;
		padding: 0 0 0 0;
		font-family: monospace;
		z-index: 1;
		position: absolute;
		top: 0;
		left: 0;
		border: 2px dashed #daa520;
		pointer-events: none;
	}

	svg circle {
		color: #fff;
		padding: 0 0 0 0;
		font-family: monospace;
		z-index: 1;
		position: absolute;
		top: 0;
		left: 0;
		border: 2px dashed rgb(218, 165, 32 / 50%);
		pointer-events: none;
	}
	.sidebar {
		background-color: rgb(35 55 75 / 90%);
		color: #fff;
		padding: 12px 12px 12px 12px;
		font-family: monospace;
		z-index: 2;
		position: absolute;
		top: 0;
		left: 0;
		margin: 12px;
		border-radius: 4px;
	}

	.sidebar select {
		padding: 3px;
		margin: 0;
		vertical-align: middle;
	}

	#resetButton {
		padding: 1px;
		margin: 0;
		vertical-align: middle;
		color: oklch(70.95% 0.0937 76.31);
		background-color: rgba(0.0, 0.0, 0.0, 0.0);
		border: none;
	}

	#resetButton :global(.resetIcon) {
		padding: 0;
		margin: 0;
		font-size: 1.8em;
		vertical-align: text-bottom;
	}

	dl.info {
		width: max-content;
		position: absolute;
		top: 0;
		right: 0;
		margin: 12px;
		
		display: grid;
		grid-auto-columns: 8em;
		grid-auto-flow: column;
		z-index: 3;
		background-color: rgb(255 255 255 / 100%);
		box-shadow: 0px 0px 5px oklch(50% 0 0);
		border-radius: 5px;
		padding-top: 10px;
		padding-bottom: 10px;

		transition-duration: 200ms;
		transition-property: opacity, visibility;

		&[hidden]:not(:hover, :focus-within) {
			opacity: 0;
			visibility: hidden;
		}
	}

	dt,
	dd {
		margin: 0;
		padding: 3px;
		font-size: small;
		/* background-color: goldenrod; */
	}
	dt {
		grid-column: 1;
		text-align-last: right;
		color: rgb(120 120 120);
		font-weight: 600;
	}
	dd {
		grid-column: 0;
	}

	.tooltip {
		position: fixed;
		top: 1em;
		right: 1em;
	}

	.map-overlay {
        position: absolute;
        bottom: 0;
        right: 0;
        background: #fff;
        margin-right: 20px;
        font-family: monospace;
        overflow: auto;
        border-radius: 3px;
    }

	.legend {
        background-color: #fff;
        bottom: 30px;
		box-shadow: 0px 0px 5px oklch(50% 0 0);
        border-radius: 5px;
        font:
            20px/30px monospace;
		text-align: right;
        padding: 10px;
        position: absolute;
        right: 10px;
        z-index: 1;
    }

    .legend h4 {
		font-size: 1.5rem;
        margin: 0;
    }
    .legend p {
		font-size: 0.8rem;
        margin: 0 0 10px 0;
		/* font-family: sans-serif; */
    }

    .legend div span {
        border-radius: 50%;
        display: inline-block;
        height: 1rem;
        width: 1rem;
        margin-right: 5px;
		vertical-align: baseline;
    }

	.switch {
		position: relative;
		display: inline-block;
		width: 3rem;
		height: 1.5rem;
	}

	.switch input {
		opacity: 0;
		width: 0;
		height: 0;
	}

	.slider {
		position: absolute;
		cursor: pointer;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: #ccc;
		-webkit-transition: .4s;
		transition: .4s;
	}

	.slider:before {
		position: absolute;
		content: "";
		height: 1rem;
		width: 1rem;
		left: 4px;
		bottom: 4px;
		background-color: white;
		-webkit-transition: .4s;
		transition: .4s;
	}

	input:checked + .slider {
		background-color: #2196F3;
	}

	input:focus + .slider {
		box-shadow: 0 0 1px #2196F3;
	}

	input:checked + .slider:before {
		-webkit-transform: translateX(1.5rem);
		-ms-transform: translateX(1.5rem);
		transform: translateX(1.5rem);
	}

	/* Rounded sliders */
	.slider.round {
		border-radius: 1.0rem;
	}

	.slider.round:before {
		border-radius: 50%;
	}
	h1{
		color: #595959;
		/* width: 75%; */
	}

	.text-wrap {
            position: absolute;
            top: 40%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: 70%;
            text-align: center;
            font-size: .9rem;
            font-family: 'Arial';
        }

	.section {
		text-align: center;
		font-size: 14px;
		/* padding: 40px; */
  	}


	.fullpage {
		position: relative;
		width: 100%;
		height: 100vh;
		scroll-behavior: smooth;
		overflow-y: scroll;
		scroll-snap-type: y mandatory;
	}


	.section {
		position: relative;
		width: 100%;
		height: 100vh;
		background-size: cover;
		scroll-snap-align: start;
	}

	mark {
		color: #FF6B00;
		background: none;
	}

	.text-wrap {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		height: 100%;
	}

	img {
		width: 40%;
	}

	.text-wrap p {
		width: 50%;
		text-align: center;
		font-size: 1.5rem;
		font-family: 'Arial';
	}

	@media screen and (max-width: 600px) {
		.text-wrap p {
			width: 80%;
			text-align: center;
			font-size: 1.2rem;
			font-family: 'Arial';
		}
		
	img {
		width: 70%;
	}
	}

	@keyframes scroll {
	0% { transform: translateY(0); }
	100% { transform: translateY(-100%); }
	}
</style>
