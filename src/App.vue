<template>
	<div id="map"></div>
	<div class="overlay top left flex-col gap">
		<Logo class="mb-2" />
		<div v-if="!state.started">
			<h4 class="select mb-2">{{ select }}</h4>
			<div v-if="error" class="bg-danger">{{ error }}</div>
			<div class="flex gap">
				<Button @click="loadFromClipboard" class="bg-success" text="import polygons from clipboard" title="import from clipboard" />
				<input @change="loadFromJSON" type="file" id="file" class="input-file" accept="application/json" />
				<label for="file" class="btn bg-success">Import polygons from GeoJSON</label>
			</div>
		</div>
		<div v-if="!state.started">
			<div class="flex gap">
				<Button @click="selectAllCountries" class="bg-success" text="Select all Countries" title="Select all Countries" />
				<Button v-if="imported" @click="selectAllImports" class="bg-success" text="Select All Imports"
					title="Select All Imports" />
				<Button v-if="selected.length" @click="deselectAll" class="bg-danger" text="Deselect all"
					title="Deselect all" />
			</div>
		</div>

		<div v-if="selected.length" class="selected">
			<h4 class="center mb-2">Countries/Territories ({{ selected.length }})</h4>
			<div v-for="country of selected" class="line flex space-between">
				<div class="flex-center">
					<span v-if="country.feature.properties.code"
						:class="`flag-icon flag-` + country.feature.properties.code.toLowerCase()"></span>
					{{ country.feature.properties.name }}
					<Spinner v-if="state.started && country.isProcessing" class="ml-2" />
				</div>
				<div>
					{{ country.found ? country.found.length : "0" }} / <input type="number"
						:min="country.found ? country.found.length : 0" v-model="country.nbNeeded" />
				</div>
			</div>
		</div>
		<Button @click="clearMarkers" class="bg-warning" text="Clear markers"
			optText="(for performance, this won't erase your generated locations)" title="Clear markers" />
	</div>

	<div class="overlay top right flex-col gap">
		<div v-if="!state.started" class="settings">
			<h4 class="center">Settings</h4>
			<Checkbox v-model:checked="settings.rejectUnofficial" label="Reject unofficial" />
			<hr />

			<Checkbox v-model:checked="settings.rejectDuplicates" label="Reject exact duplicates" />
			<hr />

			<div v-if="settings.rejectUnofficial">
				<Checkbox v-model:checked="settings.rejectNoDescription" label="Reject locations without description" />
				<small>This might prevent trekkers in most cases, but can reject regular streetview without description.
					(eg. Mongolia/South Korea panoramas mostly don't
					have description)</small>
				<hr />

				<Checkbox v-model:checked="settings.rejectDateless" label="Reject locations without date" />
				<small>This will prevent the local business tripod coverage that doesn't have a date.</small>
				<hr />

				<Checkbox @change="settings.getIntersection ? settings.deadEndsOnly = false : true"
					v-model:checked="settings.getIntersection" label="Prefer intersections" />
				<hr />

				<Checkbox @change="settings.deadEndsOnly ? settings.getIntersection = false : true"
					v-model:checked="settings.deadEndsOnly" label="Dead ends only (end of coverage)" />
				<div v-if="settings.deadEndsOnly" class="indent">
					<Checkbox v-model:checked="settings.lookBackwards" label="Look backwards" />
				</div>
				<hr />
			</div>

			<Checkbox v-model:checked="settings.adjustHeading" label="Adjust heading" />
			<div v-if="settings.adjustHeading" class="indent">
				<label class="flex wrap">
					Deviation <input type="range" v-model.number="settings.headingDeviation" min="-90" max="90" /> ({{
						settings.headingDeviation
					}}°)
				</label>
				<small>0° will point directly towards the road.</small>
				<Checkbox v-model:checked="settings.randomHeadingDeviation" label="Randomize in range" />
			</div>
			<hr />

			<Checkbox v-model:checked="settings.adjustPitch" label="Adjust pitch" />
			<div v-if="settings.adjustPitch" class="indent">
				<label class="flex wrap">
					Pitch deviation <input type="range" v-model.number="settings.pitchDeviation" min="-90" max="90" />
					({{ settings.pitchDeviation }}°)
				</label>
				<small>0 by default. -90° for tarmac/+90° for sky</small>
			</div>
			<hr />

			<div>
				Radius
				<input type="number" v-model.number="settings.radius" @change="handleRadiusInput" />
				m
			</div>
			<small>
				Radius in which to search for a panorama.<br />
				Keep it between 100-1000m for best results. Increase it for poorly covered
				territories/intersections/specific search cases.
			</small>
			<hr />
			<div>
				Generators
				<input type="number" v-model.number="settings.num_of_generators" />
			</div>
			<small> Number of generators per polygon. </small>
			<hr />

			<div class="flex space-between mb-2">
				<label>From</label>
				<input type="month" v-model="settings.fromDate" min="2007-01" :max="dateToday" />
			</div>
			<div class="flex space-between">
				<label>To</label>
				<input type="month" v-model="settings.toDate" :max="dateToday" />
			</div>
			<hr />

			<Checkbox v-model:checked="settings.checkAllDates" label="Check all dates" />
			<small>
				This will check the dates of nearby coverage (the dates shown when you click the time machine/clock
				icon). This is helpful for finding coverage within a
				specific timeframe.
			</small>
			<hr />

			<Checkbox v-model:checked="settings.hideMarkers" label="Hide markers" />
		</div>

		<Button v-if="canBeStarted" @click="handleClickStart" :class="state.started ? 'bg-danger' : 'bg-success'"
			:text="state.started ? 'Pause' : 'Start'" title="Space bar/Enter" />
	</div>

	<div v-if="!state.started && hasResults" class="overlay export bottom right">
		<h4 class="center mb-2">Export selection to</h4>
		<div class="flex gap">
			<CopyToClipboard :selection="selected" />
			<ExportToJSON :selection="selected" />
			<ExportToGeoJSON :selection="selected" />
			<ExportToCSV :selection="selected" />
		</div>
	</div>
