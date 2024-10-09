<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Dynamic Marathon Map</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <style>
    #map {
      height: 600px;
      width: 100%;
    }
  </style>
</head>
<body>
  <h2>Marathon Map with Dynamic Colored Line</h2>
  
  <div id="map"></div>

  <script>
    // Initialize the map
    var map = L.map('map').setView([29.3759, 47.9774], 13); // Starting at Kuwait coordinates

    // Add the OpenStreetMap tiles
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    let participantData = [];

    // Get the Bib No from the URL
    function getBibFromURL() {
      const params = new URLSearchParams(window.location.search);
      return params.get('bib');
    }

    // Fetch data from the API
    fetch('https://mocki.io/v1/8e7119df-445f-4074-958e-0ffbf7f4dcc8')
      .then(response => response.json())
      .then(data => {
        participantData = data; // Store fetched data
        
        const bibFilter = getBibFromURL(); // Get the Bib No from the URL

        if (bibFilter) {
          const filteredData = participantData.filter(participant => participant.bib_no === bibFilter);
          displayMarkersAndPolylines(filteredData); // Display filtered participants
        } else {
          displayMarkersAndPolylines(participantData); // Display all participants if no Bib No filter
        }
      })
      .catch(error => console.error('Error fetching participant data:', error));

    // Function to display markers and polylines
    function displayMarkersAndPolylines(data) {
      data.forEach(participant => {
        const locations = participant.locations;

        // Store the coordinates for the polyline
        const polylineCoordinates = [];

        // Store markers separately to synchronize with polylines
        const markers = [];

        // Iterate over locations
        locations.forEach((loc, index) => {
          polylineCoordinates.push([loc.lat, loc.lng]); // Add each location to the polyline

          // Check if the `number` field is not empty to display a marker
          if (loc.number) {
            const marker = L.marker([loc.lat, loc.lng])
              .bindPopup(`<b>${participant.name}</b><br> Bib No: ${participant.bib_no}<br> Speed: ${loc.number}`);
            markers.push(marker);
          }
        });

        // Draw polyline connecting all locations and synchronize popups with marker locations
        animatePolylinesSequentially(polylineCoordinates, markers);
      });
    }

    // Function to animate polylines and synchronize marker popups
    function animatePolylinesSequentially(locations, markers) {
      let currentStep = 0;

      function animateSegment(index) {
        if (index >= locations.length - 1) return; // Stop if there's no next point

        const loc1 = locations[index];
        const loc2 = locations[index + 1];

        const isMarkerAtNext = markers.some(marker => marker.getLatLng().lat === loc2[0] && marker.getLatLng().lng === loc2[1]); // Check if there's a marker at the next location

        const polyline = L.polyline([[loc1[0], loc1[1]], [loc1[0], loc1[1]]], { color: 'red', weight: 5 }).addTo(map);

        // Use medium speed for marker vs. non-marker points
        const totalSteps = isMarkerAtNext ? 50 : 15; // More steps for markers, fewer for non-markers
        const intervalDuration = isMarkerAtNext ? 10 : 10; // Medium interval for non-markers

        const interval = setInterval(() => {
          if (currentStep >= totalSteps) {
            clearInterval(interval);
            // Finalize the polyline at the end location
            polyline.setLatLngs([[loc1[0], loc1[1]], [loc2[0], loc2[1]]]);
            currentStep = 0; // Reset step for next segment

            // If there is a marker at this location, show the popup
            if (isMarkerAtNext) {
              const marker = markers.find(marker => marker.getLatLng().lat === loc2[0] && marker.getLatLng().lng === loc2[1]);
              if (marker) {
                marker.addTo(map);
                setTimeout(() => {
                  marker.openPopup(); // Open the marker popup when polyline reaches this point
                }, 300); // Delay to sync popup with polyline animation
              }
            }

            setTimeout(() => animateSegment(index + 1), isMarkerAtNext ? 500 : 100); // Slight delay before next animation
          } else {
            const lat = loc1[0] + (loc2[0] - loc1[0]) * (currentStep / totalSteps);
            const lng = loc1[1] + (loc2[1] - loc1[1]) * (currentStep / totalSteps);
            polyline.setLatLngs([[loc1[0], loc1[1]], [lat, lng]]);
            currentStep++;
          }
        }, intervalDuration); // Adjust speed of animation
      }

      animateSegment(0); // Start the animation from the first segment
    }

  </script>
</body>
</html>

