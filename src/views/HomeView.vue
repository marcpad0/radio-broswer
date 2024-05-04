<template>
  <v-container>
    <h1>Homepage</h1>
    <v-row>
      <v-col cols="12" sm="6" md="4" lg="3" v-for="radio in radios" :key="radio.id">
        <v-card class="d-flex flex-row card" flat tile @mouseenter="radio.showControls = true"
          @mouseleave="radio.showControls = false">
          <v-img :src="radio.favicon || '../assets/images (2).jpeg'" class="card-image" :alt="radio.name" />
          <v-card-title class="flex-grow-1">{{ radio.name }}</v-card-title>
          <div v-if="radio.showControls" class="controls">
            <v-btn @click="togglePlayPause(radio)" :color="radio.playing ? 'error' : 'primary'" small>
              {{ radio.playing ? 'Pause' : 'Play' }}
            </v-btn>
            <div v-if="radio.playing" class="sound-wave">
              <div class="bar"></div>
              <div class="bar"></div>
              <div class="bar"></div>
              <div class="bar"></div>
            </div>
          </div>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import Hls from 'hls.js';

export default {
  name: 'HomeView',
  data() {
    return {
      radios: [],
    }
  },
  methods: {
    async fetchRadios() {
      const response = await fetch('https://nl1.api.radio-browser.info/json/stations/search?limit=100&countrycode=IT&hidebroken=true&order=clickcount&reverse=true');
      const data = await response.json();
      return data.filter(radio => radio.countrycode === 'IT');
    },
    async getRadios() {
      try {
        this.radios = await this.fetchRadios();
        this.radios.forEach(radio => {
          radio.showControls = false;
          radio.playing = false;
          radio.audioPlayer = new Audio();
        });
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
      const audioUrl = radio.url_resolved || radio.url;
      if (audioUrl.includes('m3u8')) {
        if (Hls.isSupported()) {
          const hls = new Hls();
          hls.loadSource(audioUrl);
          hls.attachMedia(radio.audioPlayer);
        } else {
          console.error('HLS is not supported in this browser.');
        }
      } else {
        radio.audioPlayer.src = audioUrl;
      }
      radio.audioPlayer.play()
        .catch(error => {
          console.error('Error playing audio:', error);
          if (error.name === 'NotAllowedError') {
            console.error('Please ensure that the audio playback is allowed in your browser settings.');
          }
        });
      radio.playing = true;
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
  },
  created() {
    this.getRadios();
  },
}
</script>

<style scoped>
.v-container {
  background-color: #f5f5f5;
  border-radius: 15px;
  padding: 20px;
}

h1 {
  color: #333;
  text-align: center;
  font-size: 2em;
  margin-bottom: 20px;
}

.v-row {
  justify-content: space-around;
}

.card {
  max-width: 400px;
  box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
  transition: transform 0.3s;
  border-radius: 5px;
  position: relative;
}

.card:hover {
  transform: scale(1.05);
  box-shadow: 0 8px 16px 0 rgba(0, 0, 0, 0.2);
}

.card-image {
  width: 100px;
  height: 100px;
  object-fit: cover;
  border-radius: 5px;
  display: none;
}

.card:hover .card-image {
  display: block;
}

.v-card-title {
  color: #333;
  font-size: 1.2em;
  font-weight: bold;
}

.controls {
  position: absolute;
  bottom: 10px;
  left: 160px;
  /* Changed from 'right' to 'left' */
  display: flex;
  align-items: center;
}

.controls .v-btn {
  margin-right: 50px;
  /* Changed from 'margin-left' to 'margin-right' */
  margin-top: 100px;
}

.sound-wave {
  display: flex;
  align-items: center;
  height: 20px;
  margin-left: 10px; /* Adjust this if necessary */
  margin-top: 100px; /* Added to push the sound wave down */
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
    transform: scaleY(0.1);
  }

  100% {
    transform: scaleY(1);
  }
}
</style>