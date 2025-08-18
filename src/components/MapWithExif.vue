<template>
  <div>
    <h1>EXIF Geotagging and Map Markers</h1>
    <input type="file" @change="handleFileInput" accept="image/*" />
    <div id="map" ref="map"></div>
  </div>
</template>

<script>
import L from "leaflet";
import EXIF from "exif-js";
import "leaflet/dist/leaflet.css";

// Fix Leaflet's default icon issue
delete L.Icon.Default.prototype._getIconUrl;
L.Icon.Default.mergeOptions({
  iconRetinaUrl: require("leaflet/dist/images/marker-icon-2x.png"),
  iconUrl: require("leaflet/dist/images/marker-icon.png"),
  shadowUrl: require("leaflet/dist/images/marker-shadow.png"),
});

export default {
  name: "MapWithExif",
  data() {
    return {
      map: null, // Leaflet map instance
    };
  },
  mounted() {
    // Initialize the map when the component is mounted
    this.map = L.map(this.$refs.map).setView([0, 0], 2); // Default to world view
    L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: "© OpenStreetMap contributors",
    }).addTo(this.map);

    // Load the example image
    this.loadExampleImage();
  },
  methods: {
    // Convert EXIF GPS data to decimal degrees
    convertToDecimalDegrees(gpsData, ref) {
      const [degrees, minutes, seconds] = gpsData;
      let decimal = degrees + minutes / 60 + seconds / 3600;
      if (ref === "S" || ref === "W") {
        decimal *= -1; // South and West are negative
      }
      return decimal;
    },
    // Handle file input and extract EXIF metadata
    handleFileInput(event) {
      const file = event.target.files[0];
      if (!file) return;

      // Create a URL for the uploaded image
      const imageUrl = URL.createObjectURL(file);

      // Read EXIF data from the image
      EXIF.getData(file, () => {
        const exifData = EXIF.getAllTags(file);

        // Extract latitude and longitude
        const latitude = exifData.GPSLatitude
          ? this.convertToDecimalDegrees(exifData.GPSLatitude, exifData.GPSLatitudeRef)
          : null;
        const longitude = exifData.GPSLongitude
          ? this.convertToDecimalDegrees(exifData.GPSLongitude, exifData.GPSLongitudeRef)
          : null;

        if (latitude && longitude) {
          // Add a marker to the map
          L.marker([latitude, longitude])
            .addTo(this.map)
            .bindPopup(`
              <div>
                <p><strong>Latitude:</strong> ${latitude}</p>
                <p><strong>Longitude:</strong> ${longitude}</p>
                <img src="${imageUrl}" alt="Uploaded Image" style="width: 200px; height: auto; border-radius: 5px;" />
              </div>
            `)
            .openPopup();

          // Center the map on the marker
          this.map.setView([latitude, longitude], 13);
        } else {
          alert("No GPS data found in the image.");
        }
      });
    },
    // Load the example image and extract its EXIF metadata
    loadExampleImage() {
      const exampleImageUrl = "/example.jpg"; // Path to the example image in the public folder

      // Fetch the example image as a Blob
      fetch(exampleImageUrl)
        .then((response) => response.blob())
        .then((blob) => {
          const imageUrl = URL.createObjectURL(blob); // Create a URL for the example image

          EXIF.getData(blob, () => {
            const exifData = EXIF.getAllTags(blob);

            // Extract latitude and longitude
            const latitude = exifData.GPSLatitude
              ? this.convertToDecimalDegrees(exifData.GPSLatitude, exifData.GPSLatitudeRef)
              : null;
            const longitude = exifData.GPSLongitude
              ? this.convertToDecimalDegrees(exifData.GPSLongitude, exifData.GPSLongitudeRef)
              : null;

            if (latitude && longitude) {
              // Add a marker to the map
              L.marker([latitude, longitude])
                .addTo(this.map)
                .bindPopup(`
                  <div>
                    <p><strong>Latitude:</strong> ${latitude}</p>
                    <p><strong>Longitude:</strong> ${longitude}</p>
                    <img src="${imageUrl}" alt="Example Image" style="width: 200px; height: auto; border-radius: 5px;" />
                  </div>
                `)
                .openPopup();

              // Center the map on the marker
              this.map.setView([latitude, longitude], 13);
            } else {
              console.warn("No GPS data found in the example image.");
            }
          });
        })
        .catch((error) => {
          console.error("Error loading the example image:", error);
        });
    },
  },
};
</script>

<style>
#map {
  height: 90vh; /* Set the height of the map */
  width: 100%;   /* Set the width of the map */
}
</style>
