<template>
  <div id="app">
    <div class="controls">
      <h1>Postcode Midpoint Calculator</h1>

      <div class="postcodes">
        <div class="postcode" v-for="n in postcodes.length" :key="`postcode-${n}`">
          <label :for="`postcode-input-${n}`">Postcode {{ n }}</label>
          <input
            v-model="postcodes[n - 1]"
            @keydown.enter="addInput"
            :ref="`postcode-${n - 1}`"
            :id="`postcode-input-${n}`"
            type="text"
            placeholder="Enter a postcode"
          />
        </div>
      </div>

      <button @click="calculateMidpoint" class="calculate">Calculate Midpoint</button>
    </div>

    <div id="map"></div>
  </div>
</template>

<script>
import L from 'leaflet';
import axios from 'axios';
import secrets from './assets/secrets.json';

export default {
  name: 'App',
  data() {
    return {
      map: null,
      postcodes: [''],
    };
  },
  mounted() {
    this.map = L.map('map').setView([51.2, -0.8], 9);

    L.tileLayer('https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token={accessToken}', {
      id: 'mapbox/streets-v11',
      tileSize: 512,
      zoomOffset: -1,
      accessToken: secrets.mapboxToken,
      attribution:
        'Map data &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
    }).addTo(this.map);
  },
  methods: {
    addInput() {
      this.postcodes.push('');

      this.$nextTick(() => {
        const lastInput = this.$refs[`postcode-${this.postcodes.length - 1}`][0];
        lastInput.focus();
      });
    },

    async calculateMidpoint() {
      const locs = await this.bulkGeocode(this.postcodes);
      let latSum = 0,
        longSum = 0;
      locs.forEach((loc) => {
        L.marker([loc.latitude, loc.longitude]).addTo(this.map).bindPopup(loc.postcode);
        latSum += loc.latitude;
        longSum += loc.longitude;
      });

      const avgLat = latSum / locs.length;
      const avgLong = longSum / locs.length;
      L.marker([avgLat, avgLong]).addTo(this.map).bindPopup('Midpoint!').openPopup();

      this.map.panTo([avgLat, avgLong]).setZoom(10);
    },

    async bulkGeocode(postcodes) {
      const validPostcodes = postcodes.filter((postcode) => !!postcode);

      const { data } = await axios({
        method: 'POST',
        url: 'https://api.postcodes.io/postcodes',
        data: {
          postcodes: validPostcodes,
        },
      });

      return data.result.map((result) => ({
        query: result.query,
        ...result.result,
      }));
    },
  },
};
</script>

<style lang="scss">
* {
  box-sizing: border-box;
  padding: 0;
  margin: 0;
  font-family: Arial, Helvetica, sans-serif;
}

#app {
  height: 100vh;
  display: grid;
  grid-template-columns: 30% 1fr;
}

#map {
  height: 100%;
}

.controls {
  padding: 1.5rem;
  overflow-y: auto;

  .postcodes {
    margin-top: 1rem;

    .postcode {
      margin-bottom: 1rem;

      &:last-child {
        margin-bottom: 0;
      }

      label {
        display: block;
      }

      input {
        width: 100%;
        padding: 0.5rem;
        margin-top: 0.5rem;
        display: block;
        border: 2px solid grey;
        border-radius: 0.25rem;
        font-size: 16px;
      }
    }
  }

  .calculate {
    padding: 0.5rem 1rem;
    margin-top: 1rem;
    border: 2px solid grey;
    border-radius: 0.25rem;
    cursor: pointer;
    font-size: 16px;
  }
}
</style>
