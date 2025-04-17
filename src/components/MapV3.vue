<template>
  <div class="map-container">
    <!-- Map container -->
    <div ref="mapContainer" style="height: 500px; width: 100%"></div>

    <!-- Controls -->
    <div class="controls">
      <i class="fas fa-trash-alt control-icon" @click="clearPoints" title="Clear All Points"></i>
      <i class="fas fa-eye control-icon" @click="fetchGraph" :title="points.length > 0 ? 'Show Connections' : 'No Points to Connect'" :class="{ disabled: points.length === 0 }"></i>
      <i class="fas fa-times control-icon" @click="clearSelection" title="Clear Selection"></i>
    </div>

    <!-- Display selected points -->
    <div v-if="selectedPoints && selectedPoints.length > 0" class="selected-points">
      <p>Selected Points:</p>
      <ul>
        <li v-for="(point, index) in selectedPoints" :key="index">{{ point }}</li>
      </ul>
    </div>

    <!-- Display total distance -->
    <div v-if="totalDistance" class="total-distance">
      <p>{{ totalDistance }}</p>
    </div>
  </div>
</template>

<script>
import { onMounted, ref, watch } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import '@fortawesome/fontawesome-free/css/all.min.css'; // Add Font Awesome for icons

export default {
  name: 'InteractiveMap',
  setup() {
    const mapContainer = ref(null);
    const map = ref(null); // Reference to the Leaflet map
    const pointsLayer = ref(null); // Layer group for points
    const points = ref([]); // Store all points (initialized as an empty array)
    const selectedPoints = ref([]); // Stores [start, end] (initialized as an empty array)
    const graphEdges = ref([]); // Store edges for visualization
    const pathLine = ref(null); // Reference to the polyline for the optimal path
    const totalDistance = ref(null); // To store and display the total distance

    // Initialize map
    onMounted(() => {
      if (!mapContainer.value) {
        console.error("Map container not found!");
        return;
      }

      // Initialize the Leaflet map
      map.value = L.map(mapContainer.value).setView([39.8283, -98.5795], 4);

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
      }).addTo(map.value);

      // Create a layer group for our points
      pointsLayer.value = L.layerGroup().addTo(map.value);

      // Add click event to map
      map.value.on('click', addPointOnClick);
    });

    // Function to add a point where the user clicked
    const addPointOnClick = (e) => {
      const newPoint = {
        id: `Point_${points.value.length + 1}`,
        latlng: e.latlng,
        properties: {
          name: `Point ${points.value.length + 1}`,
          timestamp: new Date().toISOString()
        }
      };

      points.value.push(newPoint);
      updatePointsLayer();
    };

    // Update the points layer with all current points
    const updatePointsLayer = () => {
      if (!pointsLayer.value) return;

      pointsLayer.value.clearLayers();

      points.value.forEach((point, index) => {
        const color = selectedPoints.value.includes(point.id)
          ? (selectedPoints.value.indexOf(point.id) === 0 ? "#28a745" : "#dc3545") // Green for start, red for end
          : "#3388ff";

        L.circleMarker(point.latlng, {
          radius: 8,
          fillColor: color,
          color: "#000",
          weight: 1,
          opacity: 1,
          fillOpacity: 0.8
        })
        .bindPopup(`<b>${point.properties.name}</b><br>${new Date(point.properties.timestamp).toLocaleString()}`)
        .on('click', () => selectPoint(index)) // Select point when clicked
        .addTo(pointsLayer.value);
      });
    };

    // Function to select departure and arrival points
    const selectPoint = (index) => {
      if (selectedPoints.value.length < 2) {
        selectedPoints.value.push(points.value[index].id);
        console.log(`Selected Point: ${points.value[index].id}`);
      }

      if (selectedPoints.value.length === 2) {
        console.log(`Departure: ${selectedPoints.value[0]}, Arrival: ${selectedPoints.value[1]}`);
        calculateOptimalPath(); // Automatically calculate the path
      }

      updatePointsLayer(); // Update colors based on selection
    };

    // Clear all points
    const clearPoints = () => {
      // Clear all points and reset selected points
      points.value = [];
      selectedPoints.value = [];
      // Clear graph edges
      if (graphEdges.value && graphEdges.value.length > 0) {
        graphEdges.value.forEach(edge => {
          if (map.value) {
            map.value.removeLayer(edge);
          }
        });
        graphEdges.value = [];
      }
      // Clear the optimal path line
      clearPathLine();
      // Clear the points layer
      if (pointsLayer.value) {
        pointsLayer.value.clearLayers();
      }
      // Reset the map view to the default position
      if (map.value) {
        map.value.setView([39.8283, -98.5795], 4); // Default center and zoom level
      }
      // Reset total distance
      totalDistance.value = null;
      console.log("All points, selections, and layers have been cleared.");
    };

    // Clear the selected departure and arrival points
    const clearSelection = () => {
      selectedPoints.value = []; // Reset the selected points
      totalDistance.value = 0;
      clearPathLine(); // Clear the path line if it exists
      console.log("Selection cleared.");
      updatePointsLayer(); // Reset point colors
    };

    // Clear the path line
    const clearPathLine = () => {
      if (pathLine.value && map.value) {
        map.value.removeLayer(pathLine.value); // Remove the polyline from the map
        pathLine.value = null; // Reset the reference
      }
    };

    // Draw the graph edges based on the reduced graph from the backend
    const drawGraphEdges = (reducedGraph) => {
      if (!map.value) return;

      // Clear existing edges
      graphEdges.value.forEach(edge => {
        map.value.removeLayer(edge);
      });
      graphEdges.value = [];

      // Draw edges based on the reduced graph
      reducedGraph.edges.forEach(edge => {
        const sourcePoint = points.value.find(p => p.id === edge.source);
        const targetPoint = points.value.find(p => p.id === edge.target);

        if (sourcePoint && targetPoint) {
          const line = L.polyline(
            [[sourcePoint.latlng.lat, sourcePoint.latlng.lng], [targetPoint.latlng.lat, targetPoint.latlng.lng]],
            { color: 'gray', weight: 1 }
          ).addTo(map.value);

          graphEdges.value.push(line);
        }
      });
    };

    // Fetch the graph from the backend
    const fetchGraph = async () => {
      if (!points.value || points.value.length === 0) {
        alert('Please add points to the map first.');
        return;
      }

      const geoJSON = {
        type: "FeatureCollection",
        features: points.value.map((point) => ({
          type: "Feature",
          properties: {
            ...point.properties,
            id: point.id
          },
          geometry: {
            type: "Point",
            coordinates: [point.latlng.lng, point.latlng.lat]
          }
        }))
      };

      try {
        const response = await fetch('http://localhost:5000/get_graph', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({ geojson: geoJSON })
        });

        if (!response.ok) {
          alert('Failed to fetch the graph.');
          return;
        }

        const result = await response.json();
        console.log("Reduced Graph Received:", result);

        // Pass the reduced graph to the drawGraphEdges function
        drawGraphEdges(result);
      } catch (error) {
        console.error('Error fetching graph:', error);
        alert('An error occurred while fetching the graph.');
      }
    };

    // Calculate the optimal path
    const calculateOptimalPath = async () => {
      const geoJSON = {
        type: "FeatureCollection",
        features: points.value.map((point) => ({
          type: "Feature",
          properties: {
            ...point.properties,
            id: point.id
          },
          geometry: {
            type: "Point",
            coordinates: [point.latlng.lng, point.latlng.lat]
          }
        }))
      };

      const payload = {
        geojson: geoJSON,
        departure: selectedPoints.value[0],
        arrival: selectedPoints.value[1]
      };

      try {
        const response = await fetch('http://localhost:5000/calculate_path', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(payload)
        });

        if (response.ok) {
          const result = await response.json();
          drawPathLine(result); // Draw the blue line for the optimal path
        } else {
          alert('Failed to calculate the optimal path.');
        }
      } catch (error) {
        console.error('Error sending GeoJSON:', error);
        alert('An error occurred while calculating the optimal path.');
      }
    };

    // Draw the optimal path as a blue line
    const drawPathLine = (response) => {
      if (!map.value) return;

      const path = response.path; // Extract the path from the response
      const totalDistanceValue = response.total_distance; // Extract the total distance

      // Extract the coordinates of the points in the path
      const pathCoords = path.map(pointId => {
        const point = points.value.find(p => p.id === pointId);
        return point ? [point.latlng.lat, point.latlng.lng] : null;
      }).filter(coord => coord !== null);

      // Clear any existing path line
      clearPathLine();

      // Draw a new blue polyline
      if (pathCoords.length > 1) {
        pathLine.value = L.polyline(pathCoords, { color: 'blue', weight: 3 }).addTo(map.value);
        map.value.fitBounds(pathLine.value.getBounds());
      }

      // Update the total distance in the UI
      totalDistance.value = `Total Distance: ${totalDistanceValue.toFixed(2)} km`;
    };

    // Watch for changes in selectedPoints to ensure proper updates
    watch(selectedPoints, () => {
      updatePointsLayer();
    });

    return {
      mapContainer,
      clearPoints,
      fetchGraph,
      clearSelection,
      points, // Expose points for debugging
      selectedPoints, // Expose selectedPoints for debugging
      totalDistance // Expose totalDistance for the template
    };
  }
};
</script>

<style>
.map-container {
  margin: 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
  overflow: hidden;
}

.controls {
  padding: 10px;
  background: #f5f5f5;
  border-top: 1px solid #ddd;
  display: flex;
  gap: 10px;
}

.control-icon {
  font-size: 20px;
  cursor: pointer;
  color: #3388ff;
}

.control-icon:hover {
  color: #2a6fcc;
}

.control-icon.disabled {
  color: gray;
  cursor: not-allowed;
}

.selected-points {
  margin-top: 10px;
  padding: 10px;
  background: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.total-distance {
  margin-top: 10px;
  padding: 10px;
  background: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 4px;
  text-align: center;
}
</style>