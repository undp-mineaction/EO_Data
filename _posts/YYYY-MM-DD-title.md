<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Time Slider with Count and Metadata</title>
  <link rel="stylesheet" href="https://js.arcgis.com/4.29/esri/themes/dark/main.css" />
  <script src="https://js.arcgis.com/4.29/"></script>
  <style>
    html, body, #viewDiv {
      margin: 0;
      padding: 0;
      height: 100%;
      width: 100%;
    }

    #sliderPanel {
      position: absolute;
      bottom: 0;
      left: 0;
      width: 100%;
      background: rgba(20, 20, 20, 0.9);
      padding: 20px;
      box-sizing: border-box;
      z-index: 10;
      display: flex;
      flex-direction: column;
      align-items: center;
      margin: 0;
    }

    #yearSlider {
      width: 100%;
      max-width: 100%;
      height: 8px;
      appearance: none;
      background: #444;
      border-radius: 4px;
      outline: none;
      border: none;
    }

    #yearSlider::-webkit-slider-thumb,
    #yearSlider::-moz-range-thumb {
      width: 20px;
      height: 20px;
      border-radius: 50%;
      background: #1e88e5;
      cursor: pointer;
    }

    #yearLabel {
      margin-top: 10px;
      color: white;
      font-size: 14px;
      font-family: sans-serif;
    }

    #metadataBox {
      margin-top: 10px;
      color: #aaa;
      font-size: 13px;
      font-family: sans-serif;
      text-align: center;
    }

    #infoBox {
      position: absolute;
      top: 20px;
      left: 20px;
      background: rgba(255, 255, 255, 0.95);
      padding: 14px 20px;
      border-radius: 10px;
      box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
      z-index: 20;
      min-width: 240px;
      text-align: center;
      font-family: 'Segoe UI', sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
    }

    .info-title {
      font-size: 14px;
      font-weight: 600;
      color: #333;
      margin-bottom: 6px;
      text-transform: uppercase;
      letter-spacing: 0.5px;
    }

    .info-number {
      font-size: 18px;
      font-weight: bold;
      color: #d32f2f;
    }

    .info-line {
      display: flex;
      justify-content: space-between;
      width: 100%;
      margin-top: 10px;
      font-size: 12px;
      color: #333;
    }

    .info-line div {
      text-align: center;
      font-weight: 600;
      width: 22%;
      padding: 5px 0;
    }

    .info-line div span {
      display: block;
      font-size: 16px;
      font-weight: bold;
      color: #1976d2;
    }

    /* Vertical strip styles */
    #dayStripsContainer {
      display: flex;
      justify-content: center;
      margin-top: 10px;
      overflow-x: scroll;
      width: 100%;
      padding: 10px 0;
      position: relative;
    }

    .dayStrip {
      width: 5px;
      height: 40px;
      background-color: #ddd;
      margin: 0 2px;
      border-radius: 3px;
      cursor: pointer;
    }

    .activeDay {
      background-color: #1976d2;
    }

  </style>
