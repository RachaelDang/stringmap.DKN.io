<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>String Atlas • Global Connection</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
    :root {
        --bg: #03070b;
        --land: #1a3a32;
        --land-hover: #00ffcc;
        --string: #00ffcc;
        --glow: #00ffcc;
        --grid: rgba(0, 255, 204, 0.05);
    }
    
    body {
        margin: 0;
        height: 100vh;
        background: var(--bg);
        background-image: 
            linear-gradient(var(--grid) 1px, transparent 1px),
            linear-gradient(90deg, var(--grid) 1px, transparent 1px);
        background-size: 50px 50px;
        color: #fff;
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        overflow: hidden;
        display: flex;
        flex-direction: column;
    }
    
    header {
        padding: 30px;
        text-align: center;
        z-index: 10;
    }
    
    h1 {
        font-size: 2rem;
        margin: 0;
        letter-spacing: 8px;
        font-weight: 300;
        text-shadow: 0 0 20px var(--glow);
    }
    
    #map-container {
        flex: 1;
        position: relative;
        width: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
    }
    
    svg {
        width: 90%;
        height: 85%;
        filter: drop-shadow(0 0 20px rgba(0,0,0,0.5));
    }

    .country-path {
        fill: var(--land);
        stroke: var(--bg);
        stroke-width: 0.3;
        transition: fill 0.2s;
        cursor: crosshair;
    }

    .country-path:hover {
        fill: #2d5a4d;
    }

    .country-path.active {
        fill: var(--land-hover);
        filter: drop-shadow(0 0 8px var(--glow));
    }

    .string {
        stroke: var(--string);
        stroke-width: 1;
        fill: none;
        pointer-events: none;
        stroke-dasharray: 1000;
        stroke-dashoffset: 1000;
        animation: draw 1.2s cubic-bezier(0.4, 0, 0.2, 1) forwards;
    }

    @keyframes draw {
        to { stroke-dashoffset: 0; }
    }

    .label-group text {
        fill: #fff;
        font-size: 14px;
        font-weight: bold;
        text-transform: uppercase;
        letter-spacing: 2px;
        pointer-events: none;
        paint-order: stroke;
        stroke: #000;
        stroke-width: 3px;
        stroke-linecap: round;
        stroke-linejoin: round;
    }

    .controls {
        position: absolute;
        bottom: 40px;
        left: 50%;
        transform: translateX(-50%);
        z-index: 20;
        text-align: center;
    }

    #search {
        background: rgba(5, 15, 12, 0.9);
        border: 1px solid var(--glow);
        color: white;
        padding: 15px 30px;
        border-radius: 4px;
        width: 350px;
        font-size: 1.1rem;
        outline: none;
        letter-spacing: 2px;
        box-shadow: 0 0 30px rgba(0, 255, 204, 0.1);
        transition: all 0.3s;
    }

    #search:focus {
        box-shadow: 0 0 40px rgba(0, 255, 204, 0.3);
        width: 400px;
    }

    .status {
        margin-top: 15px;
        font-size: 0.7rem;
        color: var(--glow);
        letter-spacing: 3px;
        text-transform: uppercase;
        opacity: 0.6;
    }

    /* Loading overlay */
    #loader {
        position: fixed;
        inset: 0;
        background: var(--bg);
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 100;
        letter-spacing: 5px;
    }
</style>
</head>
<body>

<div id="loader">INITIALIZING ATLAS...</div>

<header>
    <h1>STRING ATLAS</h1>
</header>

<div id="map-container">
    <svg id="worldMap" viewBox="0 0 1000 500" preserveAspectRatio="xMidYMid meet">
        <g id="countries-group"></g>
        <g id="strings-group"></g>
        <g id="labels-group"></g>
    </svg>
</div>

<div class="controls">
    <input type="text" id="search" placeholder="SEARCH COUNTRY..." autocomplete="off">
    <div class="status" id="statusText">AWAITING INPUT</div>
</div>

<script>
const mapConfig = { width: 1000, height: 500 };
const countriesGroup = document.getElementById('countries-group');
const stringsGroup = document.getElementById('strings-group');
const labelsGroup = document.getElementById('labels-group');
const searchInput = document.getElementById('search');
const statusText = document.getElementById('statusText');

