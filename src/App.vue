<script setup>
import { reactive, ref, onMounted } from 'vue';

// --- STATE ---
let crime_url = ref('');
let dialog_err = ref(false);

let map = reactive({
    leaflet: null,
    center: { lat: 44.955139, lng: -93.102222, address: '' },
    zoom: 12,
    bounds: { nw: {lat: 45.008206, lng: -93.217977}, se: {lat: 44.883658, lng: -92.993787} },
    neighborhood_markers: [
        {location: [44.942068, -93.020521], neighborhood_number: 1, marker: null},
        {location: [44.977413, -93.025156], neighborhood_number: 2, marker: null},
        {location: [44.931244, -93.079578], neighborhood_number: 3, marker: null},
        {location: [44.956192, -93.060189], neighborhood_number: 4, marker: null},
        {location: [44.978883, -93.068163], neighborhood_number: 5, marker: null},
        {location: [44.975766, -93.113887], neighborhood_number: 6, marker: null},
        {location: [44.959639, -93.121271], neighborhood_number: 7, marker: null},
        {location: [44.947700, -93.128505], neighborhood_number: 8, marker: null},
        {location: [44.930276, -93.119911], neighborhood_number: 9, marker: null},
        {location: [44.982752, -93.147910], neighborhood_number: 10, marker: null},
        {location: [44.963631, -93.167548], neighborhood_number: 11, marker: null},
        {location: [44.973971, -93.197965], neighborhood_number: 12, marker: null},
        {location: [44.949043, -93.178261], neighborhood_number: 13, marker: null},
        {location: [44.934848, -93.176736], neighborhood_number: 14, marker: null},
        {location: [44.913106, -93.170779], neighborhood_number: 15, marker: null},
        {location: [44.937705, -93.136997], neighborhood_number: 16, marker: null},
        {location: [44.949203, -93.093739], neighborhood_number: 17, marker: null},
    ]
});

let crimes = ref([]);
let newIncident = reactive({
    case_number: '',
    date_time: '',
    code: '',
    incident: '',
    police_grid: '',
    neighborhood_number: '',
    block: ''
});
let incidentError = ref('');

// --- FUNCTIONS ---
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        loadCrimesInBounds();
    } else {
        dialog_err.value = true;
    }
}

async function goToLocation() {
    let value = map.center.address.trim();
    if (/^\-?\d+(\.\d+)?,\s*\-?\d+(\.\d+)?$/.test(value)) {
        // lat,lng input
        let [lat, lng] = value.split(',').map(Number);
        lat = Math.min(Math.max(lat, map.bounds.se.lat), map.bounds.nw.lat);
        lng = Math.min(Math.max(lng, map.bounds.nw.lng), map.bounds.se.lng);
        map.leaflet.setView([lat, lng], map.zoom);
    } else {
        // address input -> Nominatim
        try {
            let res = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(value)}`);
            let data = await res.json();
            if (data && data.length > 0) {
                let lat = parseFloat(data[0].lat);
                let lng = parseFloat(data[0].lon);
                lat = Math.min(Math.max(lat, map.bounds.se.lat), map.bounds.nw.lat);
                lng = Math.min(Math.max(lng, map.bounds.nw.lng), map.bounds.se.lng);
                map.leaflet.setView([lat, lng], map.zoom);
                map.center.address = `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
            } else {
                alert('Location not found');
            }
        } catch(err) {
            console.error(err);
        }
    }
}

async function loadCrimesInBounds() {
    if (!crime_url.value) return;

    const bounds = map.leaflet.getBounds();
    const nw = bounds.getNorthWest();
    const se = bounds.getSouthEast();

    try {
        let res = await fetch(`${crime_url.value}/incidents?limit=1000`);
        let data = await res.json();
        // Filter by visible map area
        crimes.value = data.filter(c => {
            let n = map.neighborhood_markers.find(n => n.neighborhood_number === c.neighborhood_number);
            if (!n) return false;
            let [lat, lng] = n.location;
            return lat >= se.lat && lat <= nw.lat && lng >= nw.lng && lng <= se.lng;
        });

        // Update markers with crime count
        map.neighborhood_markers.forEach(n => {
            let count = crimes.value.filter(c => c.neighborhood_number === n.neighborhood_number).length;
            if (!n.marker) {
                n.marker = L.marker(n.location).addTo(map.leaflet);
            }
            n.marker.bindPopup(`${count} crimes in this neighborhood`);
        });
    } catch(err) {
        console.error(err);
    }
}