</template>

<script setup>
import { onMounted, ref, reactive, computed } from "vue";
import Button from "@/components/Elements/Button.vue";
import Checkbox from "@/components/Elements/Checkbox.vue";
import Spinner from "@/components/Elements/Spinner.vue";
import Logo from "@/components/Elements/Logo.vue";

import CopyToClipboard from "@/components/copyToClipboard.vue";
import ExportToJSON from "@/components/exportToJSON.vue";
import ExportToGeoJSON from "@/components/exportToGeoJSON.vue";
import ExportToCSV from "@/components/exportToCSV.vue";

import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "@/assets/leaflet-draw/leaflet.draw.js";
import "@/assets/leaflet-draw/leaflet.draw.css";
import marker from "@/assets/images/marker-icon.png";

import booleanPointInPolygon from "@turf/boolean-point-in-polygon";

import SVreq from "@/utils/SVreq";
import borders from "@/utils/borders.json";

const state = reactive({
	started: false,
	polygonID: 0,
});

const dateToday = new Date().getFullYear() + "-" + ("0" + (new Date().getMonth() + 1)).slice(-2);

const settings = reactive({
	radius: 500,
	rejectUnofficial: true,
	rejectNoDescription: true,
	rejectDateless: true,
	rejectDuplicates: true,
	adjustHeading: true,
	headingDeviation: 0,
	randomHeadingDeviation: false,
	adjustPitch: true,
	pitchDeviation: 10,
	rejectByYear: false,
	fromDate: "2009-01",
	toDate: dateToday,
	checkAllDates: false,
	num_of_generators: 1,
	getIntersection: false,
	hideMarkers: false,
	deadEndsOnly: false,
	lookBackwards: false,
});

const error = ref("");

const select = ref("Select a country or draw a polygon");
const selected = ref([]);
const imported = ref(0);
const canBeStarted = computed(() => selected.value.some((country) => country.found.length < country.nbNeeded));
const hasResults = computed(() => selected.value.some((country) => country.found.length > 0));
const hashCoords = (res) => { return res.lat + "," + res.lng };

let map;
const ImportedPolygonsLayer = new L.FeatureGroup();
const customPolygonsLayer = new L.FeatureGroup();
const markerLayer = L.layerGroup();
const geojson = L.geoJson(borders, {
	style: style,
	onEachFeature: onEachFeature,
});

const roadmapBaseLayer = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m2!1e0!2sm!3m5!2sen!3sus!5e1105!12m1!1e3!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0");
const roadmapLabelsLayer = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m2!1e0!2sm!3m5!2sen!3sus!5e1105!12m1!1e15!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0",
	{ pane: "labelPane" });
const roadmapLayer = L.layerGroup([roadmapBaseLayer, roadmapLabelsLayer]);

const satelliteBaseLayer = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m2!1e1!2sm!3m3!2sen!3sus!5e1105!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0");
const satelliteLabelsLayer = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m2!1e0!2sm!3m5!2sen!3sus!5e1105!12m1!1e4!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0",
	{ pane: "labelPane" });
