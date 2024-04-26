<script>
	import { Map, LngLat } from "mapbox-gl";
	import "../node_modules/mapbox-gl/dist/mapbox-gl.css";
	import { onMount, onDestroy } from "svelte";
	import * as d3 from "d3";
	import proj4 from "proj4";
	import * as turf from "@turf/turf";
	import MiniSearch from "minisearch";
	import { computePosition, autoPlacement, offset } from "@floating-ui/dom";

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
	let targetMuni;
	let sourceMuni;
	let counterfactualHousing = null;
	$: console.log(sourceMuni);
	$: {
		if (targetMuni && sourceMuni) {
			counterfactualHousing = Math.floor(dataLookup[targetMuni].area * dataLookup[sourceMuni].density) - dataLookup[targetMuni].pop;
		}
	}

	let data;
	let densityData;
	let dataLookup = {};
	let geography;
	$: features = [];
	$: centroids = [];

	lng = -71.224518;
	lat = 42.213995;
	zoom = 10;

	onMount(async function () {
		const initialState = { lng: lng, lat: lat, zoom: zoom };

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
				let zonedDensity = dataLookup[feature.properties.muni_id].avg_zoned_density;
				feature.properties = {
					...feature.properties,
					"area": +area,
					"pop": +pop,
					"density": +density,
					"zonedDensity": +zonedDensity,
				};
				dataLookup[feature.properties.muni_id]["area"] = area;
				dataLookup[feature.properties.muni_id]["pop"] = pop;
				dataLookup[feature.properties.muni_id]["density"] = density;
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
						["get", "zonedDensity"],
						// "#e4f1e1",10,"#b4d9cc",100,"#89c0b6",1000,"#63a6a0",3000,"#448c8a",6000,"#287274",9000,"#0d585f",
						// "#d1eeea",100,"#a8dbd9",500,"#85c4c9",1000,"#68abb8",3000,"#4f90a6",6000,"#3b738f",9000,"#2a5674"
						// "#d1eeea",0,"#a8dbd9",25,"#85c4c9",50,"#68abb8",75,"#4f90a6",100,"#3b738f",120,"#2a5674"
						"#d1eeea",0,"#a8dbd9",5,"#85c4c9",10,"#68abb8",20,"#4f90a6",50,"#3b738f",100,"#2a5674"
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
		return { cx: x, cy: y };
	}
</script>

<head> </head>

<div class="sidebar">
	If
	<select bind:value={targetMuni}>
		{#each features as feature}
			<option value={feature.properties.muni_id}>{feature.properties.municipal}</option>
		{/each}
	</select>
	had the density of
	<select bind:value={sourceMuni}>
		{#each features as feature}
			<option value={feature.properties.muni_id}>{feature.properties.municipal}</option>
		{/each}
	</select>
	{#if counterfactualHousing}
		{#if counterfactualHousing >= 0}
			we would <b>house {counterfactualHousing}</b> more people
		{:else}
			we would <b>displace {-counterfactualHousing}</b> people
		{/if}
	{/if}.
</div>

<!-- <svg width="100%" height="100vh">
	{#if dataLookup != {}}
		{#key mapViewChanged}
			{#each centroids as centroid}
				<p>
					{centroid.properties.muni_id}: {centroid.properties.area}
				</p>
				<circle
					r={Math.sqrt(centroid.properties.density) / 10}
					opacity=0.7
					fill="#daa520"
					stroke="#daa520"
					stroke-width=1
					stroke-opacity=1
					{...projectCentroid(centroid.geometry.coordinates)}
					on:mouseenter={(evt) => {
						hoveredProperties = centroid.properties;
						cursor = { x: evt.x, y: evt.y };
					}}
				>
					<title>{centroid.properties.municipal}</title>
				</circle>
			{/each}
		{/key}
	{/if}
</svg> -->

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
		<dd>{hoveredProperties.zonedDensity}</dd>

		<dt>Single Family</dt>
		<dd>{(+hoveredData["%_single_family"]).toFixed(2)}%</dd>
	{/if}
</dl>

<div class="map-wrap">
	<div class="map" bind:this={mapContainer} />
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
		padding: 6px 12px;
		font-family: monospace;
		z-index: 2;
		position: absolute;
		top: 0;
		left: 0;
		margin: 12px;
		border-radius: 4px;
	}
	dl.info {
		width: max-content;
		position: absolute;
		top: 3em;
		left: 1em;
		
		display: grid;
		grid-auto-columns: 9em;
		grid-auto-flow: column;
		z-index: 3;
		background-color: rgb(255 255 255 / 100%);
		box-shadow: 0px 0px 10px black;
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
		left: 1em;
	}
</style>
