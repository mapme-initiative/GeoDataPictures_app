<template>
  <div>
    <h1>EXIF Geotagging and Map Markers</h1>
    <input type="file" @change="handleFileInput" accept="image/*,image/heic,image/heif" multiple />
    <l-map style="height: 80vh; width: 100%;" :zoom="zoom" :center="center" @ready="onMapReady">
      <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        attribution="© OpenStreetMap contributors" />
      <l-tile-layer
        v-if="currentBasemap === 'esri'"
        url="https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}"
        attribution="Tiles © Esri &mdash; Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community"
      />
      <l-marker v-for="(marker, index) in markers" :key="index" :lat-lng="marker.position">
        <l-popup>
          <div>
            <p><strong>Latitude:</strong> {{ marker.position[0] }}</p>
            <p><strong>Longitude:</strong> {{ marker.position[1] }}</p>
            <img :src="marker.imageUrl" alt="Uploaded Image" style="width: 200px; height: auto; border-radius: 5px;" />
          </div>
        </l-popup>
      </l-marker>
    </l-map>

    <!-- Basemap Switcher -->
    <div style="margin-top: 10px;">
      <button @click="switchBasemap('osm')">OpenStreetMap</button>
      <button @click="switchBasemap('esri')">Esri World Imagery</button>
    </div>
  </div>
</template>

<script>
import { LMap, LTileLayer, LMarker, LPopup } from "@vue-leaflet/vue-leaflet";
import ExifReader from "exifreader";
import heic2any from "heic2any";
import piexif from "piexifjs";

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
      currentBasemap: "osm", // Default basemap (OpenStreetMap)
    };
  },
  methods: {
    async handleFileInput(event) {
      const files = Array.from(event.target.files); // Convert FileList to an array
      if (!files.length) return;

      const newMarkers = []; // Temporary array to store new markers

      // Process each file and wait for all asynchronous operations to complete
      await Promise.all(
        files.map(async (file) => {
          try {
            console.log(`Processing file: ${file.name}`);
            const fileExtension = file.name.split('.').pop().toLowerCase();

            let processedFile = file;

            // Check if the file is HEIC/HEIF
            if (fileExtension === "heic" || fileExtension === "heif") {
              console.log(`File identified as HEIC/HEIF: ${file.name}`);

              // Step 1: Extract metadata from the original file
              const originalArrayBuffer = await file.arrayBuffer();
              const originalTags = ExifReader.load(originalArrayBuffer);
              console.log(`Original Metadata for ${file.name}:`, originalTags);

              // Step 2: Convert HEIC/HEIF to JPEG
              const convertedBlob = await heic2any({ blob: file, toType: "image/jpeg" });
              processedFile = new File([convertedBlob], file.name.replace(/\.[^/.]+$/, ".jpg"), {
                type: "image/jpeg",
              });
              console.log(`Conversion successful: ${processedFile.name}`);

              // Step 3: Inject metadata into the converted JPEG
              const reader = new FileReader();
              const metadataInjectedFile = await new Promise((resolve, reject) => {
                reader.onload = function (event) {
                  try {
                    const jpegData = event.target.result; // Base64 string of the JPEG
                    const exifBytes = piexif.dump(originalTags); // Convert original metadata to EXIF bytes
                    const newJpeg = piexif.insert(exifBytes, jpegData); // Inject metadata into the JPEG

                    // Create a Blob from the updated JPEG
                    const updatedBlob = new Blob(
                      [new Uint8Array(atob(newJpeg.split(",")[1]).split("").map((c) => c.charCodeAt(0)))],
                      { type: "image/jpeg" }
                    );

                    const updatedFile = new File([updatedBlob], processedFile.name, { type: "image/jpeg" });
                    console.log(`Metadata injected successfully into: ${updatedFile.name}`);
                    resolve(updatedFile);
                  } catch (error) {
                    reject(error);
                  }
                };
                reader.readAsDataURL(processedFile);
              });

              // Step 4: Extract GPS data from the original metadata
              const latitude = originalTags.GPSLatitude?.description;
              const latitudeRef = originalTags.GPSLatitudeRef?.description; // 'N' or 'S'
              const longitude = originalTags.GPSLongitude?.description;
              const longitudeRef = originalTags.GPSLongitudeRef?.description; // 'E' or 'W';

              console.log("Original Raw Latitude:", latitude);
              console.log("Original Latitude Reference:", latitudeRef);
              console.log("Original Raw Longitude:", longitude);
              console.log("Original Longitude Reference:", longitudeRef);

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
                const imageUrl = URL.createObjectURL(metadataInjectedFile);
                newMarkers.push({
                  position: [interpretedLatitude, interpretedLongitude],
                  imageUrl,
                });
                console.log(`Marker added for file: ${metadataInjectedFile.name}`);
              } else {
                console.warn(`No GPS data found in the original image: ${file.name}`);
              }
            } else {
              console.log(`File is not HEIC/HEIF: ${file.name}`);
              // Process non-HEIC/HEIF files as usual
              const arrayBuffer = await file.arrayBuffer();
              const tags = ExifReader.load(arrayBuffer);
              console.log(`Metadata for ${file.name}:`, tags);

              const latitude = tags.GPSLatitude?.description;
              const latitudeRef = tags.GPSLatitudeRef?.description; // 'N' or 'S'
              const longitude = tags.GPSLongitude?.description;
              const longitudeRef = tags.GPSLongitudeRef?.description; // 'E' or 'W';

              const interpretedLatitude =
                latitude && latitudeRef
                  ? (latitudeRef === "South latitude" ? -1 : 1) * latitude
                  : null;
              const interpretedLongitude =
                longitude && longitudeRef
                  ? (longitudeRef === "West longitude" ? -1 : 1) * longitude
                  : null;

              if (interpretedLatitude !== null && interpretedLongitude !== null) {
                const imageUrl = URL.createObjectURL(file);
                newMarkers.push({
                  position: [interpretedLatitude, interpretedLongitude],
                  imageUrl,
                });
              }
            }
          } catch (error) {
            console.error(`Error processing file ${file.name}:`, error);
          }
        })
      );

      // Update the markers array and adjust the map view
      if (newMarkers.length > 0) {
        this.markers.push(...newMarkers);
        this.fitToBounds();
      } else {
        console.warn("No valid markers to add to the map.");
      }
    },
    switchBasemap(basemap) {
      this.currentBasemap = basemap;
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