const satelliteLayer = L.layerGroup([satelliteBaseLayer, satelliteLabelsLayer]);

const terrainBaseLayer = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m2!1e0!2sm!2m1!1e4!3m7!2sen!3sus!5e1105!12m1!1e67!12m1!1e3!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0");
const terrainLabelsLayer = roadmapLabelsLayer;
const terrainLayer = L.layerGroup([terrainBaseLayer, terrainLabelsLayer]);

const osmLayer = L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", { attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors' });

const gsvLayer = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m8!1e2!2ssvv!4m2!1scc!2s*211m3*211e2*212b1*213e2*211m3*211e3*212b1*213e2*212b1*214b1!4m2!1ssvl!2s*211b0*212b1!3m8!2sen!3sus!5e1105!12m4!1e68!2m2!1sset!2sRoadmap!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0");
const gsvLayer2 = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m8!1e2!2ssvv!4m2!1scc!2s*211m3*211e2*212b1*213e2*212b1*214b1!4m2!1ssvl!2s*211b0*212b1!3m8!2sen!3sus!5e1105!12m4!1e68!2m2!1sset!2sRoadmap!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0");
const gsvLayer4 = L.tileLayer("https://www.google.com/maps/vt?pb=!1m7!8m6!1m3!1i{z}!2i{x}!3i{y}!2i9!3x1!2m8!1e2!2ssvv!4m2!1scc!2s*211m3*211e3*212b1*213e2*212b1*214b1!4m2!1ssvl!2s*211b0*212b1!3m8!2sen!3sus!5e1105!12m4!1e68!2m2!1sset!2sRoadmap!4e0!5m4!1e0!8m2!1e1!1e1!6m6!1e12!2i2!11e0!39b0!44e0!50e0");
const baseMaps = {
	Roadmap: roadmapLayer,
	Satellite: satelliteLayer,
	Terrain: terrainLayer,
	OSM: osmLayer
};
const overlayMaps = {
	"Google Street View": gsvLayer,
	"Google Street View Official Only": gsvLayer2,
	"Unofficial coverage only": gsvLayer4,
};

const drawControl = new L.Control.Draw({
	position: "bottomleft",
	draw: {
		polyline: false,
		marker: false,
		circlemarker: false,
		circle: false,
		polygon: {
			allowIntersection: false,
			drawError: {
				color: "#e1e100",
				message: "<strong>Polygon draw does not allow intersections!<strong> (allowIntersection: false)",
			},
			shapeOptions: { color: "#5d8ce3" },
		},
		rectangle: { shapeOptions: { color: "#5d8ce3" } },
	},
	edit: { featureGroup: customPolygonsLayer },
});

// Import
const loadFromClipboard = () => {
	navigator.clipboard
		.readText()
		.then((data) => {
			checkJSON(data);
		})
		.catch((err) => {
			error.value = "Something went wrong.";
		});
};

const loadFromJSON = (e) => {
	const files = e.target.files || e.dataTransfer.files;
	if (!files.length) return;
	readFile(files[0]);
};

const readFile = (file) => {
	const reader = new FileReader();
	reader.onload = (e) => {
		checkJSON(e.target.result);
	};
	reader.readAsText(file);
};

//const hasLatLng = (objectArray) => objectArray.every(obj => obj.hasOwnProperty('lat')) && objectArray.every(obj => obj.hasOwnProperty('lng'));

//const hasCoordinates = (objectArray) => objectArray.every(obj => {obj.hasOwnProperty('coorinates') && hasLatLng(obj)});

const checkJSON = (data) => {
	try {
		let mapData = JSON.parse(data);
		let customgeojson = L.geoJson(mapData, {
			style: customPolygonStyle,
			onEachFeature: function(feature, layer) {
				imported.value ++;
				layer.found = []
				layer.coordSet = new Set();
				layer.nbNeeded = 100;
				layer.setStyle(customPolygonStyle());
				layer.setStyle(highlighted());
				layer.on("mouseover", (e) => highlightFeature(e));
				layer.on("mouseout", (e) => resetHighlight(e));
				layer.on("click", (e) => selectCountry(e));
				layer.feature.properties.name = `Imported Polygon `+ imported.value;
				ImportedPolygonsLayer.addLayer(layer);
			}
		});
		error.value = "";
		state.loaded = true;
	} catch (err) {
		console.log(err)
		state.loaded = false;
		error.value = "Invalid map data";
	}
};