let countryData = {};

// Accurate Equirectangular Projection
function project(lon, lat) {
    const x = (lon + 180) * (mapConfig.width / 360);
    const y = (90 - lat) * (mapConfig.height / 180);
    return { x, y };
}

async function init() {
    try {
        // High quality GeoJSON containing all world countries
        const res = await fetch('https://raw.githubusercontent.com/datasets/geo-countries/master/data/countries.geojson');
        const data = await res.json();
        
        renderMap(data.features);
        document.getElementById('loader').style.display = 'none';
    } catch (e) {
        document.getElementById('loader').innerText = "FAILED TO LOAD DATABASE";
    }
}

function renderMap(features) {
    features.forEach(feature => {
        const name = feature.properties.ADMIN || feature.properties.name;
        const id = name.toLowerCase();
        
        const pathData = generateSVGPath(feature.geometry);
        
        const path = document.createElementNS("http://www.w3.org/2000/svg", "path");
        path.setAttribute("d", pathData);
        path.setAttribute("class", "country-path");
        
        countriesGroup.appendChild(path);
        
        // Calculate the "anchor" using the bounding box center for accuracy
        const bbox = path.getBBox();
        const anchor = {
            x: bbox.x + bbox.width / 2,
            y: bbox.y + bbox.height / 2
        };

        const countryObj = { el: path, name, anchor };
        countryData[id] = countryObj;

        path.addEventListener('mouseenter', () => activateCountry(id));
        path.addEventListener('mouseleave', deactivateCountries);
    });
}

// Robust GeoJSON to SVG Path converter
function generateSVGPath(geometry) {
    if (!geometry) return "";
    
    const convertRing = (ring) => {
        return ring.map((coords, i) => {
            const p = project(coords[0], coords[1]);
            return `${i === 0 ? 'M' : 'L'} ${p.x.toFixed(2)} ${p.y.toFixed(2)}`;
        }).join(' ') + ' Z';
    };

    if (geometry.type === "Polygon") {
        return geometry.coordinates.map(convertRing).join(' ');
    } else if (geometry.type === "MultiPolygon") {
        return geometry.coordinates.map(poly => poly.map(convertRing).join(' ')).join(' ');
    }
    return "";
}

function activateCountry(id) {
    deactivateCountries();
    const country = countryData[id];
    if (!country) return;

    country.el.classList.add('active');
    statusText.innerText = `NODE ACTIVE: ${country.name.toUpperCase()}`;

    // Create the "String"
    const stringPath = document.createElementNS("http://www.w3.org/2000/svg", "path");
    const startX = country.anchor.x;
    const startY = country.anchor.y;
    const endX = startX + (Math.random() * 100 - 50);
    const endY = -100; // Aiming off-screen top

    const d = `M ${startX} ${startY} C ${startX} ${startY - 150}, ${endX} ${startY - 200}, ${endX} ${endY}`;
    
    stringPath.setAttribute("d", d);
    stringPath.setAttribute("class", "string");
    stringsGroup.appendChild(stringPath);

    // Create Label
    const text = document.createElementNS("http://www.w3.org/2000/svg", "text");
    text.setAttribute("x", startX);
    text.setAttribute("y", startY - 20);
    text.setAttribute("text-anchor", "middle");
    text.textContent = country.name;
    
    const g = document.createElementNS("http://www.w3.org/2000/svg", "g");
    g.setAttribute("class", "label-group");
    g.appendChild(text);
    labelsGroup.appendChild(g);
}

function deactivateCountries() {
    Object.values(countryData).forEach(c => c.el.classList.remove('active'));
    stringsGroup.innerHTML = '';
    labelsGroup.innerHTML = '';
    statusText.innerText = "AWAITING INPUT";
}

// Search with fuzzy matching
searchInput.addEventListener('input', (e) => {
    const val = e.target.value.toLowerCase().trim();
    if (!val) {
        deactivateCountries();
        return;
    }

    const match = Object.keys(countryData).find(key => key.includes(val));
    if (match) {
        activateCountry(match);
    }
});

init();
</script>
</body>
</html>
