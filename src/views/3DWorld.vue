<template>
    <div ref="container"></div>
    <div class="navbar" v-if="selectedRadio">
        <img :src="selectedRadio.favicon || defaultImage" class="radio-logo" alt="Radio logo">
        <h4>{{ selectedRadio.name }}</h4>
        <button @click="togglePlayPause(selectedRadio)">{{ selectedRadio.playing ? 'Pause' : 'Play' }}</button>
        <div v-if="selectedRadio.playing" class="sound-wave">
            <div class="bar"></div>
            <div class="bar"></div>
            <div class="bar"></div>
            <div class="bar"></div>
        </div>
        <div @click="toggleFavorite(selectedRadio)" class="heart-container">
            <div :class="['heart', { 'liked': selectedRadio.favorite }]">
            </div>
        </div>
    </div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import earthTexture from '../assets/8k_earth_daymap.jpg';
import Hls from 'hls.js';
import defaultImage from '../assets/images (2).jpeg';

export default {
    name: 'ThreeJsScene',
    data() {
        return {
            camera: null,
            renderer: null,
            controls: null,
            earthRadius: 1,
            radios: [],
            selectedRadio: null,
            defaultImage,
        };
    },
    mounted() {
        this.init();
        this.animate();
        this.getRadios();
        window.addEventListener('resize', this.handleWindowResize);
        this.raycaster = new THREE.Raycaster();
        this.mouse = new THREE.Vector2();
        window.addEventListener('click', this.onDocumentMouseClick);

    },
    beforeUnmount() {
        window.removeEventListener('resize', this.handleWindowResize);
        window.removeEventListener('click', this.onDocumentMouseClick);

    },
    methods: {
        init() {
            this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            this.camera.position.z = 5;

            this.renderer = new THREE.WebGLRenderer();
            this.renderer.setSize(window.innerWidth, window.innerHeight);
            this.$refs.container.appendChild(this.renderer.domElement);

            this.controls = new OrbitControls(this.camera, this.renderer.domElement);
            this.controls.enableDamping = true;

            const scene = new THREE.Scene();

            const geometry = new THREE.SphereGeometry(this.earthRadius, 64, 64); // Increase segments for smoother surface
            const texture = new THREE.TextureLoader().load(earthTexture);
            const material = new THREE.MeshPhongMaterial({ map: texture });
            const earth = new THREE.Mesh(geometry, material);
            scene.add(earth);

            const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(1, 1, 1).normalize();
            scene.add(directionalLight);

            this.scene = scene;
        },
        onDocumentMouseClick(event) {
            event.preventDefault();
            this.mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
            this.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
            this.raycaster.setFromCamera(this.mouse, this.camera);
            const intersects = this.raycaster.intersectObjects(this.scene.children);

            if (intersects.length > 0) {
                console.log('Clicked on:', intersects[0].object.userData);
                this.handleMarkerClick(intersects[0]);
            }
        },
        handleMarkerClick(intersectedObject) {
            if (intersectedObject.object.onClick) {
                intersectedObject.object.onClick();
                const previousRadio = this.selectedRadio;
                this.selectedRadio = intersectedObject.object.userData;
                this.$nextTick(() => {
                    if (this.selectedRadio.url_resolved || this.selectedRadio.url) {
                        this.lazyLoadAudio(this.selectedRadio);
                        if (previousRadio && previousRadio !== this.selectedRadio) {
                            this.pauseRadio(previousRadio);
                        }
                    }
                });
            }
        },

        lazyLoadAudio(radio) {
            console.log('Lazy loading audio for:', radio);
            if (!radio.audioPlayer) {
                console.log('Creating new audio player');
                radio.audioPlayer = new Audio(radio.url_resolved || radio.url);
                console.log('Audio player:', radio.audioPlayer);
                radio.audioPlayer.onloadeddata = () => {
                    console.log('Audio loaded');
                };
                radio.audioPlayer.onerror = () => {
                    console.error('Error loading audio');
                };
            }
            else {
                console.log('Audio player already exists');
            }
        },
        addMarker(longitude, latitude, radio) {
            if (longitude !== null && latitude !== null) {
                const phi = (90 - latitude) * (Math.PI / 180);
                const theta = (longitude + 180) * (Math.PI / 180);
                const x = -this.earthRadius * Math.sin(phi) * Math.cos(theta);
                const y = this.earthRadius * Math.cos(phi);
                const z = this.earthRadius * Math.sin(phi) * Math.sin(theta);

                const geometry = new THREE.SphereGeometry(0.01, 32, 32);
                const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
                const marker = new THREE.Mesh(geometry, material);
                marker.position.set(x, y, z);
                marker.userData = radio;
                marker.onClick = () => {
                    console.log('Clicked on marker:', marker.userData);
                };
                this.scene.add(marker);

                const invisibleSphereGeometry = new THREE.SphereGeometry(0.4, 32, 32);
                const invisibleSphereMaterial = new THREE.MeshBasicMaterial({ visible: false });
                const invisibleSphere = new THREE.Mesh(invisibleSphereGeometry, invisibleSphereMaterial);
                invisibleSphere.position.set(x, y, z);
                invisibleSphere.userData = radio;
                invisibleSphere.onClick = marker.onClick;
                this.scene.add(invisibleSphere);
            } else {
                console.error('Longitude or latitude is null');
            }
        },
        animate() {
            requestAnimationFrame(this.animate);

            if (this.controls) {
                this.controls.update();
            }

            if (this.renderer && this.scene && this.camera) {
                this.renderer.render(this.scene, this.camera);
            }
        },
        handleWindowResize() {
            this.camera.aspect = window.innerWidth / window.innerHeight;
            this.camera.updateProjectionMatrix();
            this.renderer.setSize(window.innerWidth, window.innerHeight);
        },
        async fetchRadios() {
            const response = await fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&countrycode=IT&hidebroken=true&order=clickcount&reverse=true');
            const data = await response.json();
            return data;
        },
        async getRadios() {
            try {
                this.radios = await this.fetchRadios();
                this.radios.forEach(radio => {
                    this.addMarker(radio.geo_long, radio.geo_lat, radio);
                    radio.playing = false;
                    radio.audioPlayer = new Audio();
                });
                this.retrieveFavorites();
            } catch (error) {
                console.error('Error fetching radios:', error);
            }
        },
        togglePlayPause(radio) {
            if (radio.playing) {
                this.pauseRadio(radio);
            } else {
                this.pauseAllRadios(); // Pause all other radios
                this.playRadio(radio);
            }
        },
        playRadio(radio) {
            console.log('playRadio called', radio);
            const audioUrl = radio.url_resolved || radio.url;
            console.log('Audio URL:', audioUrl);
            if (audioUrl.includes('m3u8')) {
                if (Hls.isSupported()) {
                    console.log('HLS is supported');
                    const hls = new Hls();
                    hls.loadSource(audioUrl);
                    hls.attachMedia(radio.audioPlayer);
                } else {
                    console.error('HLS is not supported in this browser.');
                }
            } else {
                console.log('Setting audio source:', audioUrl);
                radio.audioPlayer.src = audioUrl;
            }
            radio.audioPlayer.play()
                .then(() => {
                    console.log('Audio started playing');
                    radio.playing = true;
                })
                .catch(error => {
                    console.error('Error playing audio:', error);
                    if (error.name === 'NotAllowedError') {
                        console.error('Please ensure that the audio playback is allowed in your browser settings.');
                    }
                });
        },
        pauseRadio(radio) {
            radio.audioPlayer.pause();
            radio.playing = false;
        },
        pauseAllRadios() {
            this.radios.forEach(radio => {
                if (radio.playing) {
                    this.pauseRadio(radio);
                }
            });
        },
        toggleFavorite(radio) {
            radio.favorite = !radio.favorite;
            if (!radio.favorite) {
                this.removeFavorite(radio);
            } else {
                this.saveFavorites();
            }
        },
        saveFavorites() {
            const favorites = this.radios.map(radio => ({ changeuuid: radio.changeuuid, favorite: radio.favorite }));
            localStorage.setItem('favorites', JSON.stringify(favorites));
        },
        removeFavorite(radio) {
            let favorites = JSON.parse(localStorage.getItem('favorites')) || [];
            favorites = favorites.filter(fav => fav.changeuuid !== radio.changeuuid);
            localStorage.setItem('favorites', JSON.stringify(favorites));
        },
        retrieveFavorites() {
            const favorites = JSON.parse(localStorage.getItem('favorites')) || [];
            this.radios.forEach(radio => {
                const fav = favorites.find(f => f.changeuuid === radio.changeuuid);
                radio.favorite = fav ? fav.favorite : false;
            });
        },
        created() {
            this.getRadios();
            this.retrieveFavorites();
        }
    }
};
</script>

