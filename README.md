<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>ESP32 Mission Planner - Firebase UI</title>
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
<style>
    html, body {
        height: 100%;
        margin: 0;
        padding: 0;
    }
    body {
        font-family: Arial, sans-serif;
        display: flex;
        flex-direction: column;
    }
    #map {
        flex: 1;               /* Map takes remaining vertical space */
        min-height: 400px;     /* Ensure it's always visible */
    }
    #controls {
        padding: 15px;
        background-color: #f8f8f8;
        border-top: 1px solid #eee;
    }
    textarea {
        width: calc(100% - 10px);
        height: 100px;
        margin-top: 10px;
        padding: 5px;
        border: 1px solid #ccc;
        font-family: monospace;
        resize: vertical;
    }
    button {
        margin: 5px 5px 5px 0;
        padding: 8px 15px;
        background-color: #4CAF50;
        color: white;
        border: none;
        border-radius: 4px;
        cursor: pointer;
    }
    button:hover {
        background-color: #45a049;
    }
    #robot-status-display {
        margin-top: 10px;
        padding: 8px;
        background-color: #e0f2f7;
        border-left: 5px solid #2196F3;
        font-size: 0.9em;
    }
    .status-text { margin-right: 10px; }
    #loginForm { padding: 20px; }
    #dashboard { display: none; flex: 1; flex-direction: column; }
</style>
</head>
<body>
<!-- Login Screen -->
<div id="loginForm">
    <h2>Login</h2>
    <input type="text" id="email" placeholder="Email"><br><br>
    <input type="password" id="password" placeholder="Password"><br><br>
    <button id="loginBtn">Login</button>
</div>

<!-- Dashboard (hidden until login) -->
<div id="dashboard">
    <div id="map"></div>
    <div id="controls">
        <h3>Robot Status:</h3>
        <div id="robot-status-display">Loading status...</div>

        <button id="clearBtn">Clear Map Markers</button>
        <button id="uploadMissionBtn">Upload Mission to Robot</button>
        <button id="fetchMissionBtn">Fetch Mission from Robot</button>
        <button id="refreshConfigBtn">Refresh Config</button>

        <h4>Mission JSON (editable)</h4>
        <textarea id="missionArea" placeholder='{"waypoints":[{"lat":6.52, "lon":3.37}]}'></textarea>
    </div>
</div>

<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>

