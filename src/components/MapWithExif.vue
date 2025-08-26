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
          console.log(`EXIF data for ${file.name}:`, tags);

                // Extract GPS data
          const latitude = tags.GPSLatitude?.description;
          const latitudeRef = tags.GPSLatitudeRef?.description; // 'N' or 'S'
          const longitude = tags.GPSLongitude?.description;
          const longitudeRef = tags.GPSLongitudeRef?.description; // 'E' or 'W'
          
          console.log("Raw Latitude:", latitude);
          console.log("Latitude Reference:", latitudeRef);
          console.log("Raw Longitude:", longitude);
          console.log("Longitude Reference:", longitudeRef);
                // Interpret latitude and longitude based on their references
          const interpretedLatitude =
            latitude && latitudeRef
              ? (latitudeRef === "South latitude" ? -1 : 1) * latitude
              : null;
          const interpretedLongitude =
            longitude && longitudeRef
              ? (longitudeRef === "West longitude" ? -1 : 1) * longitude
              : null;

          console.log("Interpreted Latitude:", interpretedLatitude);
          console.log("Interpreted Longitude:", interpretedLongitude);

          if (interpretedLatitude !== null && interpretedLongitude !== null) {
            // Create a URL for the uploaded image
            const imageUrl = URL.createObjectURL(file);

            // Add the marker data to the array
            newMarkers.push({
              position: [interpretedLatitude, interpretedLongitude],
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
