<script setup lang="ts">
import { IonLoading } from "@ionic/vue";
defineOptions({
    components: { IonLoading },
});
import { NativeGeocoder } from "@capgo/nativegeocoder";
import { Capacitor } from "@capacitor/core";
import { ref, onMounted, watch } from "vue";
import { LMap, LTileLayer, LMarker } from "@vue-leaflet/vue-leaflet";
import "leaflet/dist/leaflet.css";
import { Geolocation } from "@capacitor/geolocation";
import { Preferences } from "@capacitor/preferences";
import L from "leaflet";

let zoom = 17;
let center_lat = 48.060528;
let center_lng = 8.534415;
const center = ref({ lat: center_lat, lng: center_lng });
const showLoading = ref(false);
const userPosition = ref(null);
const searchQuery = ref("");
const searchPosition = ref(null);
const mapRef = ref(null);

// Marker-Icons
const userIcon = new L.Icon({
    iconUrl: "/user-marker.png",
    iconSize: [25, 41],
    iconAnchor: [12, 41],
    shadowUrl: "https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png",
    shadowSize: [41, 41],
});

const searchIcon = new L.Icon({
    iconUrl: "/search-marker.png",
    iconSize: [25, 41],
    iconAnchor: [12, 41],
    shadowUrl: "https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png",
    shadowSize: [41, 41],
});

onMounted(async () => {
    const c_lat = await Preferences.get({ key: "center_lat" });
    const c_lng = await Preferences.get({ key: "center_lng" });
    const z = await Preferences.get({ key: "zoom" });

    if (c_lat != null && c_lng != null) {
        try {
            center_lat = parseFloat(c_lat.value);
            center_lng = parseFloat(c_lng.value);
        } catch (err) {
            console.warn("Error when getting the Preferences:", err);
        }
    }
    if (z != null) {
        try {
            zoom = parseInt(z.value);
        } catch (err) {
            console.warn("Error when getting the Preferences:", err);
        }
    }

    // Make sure to wait for a moment before manipulating the map
    setTimeout(() => {
        const map = getLeafletMap();
        if (!map) return;

        // Create a valid LatLng object with your variables
        const l_obj = L.latLng(center_lat, center_lng);

        // Set the map view with proper fallbacks
        if (l_obj.lat !== 0 || l_obj.lng !== 0) {
            map.setView(l_obj, zoom);
        } else {
            // Fallback to default location if nothing is saved
            map.setView(L.latLng(48.060528, 8.534415), 17);
        }
    }, 100);
});

function getLeafletMap(): L.Map | null {
    return (mapRef.value?.leafletObject ??
        mapRef.value?.mapObject ??
        null) as L.Map | null;
}

let saveTimer: any = null;

function onMapMoved() {
    const map = getLeafletMap();
    if (!map) return;

    const z = map.getZoom();
    const c = map.getCenter();

    zoom = z;
    center_lat = c.lat;
    center_lng = c.lng;

    console.log(
        "set zoom:" + z + "set lat:" + center_lat + "set lng:" + center_lng,
    );

    clearTimeout(saveTimer);
    saveTimer = setTimeout(async () => {
        await Preferences.set({
            key: "center_lat",
            value: center_lat.toString(),
        });
        await Preferences.set({
            key: "center_lng",
            value: center_lng.toString(),
        });
        await Preferences.set({ key: "zoom", value: zoom.toString() });
    }, 150);
}

const isWeb = Capacitor.getPlatform() === "web";

async function geocodeAddress(
    address: string,
): Promise<[number, number] | null> {
    showLoading.value = true;
    try {
        console.log("[GEOCODE] Anfrage für:", address);

        // 1) Web-Fallback (nur im Browser)
        if (isWeb) {
            console.log("[GEOCODE] Web-Fallback via OSM");
            const resp = await fetch(
                `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(address)}`,
                {
                    headers: {
                        "User-Agent": "KartenApp/1.0 (mail@beispiel.de)",
                    },
                },
            );
            if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
            const data = await resp.json();
            console.log("[GEOCODE] OSM-Daten:", data);
            if (Array.isArray(data) && data.length > 0) {
                return [parseFloat(data[0].lat), parseFloat(data[0].lon)];
            }
            return null;
        }

        // 2) Native-Geocoding auf Device/Emulator
        const raw = await NativeGeocoder.forwardGeocode({
            addressString: address,
            maxResults: 5,
            defaultLocale: "de",
        });
        console.log(
            "[GEOCODE] Native-Rohdaten (stringified):",
            JSON.stringify(raw),
        );

        // Normalisiere in ein Array von Treffern
        let hits: any[] = [];
        if (Array.isArray(raw)) {
            hits = raw;
        } else if (raw && Array.isArray((raw as any).addresses)) {
            hits = (raw as any).addresses;
        } else if (raw && Array.isArray((raw as any).coordinates)) {
            hits = (raw as any).coordinates;
        } else {
            console.warn("[GEOCODE] Unbekanntes Format, ignoriere raw");
        }
        console.log("[GEOCODE] Parsed Treffer:", hits);

        // 3) Ersten Treffer auswerten
        if (hits.length > 0) {
            const first = hits[0] as any;
            // Unterstütze unterschiedliche Property-Namen:
            const lat =
                first.latitude ??
                first.lat ??
                (Array.isArray(first) ? first[0] : null);
            const lng =
                first.longitude ??
                first.lon ??
                (Array.isArray(first) ? first[1] : null);
            if (lat != null && lng != null) {
                console.log("[GEOCODE] Gefundene Koords:", lat, lng);
                return [Number(lat), Number(lng)];
            }
            console.warn("[GEOCODE] Treffer ohne lat/lng:", first);
        }

        console.warn("[GEOCODE] Keine gültigen Treffer für Adresse");
        return null;
    } catch (error) {
        console.error("[GEOCODE] Exception:", error);
        return null;
    } finally {
        showLoading.value = false;
    }
}