onMounted(() => {
	map = L.map("map", {
		attributionControl: false,
		center: [0, 0],
		zoom: 2,
		zoomControl: false,
		worldCopyJump: true,
		// layers: [L.tileLayer("https://{s}.google.com/vt/lyrs=m&hl=en&x={x}&y={y}&z={z}", { subdomains: ["mt0", "mt1", "mt2", "mt3"], type: "roadmap" })],
	});

	map.createPane("labelPane");
	map.getPane("labelPane").style.zIndex = 300;

	geojson.addTo(map);
	ImportedPolygonsLayer.addTo(map);
	customPolygonsLayer.addTo(map);
	roadmapLayer.addTo(map);
	gsvLayer2.addTo(map);
	L.control.layers(baseMaps, overlayMaps, { position: "bottomleft" }).addTo(map);

	markerLayer.addTo(map);
	map.addControl(drawControl);

	map.on("draw:created", (e) => {
		state.polygonID++;
		const polygon = e.layer;
		polygon.feature = e.layer.toGeoJSON();
		polygon.found = [];
		polygon.coordSet = new Set();
		polygon.nbNeeded = 100;
		polygon.feature.properties.name = `Custom polygon ${state.polygonID}`;
		polygon.setStyle(customPolygonStyle());
		polygon.setStyle(highlighted());
		polygon.on("mouseover", (e) => highlightFeature(e));
		polygon.on("mouseout", (e) => resetHighlight(e));
		polygon.on("click", (e) => selectCountry(e));
		customPolygonsLayer.addLayer(polygon);
		selected.value.push(polygon);
	});
	map.on("draw:edited", (e) => {
		e.layers.eachLayer((layer) => {
			const polygon = layer;
			polygon.feature = layer.toGeoJSON();
			const index = selected.value.findIndex((x) => x.feature.properties.name === layer.feature.properties.name);
			if (index != -1) selected.value[index] = polygon;
		});
	});
	map.on("draw:deleted", (e) => {
		e.layers.eachLayer((layer) => {
			const index = selected.value.findIndex((x) => x.feature.properties.name === layer.feature.properties.name);
			if (index != -1) selected.value.splice(index, 1);
		});
	});

	// Fix hard reload issue
	const mapDiv = document.getElementById("map");
	const resizeObserver = new ResizeObserver(() => {
		map.invalidateSize();
	});
	resizeObserver.observe(mapDiv);
});

const handleRadiusInput = (e) => {
	const value = parseInt(e.target.value);
	if (!value || value < 50) {
		settings.radius = 50;
	} else if (value > 1000000) {
		settings.radius = 1000000;
	}
};

const myIcon = L.icon({
	iconUrl: marker,
	iconAnchor: [12, 41],
});

// Process
document.onkeydown = () => {
	if (window.event.keyCode == "13" || window.event.keyCode == "32") {
		handleClickStart();
	}
};
const handleClickStart = () => {
	state.started = !state.started;
	start();
};

const start = async () => {
	const generator = [];
	for (let polygon of selected.value) {
		for (let i = 0; i < settings.num_of_generators; i++) {
			generator.push(generate(polygon));
		}
	}
	await Promise.all(generator);
	state.started = false;
};

Array.prototype.chunk = function (n) {
	if (!this.length) {
		return [];
	}
	return [this.slice(0, n)].concat(this.slice(n).chunk(n));
};

const generate = async (country) => {
	return new Promise(async (resolve) => {
		while (country.found.length < country.nbNeeded) {
			if (!state.started) return;
			country.isProcessing = true;
			const randomCoords = [];
			while (randomCoords.length < country.nbNeeded) {
				const point = randomPointInPoly(country);
				if (booleanPointInPolygon([point.lng, point.lat], country.feature)) {
					randomCoords.push(point);
				}
			}
			for (let locationGroup of randomCoords.chunk(100)) {
				const responses = await Promise.allSettled(locationGroup.map((l) => SVreq(l, settings)));
				for (let res of responses) {
					if (res.status === "fulfilled" && country.found.length < country.nbNeeded) {
						let locHash = hashCoords(res.value);
						if (!country.coordSet.has(locHash) || !settings.rejectDuplicates) {
							country.found.push(res.value);
							country.coordSet.add(locHash);
							if(!settings.hideMarkers){
								L.marker([res.value.lat, res.value.lng], { icon: myIcon })
									.on("click", () => {
										window.open(
											`https://www.google.com/maps/@?api=1&map_action=pano&viewpoint=${res.value.lat},${res.value.lng}
											${res.value.heading ? "&heading=" + res.value.heading : ""}
											${res.value.pitch ? "&pitch=" + res.value.pitch : ""}`,
											"_blank"
										);
									})
									.addTo(markerLayer);
							}
						}
					}
				}
			}
		}
		resolve();
		country.isProcessing = false;
	});
};

