<template>
  <div>
    <h1>EXIF Geotagging and Map Markers</h1>
    <input type="file" @change="handleFileInput" accept="image/*" multiple />
    <l-map
      style="height: 90vh; width: 100%;"
      :zoom="zoom"
      :center="center"
      @ready="onMapReady"
    >
      <l-tile-layer
        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        attribution="© OpenStreetMap contributors"
      />
      <l-marker
        v-for="(marker, index) in markers"
        :key="index"
        :lat-lng="marker.position"
      >
        <l-popup>
          <div>
            <p><strong>Latitude:</strong> {{ marker.position[0] }}</p>
            <p><strong>Longitude:</strong> {{ marker.position[1] }}</p>
            <img
              :src="marker.imageUrl"
              alt="Uploaded Image"
              style="width: 200px; height: auto; border-radius: 5px;"
            />
          </div>
        </l-popup>
      </l-marker>
    </l-map>
  </div>
</template>

<script>
import { LMap, LTileLayer, LMarker, LPopup } from "@vue-leaflet/vue-leaflet";
import ExifReader from "exifreader";

export default {
  name: "MapWithExif",
  components: {
    LMap,
    LTileLayer,
    LMarker,
    LPopup,
  },
  data() {
    return {
      center: [0, 0], // Default map center
      zoom: 2, // Default zoom level
      markers: [], // Array to store marker data
    };
  },
  methods: {
    async handleFileInput(event) {
      const files = Array.from(event.target.files); // Convert FileList to an array
      if (!files.length) return;

      const newMarkers = []; // Temporary array to store new markers

      for (const file of files) {
        try {
          // Read the file as an ArrayBuffer
          const arrayBuffer = await file.arrayBuffer();

          // Parse EXIF data using ExifReader
          const tags = ExifReader.load(arrayBuffer);

          // Extract GPS data directly in decimal format
          const latitude = tags.GPSLatitude?.description;
          const longitude = tags.GPSLongitude?.description;

          if (latitude && longitude) {
            // Create a URL for the uploaded image
            const imageUrl = URL.createObjectURL(file);

            // Add the marker data to the array
            newMarkers.push({
              position: [latitude, longitude],
              imageUrl,
            });
          } else {
            console.warn(`No GPS data found in the image: ${file.name}`);
          }
        } catch (error) {
          console.error(`Error reading EXIF data from file ${file.name}:`, error);
        }
      }

      // Update the markers array and adjust the map view
      if (newMarkers.length > 0) {
        this.markers.push(...newMarkers);
        this.fitToBounds();
      }
    },
    onMapReady(mapInstance) {
      this.map = mapInstance;
    },
    fitToBounds() {
      if (this.markers.length > 0) {
        const bounds = this.markers.map((marker) => marker.position);
        this.map.fitBounds(bounds); // Adjust the map view to fit all markers
      }
    },
  },
};
</script>

<style>
/* Add any additional styles here */
</style>
