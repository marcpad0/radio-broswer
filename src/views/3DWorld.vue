<template>
    <div ref="container"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import earthTexture from '../assets/8k_earth_daymap.jpg';

export default {
    name: 'ThreeJsScene',
    data() {
        return {
            camera: null,
            renderer: null,
            controls: null,
            earthRadius: 1 // Radius of the Earth model
        };
    },
    mounted() {
        this.init();
        this.animate();

        // Fetch radio station data
        fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&countrycode=IT&hidebroken=true&order=clickcount&reverse=true')
            .then(response => response.json())
            .then(data => {
                const italianRadioStations = data.filter(radio => radio.countrycode === 'IT');
                italianRadioStations.forEach(station => {
                    const longitude = station.geo_long;
                    const latitude = station.geo_lat;
                    this.addMarker(longitude, latitude, 0.01); // Adjust marker size as needed
                });
            })
            .catch(error => {
                console.error('Error fetching radio station data:', error);
            });

        // Listen for window resize event
        window.addEventListener('resize', this.handleWindowResize);
    },
    beforeUnmount() { // Replace beforeDestroy with beforeUnmount
        // Remove the window resize event listener when component is destroyed
        window.removeEventListener('resize', this.handleWindowResize);
    },
    methods: {
        init() {
            // Create a camera
            this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            this.camera.position.z = 5;

            // Create a renderer
            this.renderer = new THREE.WebGLRenderer();
            this.renderer.setSize(window.innerWidth, window.innerHeight);
            this.$refs.container.appendChild(this.renderer.domElement);

            // Add OrbitControls
            this.controls = new OrbitControls(this.camera, this.renderer.domElement);
            this.controls.enableDamping = true;

            // Create a scene
            const scene = new THREE.Scene();

            // Create a more accurate Earth model
            const geometry = new THREE.SphereGeometry(this.earthRadius, 64, 64); // Increase segments for smoother surface
            const texture = new THREE.TextureLoader().load(earthTexture);
            const material = new THREE.MeshPhongMaterial({ map: texture });
            const earth = new THREE.Mesh(geometry, material);
            scene.add(earth);

            // Add ambient light
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            // Add directional light
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(1, 1, 1).normalize();
            scene.add(directionalLight);

            // Set the scene to component data
            this.scene = scene;
        },
        addMarker(longitude, latitude, markerSize = 0.05) {
            // Only proceed if longitude and latitude are not null
            if (longitude !== null && latitude !== null) {
                // Create a marker
                const geometry = new THREE.SphereGeometry(markerSize, 32, 32);
                const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
                const marker = new THREE.Mesh(geometry, material);

                const preciseLatitude = parseFloat(latitude.toFixed(6));
                const preciseLongitude = parseFloat(longitude.toFixed(6));

                // Convert latitude and longitude to radians
                const latRad = preciseLatitude * Math.PI / 180;
                const lonRad = preciseLongitude * Math.PI / 180;

                // Use the Haversine formula to calculate the x, y, and z coordinates
                const x = this.earthRadius * Math.cos(latRad) * Math.cos(lonRad);
                const y = this.earthRadius * Math.cos(latRad) * Math.sin(lonRad);
                const z = this.earthRadius * Math.sin(latRad);

                marker.position.set(x, y, z);

                // Add the marker to the scene
                this.scene.add(marker);
            } else {
                console.error('Longitude or latitude is null');
            }
        },
        animate() {
            requestAnimationFrame(this.animate);

            // Update OrbitControls
            if (this.controls) {
                this.controls.update();
            }

            // Render the scene
            if (this.renderer && this.scene && this.camera) {
                this.renderer.render(this.scene, this.camera);
            }
        },
        handleWindowResize() {
            // Update camera aspect ratio and renderer size on window resize
            this.camera.aspect = window.innerWidth / window.innerHeight;
            this.camera.updateProjectionMatrix();
            this.renderer.setSize(window.innerWidth, window.innerHeight);
        }
    }
};
</script>