const randomPointInPoly = (polygon) => {
	const bounds = polygon.getBounds();
	const x_min = bounds.getEast();
	const x_max = bounds.getWest();
	const y_min = bounds.getSouth();
	const y_max = bounds.getNorth();

	const lat = (Math.asin(Math.random() * (Math.sin((y_max * Math.PI) / 180) - Math.sin((y_min * Math.PI) / 180)) + Math.sin((y_min * Math.PI) / 180)) * 180) / Math.PI;
	const lng = x_min + Math.random() * (x_max - x_min);
	return { lat, lng };
};

// Map features
function onEachFeature(feature, layer) {
	layer.on({
		mouseover: highlightFeature,
		mouseout: resetHighlight,
		click: selectCountry,
	});
}

function selectCountry(e) {
	if (state.started) return;
	const country = e.target;
	const index = selected.value.findIndex((x) => x.feature.properties.name === country.feature.properties.name);
	if (index == -1) {
		if (!country.found) country.found = [];
		if (!country.coordSet) country.coordSet = new Set();
		if (!country.nbNeeded) country.nbNeeded = 100;
		country.setStyle(highlighted());

		selected.value.push(country);
	} else {
		selected.value.splice(index, 1);
		resetHighlight(e);
	}
}

function selectAllCountries() {
	selected.value = geojson.getLayers().map((country) => {
		if (!country.found) country.found = [];
		if (!country.coordSet) country.coordSet = new Set();
		if (!country.nbNeeded) country.nbNeeded = 100;
		return country;
	});
	geojson.setStyle(highlighted);
}

function selectAllImports() {
	selected.value = ImportedPolygonsLayer.getLayers().map((country) => {
		if (!country.found) country.found = [];
		if (!country.coordSet) country.coordSet = new Set();
		if (!country.nbNeeded) country.nbNeeded = 100;
		return country;
	});
	ImportedPolygonsLayer.setStyle(highlighted);
}

function deselectAll() {
	selected.value.length = 0;
	geojson.setStyle(style());
	customPolygonsLayer.setStyle(customPolygonStyle());
	ImportedPolygonsLayer.setStyle(customPolygonStyle())
}

function highlightFeature(e) {
	if (state.started) return;
	const layer = e.target;
	const index = selected.value.findIndex((x) => x.feature.properties.name === layer.feature.properties.name);
	if (index == -1) {
		layer.setStyle(highlighted());
	}
	select.value = `${layer.feature.properties.name} ${layer.found ? "(" + layer.found.length + ")" : "(0)"}`;
}
function resetHighlight(e) {
	const layer = e.target;
	const index = selected.value.findIndex((x) => x.feature.properties.name === layer.feature.properties.name);
	if (index == -1) {
		layer.setStyle(removeHighlight());
	}
	select.value = "Select a country or draw a polygon";
}
function style() {
	return {
		opacity: 0,
		fillOpacity: 0,
	};
}
function customPolygonStyle() {
	return {
		weight: 2,
		opacity: 1,
		color: getRandomColor(),
		fillOpacity: 0,
	};
}
function highlighted() {
	return {
		fillColor: getRandomColor(),
		fillOpacity: 0.6,
	};
}
function removeHighlight() {
	return { fillOpacity: 0 };
}
function getRandomColor() {
	const red = Math.floor(((1 + Math.random()) * 256) / 2);
	const green = Math.floor(((1 + Math.random()) * 256) / 2);
	const blue = Math.floor(((1 + Math.random()) * 256) / 2);
	return "rgb(" + red + ", " + green + ", " + blue + ")";
}
function clearMarkers() {
	markerLayer.clearLayers();
}
</script>

<style>
@import "@/assets/main.css";

#map {
	z-index: 0;
	height: 100vh;
}

.leaflet-container {
	background-color: #2c2c2c;
}

.overlay {
	position: absolute;
}

.select,
.selected,
.settings,
.export {
	padding: var(--space-2);
	border-radius: 5px;
	background: rgba(0, 0, 0, 0.7);
	box-shadow: 0 0 2px rgba(0, 0, 0, 0.4);
}

.selected {
	max-height: calc(100vh - 390px);
	overflow: auto;
}

.settings {
	max-width: 375px;
	max-height: calc(100vh - 180px);
	overflow: auto;
}

.export {
	min-width: 375px;
}

.line {
	line-height: 1.5rem;
}
</style>