<style scoped>
.navbar {
    position: fixed;
    bottom: 0;
    left: 0;
    width: 100%;
    background-color: #fff;
    box-shadow: 0 -2px 5px rgba(0, 0, 0, 0.3);
    display: flex;
    align-items: center;
    justify-content: space-around;
    padding: 10px;
}

.radio-logo {
    width: 50px;
    height: 50px;
    border-radius: 25px;
}

.sound-wave {
    display: flex;
    align-items: center;
    height: 20px;
    margin-left: 10px;
    margin-top: 2px;
}

.bar {
    width: 4px;
    height: 100%;
    margin: 0 2px;
    background-color: #333;
    animation: pulse 0.8s infinite ease-in-out alternate;
}

.bar:nth-child(1) {
    animation-delay: 0s;
}

.bar:nth-child(2) {
    animation-delay: 0.1s;
}

.bar:nth-child(3) {
    animation-delay: 0.2s;
}

.bar:nth-child(4) {
    animation-delay: 0.3s;
}

@keyframes pulse {
    0% {
        transform: scaleY(1);
    }

    100% {
        transform: scaleY(1.5);
    }
}

.heart-container {
    display: inline-block;
    cursor: pointer;
    margin-left: 5px;
    margin-top: 2px;
}

.heart {
    width: 20px;
    height: 18px;
    background: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z" fill="%23C1C1C1"/></svg>') center no-repeat;
    background-size: 100%;
}

.heart.liked {
    background-image: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M16.5 3c-1.74 0-3.41.81-4.5 2.09C10.91 3.81 9.24 3 7.5 3 4.42 3 2 5.42 2 8.5c0 3.78 3.4 6.86 8.55 11.54L12 21.35l1.45-1.32C18.6 15.36 22 12.28 22 8.5 22 5.42 19.58 3 16.5 3z" fill="%23FF0000"/></svg>');
}
</style>