</head>
<body>
  <div id="viewDiv"></div>

  <!-- TOP-LEFT INFO BOX -->
  <div id="infoBox">
    <div class="info-title">EO Casualties</div>
    <div class="info-number" id="totalCount">0</div>

    <div class="info-line">
      <div id="boys">
        Boys
        <span>0</span>
      </div>
      <div id="girls">
        Girls
        <span>0</span>
      </div>
      <div id="men">
        Men
        <span>0</span>
      </div>
      <div id="women">
        Women
        <span>0</span>
      </div>
    </div>
  </div>

  <!-- BOTTOM SLIDER PANEL -->
  <div id="sliderPanel">
    <input type="range" id="yearSlider" />
    <div id="yearLabel">Date: Loading...</div>
    <div id="metadataBox">Loading metadata...</div>
    <div id="dayStripsContainer"></div> <!-- Container for vertical strips -->
  </div>

  <script>
    require([
      "esri/Map",
      "esri/views/MapView",
      "esri/layers/FeatureLayer",
      "esri/TimeExtent"
    ], function(Map, MapView, FeatureLayer, TimeExtent) {

      const layer = new FeatureLayer({
        url: "https://sampleserver6.arcgisonline.com/arcgis/rest/services/Hurricanes/MapServer/0",
        outFields: ["*"],
        timeInfo: {
          timeExtent: {
            start: new Date(2000, 0, 1),
            end: new Date(2020, 11, 31)
          },
          timeInterval: 86400000, // Daily interval
          timeIntervalUnits: "milliseconds"
        }
      });

      const map = new Map({
        basemap: "dark-gray-vector",
        layers: [layer]
      });

      const view = new MapView({
        container: "viewDiv",
        map: map,
        center: [31.1656, 48.3794],  // Coordinates for Ukraine
        zoom: 6,  // Zoom level for Ukraine
        ui: { components: [] }  // This removes the zoom in/out buttons and other UI components
      });

      const slider = document.getElementById("yearSlider");
      const yearLabel = document.getElementById("yearLabel");
      const infoBox = document.getElementById("infoBox");
      const metadataBox = document.getElementById("metadataBox");
      const dayStripsContainer = document.getElementById("dayStripsContainer");

      const startDate = new Date(2000, 0, 1);
      const endDate = new Date(2020, 11, 31);
      const timeRange = endDate.getTime() - startDate.getTime();

      slider.min = 0;
      slider.max = timeRange;
      slider.step = 86400000; // Daily step (in milliseconds)
      slider.value = 0;

      // Generate vertical strips for each day
      function createDayStrips() {
        const totalDays = (timeRange / 86400000);
        for (let i = 0; i <= totalDays; i++) {
          const strip = document.createElement("div");
          strip.classList.add("dayStrip");
          strip.dataset.day = i;
          dayStripsContainer.appendChild(strip);
        }
      }

      function updateTimeFilter(timeOffset) {
        const selectedDate = new Date(startDate.getTime() + timeOffset);
        yearLabel.textContent = `Date: ${selectedDate.toDateString()}`;

        layer.timeExtent = new TimeExtent(selectedDate, selectedDate);

        const startISO = selectedDate.toISOString().split("T")[0];
        const whereClause = `timestamp >= DATE '${startISO}' AND timestamp < DATE '${startISO}'`;

        // Update total feature count
        layer.queryFeatureCount({ where: whereClause })
          .then((count) => {
            document.getElementById("totalCount").textContent = count;
          })
          .catch((err) => {
            document.getElementById("totalCount").textContent = "Error";
            console.error(err);
          });

        // Query metadata for gender and age breakdown
        layer.queryFeatures({
          where: whereClause,
          outFields: ["boys", "girls", "men", "women", "other_metadata_field"],
          returnGeometry: false
        }).then((results) => {
          let boys = 0;
          let girls = 0;
          let men = 0;
          let women = 0;

          let otherMetadata = '';

          results.features.forEach(f => {
            boys += f.attributes.boys || 0;
            girls += f.attributes.girls || 0;
            men += f.attributes.men || 0;
            women += f.attributes.women || 0;
            otherMetadata += f.attributes.other_metadata_field || ''; // Adjust field as needed
          });

          document.getElementById("boys").querySelector("span").textContent = boys;
          document.getElementById("girls").querySelector("span").textContent = girls;
          document.getElementById("men").querySelector("span").textContent = men;
          document.getElementById("women").querySelector("span").textContent = women;

          // Update metadata box with additional data
          metadataBox.innerHTML = `<strong>Other Info:</strong><br>${otherMetadata}`;
        }).catch((err) => {
          document.getElementById("metadataBox").textContent = "Metadata error";
          console.error(err);
        });

        // Highlight the active day in the vertical strips
        const strips = document.querySelectorAll(".dayStrip");
        strips.forEach((strip) => {
          strip.classList.remove("activeDay");
          if (parseInt(strip.dataset.day) === timeOffset) {
            strip.classList.add("activeDay");
          }
        });
      }

      slider.addEventListener("input", () => {
        const timeOffset = parseInt(slider.value);
        updateTimeFilter(timeOffset);
      });

      view.when(() => {
        createDayStrips(); // Create the day strips
        updateTimeFilter(parseInt(slider.value)); // Initialize the map with the first day
      });
    });
  </script>
</body>
</html