<script>
    // --- FIREBASE CONFIGURATION ---
    const firebaseConfig = {
      apiKey: "AIzaSyADd3G_8WJFmiJmB2ewOYhs9IpuJRtTQ7A",
      authDomain: "navigator-59e90.firebaseapp.com",
      databaseURL: "https://navigator-59e90-default-rtdb.firebaseio.com",
      projectId: "navigator-59e90",
      storageBucket: "navigator-59e90.appspot.com",
      messagingSenderId: "677281499595",
      appId: "1:677281499595:web:f0bafebed198b2eece4e4e"
    };
    firebase.initializeApp(firebaseConfig);

    // Reference to robot in Firebase
    const robotRef = firebase.database().ref("robot1");

    // --- LOGIN ---
    const allowedEmail = "israelefe093@gmail.com";
    const allowedPassword = "ENG2002551";

    document.getElementById("loginBtn").onclick = function() {
        const email = document.getElementById("email").value;
        const pass = document.getElementById("password").value;
        if (email === allowedEmail && pass === allowedPassword) {
            document.getElementById("loginForm").style.display = "none";
            document.getElementById("dashboard").style.display = "flex";
            initMap(); // Load map after login
        } else {
            alert("Invalid login!");
        }
    };

    // --- MAP + CONTROLS ---
    let map, markerGroup, robotMarker;

    function initMap() {
        map = L.map('map').setView([6.5244, 3.3792], 15); // Lagos default
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 19,
            attribution: '© OpenStreetMap contributors'
        }).addTo(map);

        markerGroup = L.layerGroup().addTo(map);
        robotMarker = null;

        // Test marker
        L.marker([6.5244, 3.3792]).addTo(map).bindPopup("Map Loaded ✅").openPopup();

        setupUI();
        setupFirebaseListeners();
    }

    // Add waypoint marker
    function addWaypointMarker(latlng, draggable = true) {
        let m = L.marker(latlng, { draggable: draggable }).addTo(markerGroup);
        if (draggable) {
            m.on('dragend', updateMissionJsonFromMarkers);
            m.on('click', function() {
                if (confirm('Remove this waypoint marker?')) {
                    markerGroup.removeLayer(m);
                    updateMissionJsonFromMarkers();
                }
            });
        }
        return m;
    }

    // Update mission JSON from markers
    function updateMissionJsonFromMarkers() {
        let wps = [];
        markerGroup.eachLayer(function(layer) {
            if (layer.options.draggable) {
                let p = layer.getLatLng();
                wps.push({ lat: +p.lat.toFixed(7), lon: +p.lng.toFixed(7) });
            }
        });
        let mission = {
            mission_id: 'UI-' + Date.now(),
            waypoints: wps
        };
        document.getElementById('missionArea').value = JSON.stringify(mission, null, 2);
    }

    // Setup UI buttons
    function setupUI() {
        map.on('click', function(e) {
            addWaypointMarker(e.latlng, true);
            updateMissionJsonFromMarkers();
        });

        document.getElementById('clearBtn').onclick = function() {
            markerGroup.eachLayer(function(layer) {
                if (layer.options.draggable) {
                    markerGroup.removeLayer(layer);
                }
            });
            updateMissionJsonFromMarkers();
        };

        document.getElementById('uploadMissionBtn').onclick = function() {
            let txt = document.getElementById('missionArea').value;
            try {
                const mission = JSON.parse(txt);
                robotRef.child('mission').set(mission)
                    .then(() => alert('Mission uploaded successfully!'))
                    .catch(error => alert('Upload failed: ' + error.message));
            } catch (e) {
                alert('Invalid JSON: ' + e.message);
            }
        };

        document.getElementById('fetchMissionBtn').onclick = function() {
            robotRef.child('mission').once('value', (snapshot) => {
                if (!snapshot.exists()) {
                     alert('No mission found.');
                } else {
                    alert('Mission fetched.');
                }
            });
        };

        document.getElementById('refreshConfigBtn').onclick = function() {
            robotRef.child('config').once('value', (snapshot) => {
                const config = snapshot.val();
                if (config) {
                    alert(`Config: BaseSpeed=${config.baseSpeed || 'N/A'}, SafeDistance=${config.safeDistance || 'N/A'}`);
                } else {
                    alert('No config found.');
                }
            }).catch(error => console.error("Error fetching config:", error));
        };
    }

    // Firebase listeners
    function setupFirebaseListeners() {
        robotRef.child('status').on('value', (snapshot) => {
            const status = snapshot.val();
            const statusDisplay = document.getElementById('robot-status-display');
            if (status) {
                statusDisplay.innerHTML = `
                    <span class="status-text">Lat: ${status.lat ? status.lat.toFixed(6) : 'N/A'}</span>
                    <span class="status-text">Lng: ${status.lng ? status.lng.toFixed(6) : 'N/A'}</span>
                    <span class="status-text">Heading: ${status.heading ? status.heading.toFixed(1) + '°' : 'N/A'}</span>
                    <span class="status-text">Battery: ${status.battery ? status.battery + '%' : 'N/A'}</span>
                    <span class="status-text">Ultrasonic: ${status.ultrasonic ? status.ultrasonic.toFixed(1) + 'cm' : 'N/A'}</span>
                    <span class="status-text">Updated: ${status.timestamp ? new Date(status.timestamp).toLocaleTimeString() : 'N/A'}</span>
                `;
                if (status.lat && status.lng) {
                    const robotLatLng = [status.lat, status.lng];
                    if (!robotMarker) {
                        robotMarker = L.marker(robotLatLng, {
                            icon: L.divIcon({
                                className: 'custom-robot-icon',
                                html: '<div style="font-size: 24px; color: blue;">🤖</div>',
                                iconSize: [24, 24],
                                iconAnchor: [12, 12]
                            })
                        }).addTo(map);
                    } else {
                        robotMarker.setLatLng(robotLatLng);
                    }
                    map.panTo(robotLatLng);
                }
            } else {
                statusDisplay.innerText = "Robot status not available.";
                if (robotMarker) {
                    map.removeLayer(robotMarker);
                    robotMarker = null;
                }
            }
        });

        robotRef.child('mission').on('value', (snapshot) => {
            const mission = snapshot.val();
            markerGroup.eachLayer(function(layer) {
                if (layer.options.draggable) {
                     markerGroup.removeLayer(layer);
                }
            });

            if (mission && mission.waypoints) {
                mission.waypoints.forEach(wp => {
                    addWaypointMarker([wp.lat, wp.lon], false);
                });
                if (markerGroup.getLayers().length > 0) {
                    map.fitBounds(markerGroup.getBounds());
                }
                document.getElementById('missionArea').value = JSON.stringify(mission, null, 2);
            } else {
                document.getElementById('missionArea').value = JSON.stringify({ waypoints: [] }, null, 2);
            }
        });
    }
</script>
</body>
</html>