async function handleSearch() {
    if (!searchQuery.value.trim()) return;
    const coords = await geocodeAddress(searchQuery.value);
    if (coords) {
        const map = getLeafletMap();
        if (!map) return;
        const [lat, lng] = coords;
        const l_obj = L.latLng(lat, lng);
        zoom = 17;
        map.setView(l_obj, zoom);
        searchPosition.value = [lat, lng];
        onMapMoved();
    }
}

async function requestLocationPermission() {
    try {
        const status = await Geolocation.checkPermissions();
        if (status.location === "granted") return true;
        const requestStatus = await Geolocation.requestPermissions();
        return requestStatus.location === "granted";
    } catch (error) {
        console.error("Fehler beim Anfordern der Berechtigung:", error);
        return false;
    }
}

async function locateUser() {
    showLoading.value = true;
    try {
        const hasPermission = await requestLocationPermission();
        if (!hasPermission) return;

        const pos = await Geolocation.getCurrentPosition();
        const lat = pos.coords.latitude;
        const lng = pos.coords.longitude;

        const map = getLeafletMap();
        if (!map) return;
        const l_obj = L.latLng(lat, lng);
        zoom = 17;
        map.setView(l_obj, zoom);
        userPosition.value = [lat, lng];
        onMapMoved();
    } catch (e) {
        console.error(e);
    } finally {
        showLoading.value = false;
    }
}

const attributionContent =
    '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors';
</script>

<template>
    <div class="app-container">
        <div class="search-container">
            <input
                v-model="searchQuery"
                @keyup.enter="handleSearch"
                placeholder="Adresse oder Koordinaten eingeben..."
                class="search-input"
            />
            <button @click="handleSearch" class="search-button">Suchen</button>
        </div>

        <div class="location-button-container">
            <button @click="locateUser" class="location-button">
                <span>📍</span> Meinen Standort finden
            </button>
        </div>

        <l-map
            ref="mapRef"
            v-model:zoom="zoom"
            v-model:center="center"
            :use-global-leaflet="false"
            @moveend="onMapMoved"
            @zoomend="onMapMoved"
            style="height: 100vh; width: 100%"
        >
            <l-tile-layer
                url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
                attribution="&copy; OpenStreetMap contributors"
            />
            <l-marker
                v-if="userPosition"
                :lat-lng="userPosition"
                :icon="userIcon"
            />
            <l-marker
                v-if="searchPosition"
                :lat-lng="searchPosition"
                :icon="searchIcon"
            />
        </l-map>
    </div>

    <ion-loading :is-open="showLoading" message="Lade..." />
</template>

<style scoped>
.app-container {
    position: relative;
    height: 100vh;
    width: 100%;
}

.search-container {
    position: absolute;
    top: 45px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 1000;
    display: flex;
    width: 75%;
    max-width: 600px;
    background-color: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(5px);
    border-radius: 8px;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
}

.search-input {
    flex-grow: 1;
    padding: 12px 20px;
    font-size: 16px;
    border: none;
    border-radius: 8px 0 0 8px;
}

.search-button {
    padding: 0 20px;
    background-color: #4caf50;
    color: white;
    border: none;
    border-radius: 0 8px 8px 0;
    cursor: pointer;
    font-size: 16px;
}

.search-button:hover {
    background-color: #45a049;
}

.location-button-container {
    position: absolute;
    top: 100px;
    right: 20px;
    z-index: 1000;
}

.location-button {
    display: flex;
    align-items: center;
    padding: 10px 15px;
    background-color: rgba(52, 152, 219, 0.85);
    color: white;
    border: none;
    border-radius: 22px;
    cursor: pointer;
    font-size: 16px;
    gap: 8px;
    backdrop-filter: blur(4px);
}

.location-button:hover {
    background-color: rgba(41, 128, 185, 0.95);
}
</style>
