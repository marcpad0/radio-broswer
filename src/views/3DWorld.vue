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
            controls: null
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
                    this.addMarker(longitude, latitude, 0.01);
                });
            })
            .catch(error => {
                console.error('Error fetching radio station data:', error);
            });
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

            // Create a sphere (world)
            const geometry = new THREE.SphereGeometry(1, 32, 32);
            const texture = new THREE.TextureLoader().load(earthTexture);
            const material = new THREE.MeshPhongMaterial({ map: texture });
            const earth = new THREE.Mesh(geometry, material);
            scene.add(earth);

            // Add an ambient light
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            // Add a directional light
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(1, 1, 1).normalize();
            scene.add(directionalLight);

            // Set the scene to component data
            this.scene = scene;
        },
        addMarker(longitude, latitude, markerSize = 0.05) {
            // Create a marker
            const geometry = new THREE.SphereGeometry(markerSize, 32, 32);
            const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
            const marker = new THREE.Mesh(geometry, material);

            // Convert latitude and longitude from degrees to radians
            const phi = THREE.Math.degToRad(90 - latitude); // Convert latitude to polar angle
            const theta = THREE.Math.degToRad(longitude + 338); // Adjust longitude to proper range and convert

            // Convert spherical coordinates to Cartesian coordinates for the marker
            marker.position.x = Math.sin(phi) * Math.cos(theta);
            marker.position.y = Math.cos(phi);
            marker.position.z = Math.sin(phi) * Math.sin(theta);

            // Add the marker to the scene
            this.scene.add(marker);
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
    },
};
</script>