async function submitIncident() {
    if (!newIncident.case_number || !newIncident.date_time || !newIncident.code || !newIncident.incident || !newIncident.police_grid || !newIncident.neighborhood_number || !newIncident.block) {
        incidentError.value = 'All fields must be filled out';
        return;
    }

    try {
        let res = await fetch(`${crime_url.value}/new-incident`, {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(newIncident)
        });
        if (!res.ok) {
            incidentError.value = await res.text();
        } else {
            incidentError.value = '';
            alert('Incident added!');
            loadCrimesInBounds();
        }
    } catch(err) {
        incidentError.value = 'Failed to submit incident';
        console.error(err);
    }
}

// --- ON MOUNT ---
onMounted(() => {
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // District boundaries
    let district_boundary = new L.geoJson().addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
        .then(res => res.json())
        .then(data => data.features.forEach(f => district_boundary.addData(f)))
        .catch(err => console.error(err));

    // Update input on pan/zoom
    map.leaflet.on('moveend', async () => {
        const center = map.leaflet.getCenter();
        map.center.lat = center.lat;
        map.center.lng = center.lng;
        map.center.address = `${center.lat.toFixed(6)}, ${center.lng.toFixed(6)}`;
        try {
            let res = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${center.lat}&lon=${center.lng}`);
            let data = await res.json();
            if (data && data.display_name) map.center.address = data.display_name;
        } catch(err) {}
        loadCrimesInBounds();
    });
});
</script>

<template>
    <!-- API URL Dialog -->
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>

    <!-- Location Input -->
    <div class="location-input">
        <input type="text" v-model="map.center.address" placeholder="Enter address or lat,lng" @keyup.enter="goToLocation" />
        <button @click="goToLocation">Go</button>
    </div>

    <!-- Map -->
    <div id="leafletmap"></div>

    <!-- Crimes Table -->
    <h3>Visible Crimes</h3>
    <table>
        <thead>
            <tr>
                <th>Date</th>
                <th>Time</th>
                <th>Neighborhood</th>
                <th>Incident</th>
            </tr>
        </thead>
        <tbody>
            <tr v-for="c in crimes" :key="c.case_number">
                <td>{{ c.date }}</td>
                <td>{{ c.time }}</td>
                <td>{{ c.neighborhood_number }}</td>
                <td>{{ c.incident }}</td>
            </tr>
        </tbody>
    </table>

    <!-- New Incident Form -->
    <h3>Add New Crime</h3>
    <form @submit.prevent="submitIncident">
        <input v-model="newIncident.case_number" placeholder="Case Number" required />
        <input v-model="newIncident.date_time" type="datetime-local" required />
        <input v-model="newIncident.code" placeholder="Code" required />
        <input v-model="newIncident.incident" placeholder="Incident" required />
        <input v-model="newIncident.police_grid" placeholder="Police Grid" required />
        <input v-model="newIncident.neighborhood_number" placeholder="Neighborhood #" required />
        <input v-model="newIncident.block" placeholder="Block" required />
        <button type="submit">Submit</button>
    </form>
    <p v-if="incidentError" style="color:red">{{ incidentError }}</p>
</template>

<style scoped>
#rest-dialog { width: 20rem; margin-top: 1rem; z-index: 1000; }
#leafletmap { height: 500px; margin: 1rem 0; }
.dialog-header { font-size: 1.2rem; font-weight: bold; }
.dialog-label { font-size: 1rem; }
.dialog-input { font-size: 1rem; width: 100%; }
.dialog-error { font-size: 1rem; color: #D32323; }
.location-input { margin: 0.5rem 0; }
.location-input input { width: 70%; padding: 0.3rem; }
.location-input button { padding: 0.3rem 0.5rem; }
table { width: 100%; border-collapse: collapse; margin: 1rem 0; }
table, th, td { border: 1px solid #ccc; }
th, td { padding: 0.3rem; text-align: left; }
</style>
