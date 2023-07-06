<template>
  <div id="app">
    <h1>Tunetones</h1>
    <div v-if="copy" class="notification" :style="{ 'background-color': copy, color: getContrastColor(copy) }">
      Color {{ copy }} copied to clipboard!
    </div>
    <div class="spinner" v-if="isLoading"></div>

    <div class="radio-group">
      <input type="radio" id="song" value="song" v-model="entity" class="radio-button">
      <label for="song" class="radio-label">Song</label>
      <input type="radio" id="album" value="album" v-model="entity" class="radio-button">
      <label for="album" class="radio-label">Album</label>
    </div>

      <div class="search-wrapper">
        <input v-model="searchTerm" @keyup.enter="fetchData"
          class="search-input" />
        <button @click="fetchData" class="search-button">Search</button>
      </div>
    <div class="artworks-container">
      <div v-for="item in artworks" :key="item.url" class="artwork-item">
        <template v-if="item.dummy === false">
          <img :src="item.url" :alt="item.title" :width="item.width" :height="item.height" />
          <template v-if="item.title != item.album">
            <h2>{{ item.title }}</h2>
            <h3>{{ item.artist }}</h3>
          </template>
          <template v-else>
            <h2>{{ item.album }}</h2>
            <h3>{{ item.artist }}</h3>
          </template>
          <div class="color-row">
            <div v-for="color in item.colors" :key="`${color} ${item.title}`" class="color-box"
              :style="{ backgroundColor: color }" @click="copyToClipboard(color)" @mouseover="hover = color"
              @mouseout="hover = ''"></div>
          </div>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import axios from 'axios'
import { extractColors } from 'extract-colors'

let searchTerm = ref('')
let artworks = ref([])
let entity = ref('song')
let isLoading = ref(false)
let copy = ref(null)

const copyToClipboard = (color) => {
  navigator.clipboard.writeText(color).then(() => {
    copy.value = color;
    setTimeout(() => { copy.value = null; }, 2000);
    console.log(copy.value);
    console.log(`${color} copied to clipboard`);
  }, (err) => {
    console.error('Could not copy text: ', err);
  })
}

const getContrastColor = (color) => {
  if (!color) return;

  color = color.substring(1);  // remove #
  const rgb = parseInt(color, 16);   // convert rrggbb to decimal
  const r = (rgb >> 16) & 0xff;  // extract red
  const g = (rgb >> 8) & 0xff;  // extract green
  const b = (rgb >> 0) & 0xff;  // extract blue

  const luma = 0.2126 * r + 0.7152 * g + 0.0722 * b; // per ITU-R BT.709

  return luma < 128 ? 'white' : 'black';
}



const fetchData = async () => {
  isLoading.value = true
  try {
    artworks.value = []  // This line will clear the old results
    const response = await axios.get(`https://itunes.apple.com/search?term=${encodeURIComponent(searchTerm.value)}&entity=${entity.value}&limit=25`)
    artworks.value = await Promise.all(response.data.results.map(async item => {
      const imageUrl = item.artworkUrl100.replace('100x100', '600x600')
      let colors = [];
      try {
        const options = {
          pixels: 64000,
          distance: 0.22,
          colorValidator: (red, green, blue, alpha = 255) => alpha > 250,
          saturationDistance: 0.2,
          lightnessDistance: 0.2,
          hueDistance: 0.083333333,
          crossOrigin: 'Anonymous' // use this to avoid client-side CORS
        }
        colors = await extractColors(imageUrl, options)
        colors = colors.map(color => color.hex)
      } catch (err) {
        console.error(err)
      }
      return {
        url: imageUrl,
        title: entity.value === 'album' ? item.collectionName : item.trackName,
        album: item.collectionName,
        artist: item.artistName,
        width: 600,
        height: 600,
        dummy: false,
        colors
      }
    }))
    while (artworks.value.length % 6 != 0) {
      artworks.value.push({ dummy: true });
    }

  } catch (err) {
    console.error(err)
  } finally {
    isLoading.value = false
  }
}

</script>


<style scoped>
.notification {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  padding: 10px;
  background-color: #28a745;
  /* Bootstrap's .bg-success green color */
  color: #ffffff;
  /* white color for text */
  border-color: #1e7e34;
  /* darker green for border */
  border-radius: .25rem;
  z-index: 1000;
  font-family: 'Gotham-Bold';
  box-shadow: rgba(0, 0, 0, 0.25) 0px 54px 55px, rgba(0, 0, 0, 0.12) 0px -12px 30px, rgba(0, 0, 0, 0.12) 0px 4px 6px, rgba(0, 0, 0, 0.17) 0px 12px 13px, rgba(0, 0, 0, 0.09) 0px -3px 5px;
}

.dummy-item {
  visibility: hidden !important;
}

.spinner {
  position: fixed;
  top: 50%;
  left: calc(50% - 55px);
  transform: translate(-50%, -50%);
  border: 6px solid #f3f3f3;
  border-radius: 50%;
  border-top: 6px solid #3498db;
  width: 100px;
  height: 100px;
  animation: spin 2s linear infinite;
  z-index: 1000;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}

@font-face {
  font-family: 'Gotham-Black';
  src: url('./Gotham-Black.otf') format('opentype');
}

@font-face {
  font-family: 'Gotham-Bold';
  src: url('./Gotham-Bold.otf') format('opentype');
}

@font-face {
  font-family: 'Gotham-Light';
  src: url('./Gotham-Light.otf') format('opentype');
}

h1 {
  font-family: 'Gotham-Black';
  font-size: 3em;
  margin-top: 0;
  margin-bottom: 1rem;
}

h2 {
  font-family: 'Gotham-Bold';
  font-size: 1.3em;
  margin: 0;
}

h3 {
  font-family: 'Gotham-Light';
  font-size: 1.1em;
  margin: 0;
}

img {
  max-width: 75%;
  max-height: 75%;
  height: auto;
  width: auto;
  object-fit: cover;
  border-radius: 2%;
  box-shadow: rgba(0, 0, 0, 0.25) 0px 54px 55px, rgba(0, 0, 0, 0.12) 0px -12px 30px, rgba(0, 0, 0, 0.12) 0px 4px 6px, rgba(0, 0, 0, 0.17) 0px 12px 13px, rgba(0, 0, 0, 0.09) 0px -3px 5px;
}

.search-wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
}

.artworks-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
}

.artwork-item {

  display: grid;
  grid-template-rows: 1fr auto;
  align-items: start;
  /* Align the start of the items to the start of the grid area */
  gap: 1rem;
  /* Add some spacing between the items */
  flex: 1 0 calc(100% - 4rem);
  /* Full width minus some margin on small screens */
  margin: 1rem;
  /* This will add 1rem of margin to all sides */
  background: #fff;
  /* Add a white background to the card */
  padding: 1rem;
  /* Add some padding to the card */
  border-radius: 1rem;
  /* Add some border radius to the card */
}

@media (min-width: 600px) {
  .artwork-item {
    flex: 1 0 calc(50% - 4rem);
    /* Half width on medium screens */
  }
}

@media (min-width: 900px) {
  .artwork-item {
    flex: 1 0 calc(33.33% - 4rem);
    /* One third width on large screens */
  }
}

.color-row {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
}

.color-box {
  width: 50px;
  height: 50px;
  margin: 5px;
  border-radius: 10%;
  box-shadow: rgba(0, 0, 0, 0.24) 0px 3px 8px;
}

#app {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
}

.search-bar {
  display: flex;
  text-align: center;
  margin-bottom: 2rem;
  font-family: 'Gotham-Light';
}

.artwork-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.color-box {
  margin-top: 1rem;
  transition: transform 0.2s;
}

.color-box:hover {
  transform: scale(1.1);
  cursor: pointer;
}

.color-box:active {
  transform: scale(1);
}

.radio-button {
  display: none;
  /* Hide the default radio buttons */
}

.radio-label {
  padding: 10px;
  /* Add some padding to make the labels look like buttons */
  /* Add a border */
  cursor: pointer;
  /* Change cursor to pointer when hovering over the labels */
  transition: background-color 0.2s ease;
  /* Smooth transition for background color change */
}

.radio-group {
  border-radius: 5px;
  border-style: solid;
  border-width: 1.5px;
  overflow: hidden;
  display: inline-flex;
  font-family: 'Gotham-Bold';
  margin-bottom: 1rem;
}

.radio-button:checked+.radio-label {

  background-color: #3498db;
  /* Change the background color when the radio button is checked */
  color: white;
  /* Change the text color when the radio button is checked */
}


input {
  font-family: "Gotham-Light";
}

.radio-group {
  display: inline-flex;
  /* Make the elements inside the group align in one line */
}

.search-input {
  height: 2rem;
  /* Adjust the height of the search box to match the height of the radio buttons */
  margin-right: 10px;
  border-style: solid;
  border-width: 1.5px;
  border-radius: 5px;
  border-color: black;
  /* Add some spacing between the search box and the search button */
}

.search-button {
  height: 2rem;
  /* Adjust the height of the search button to match the height of the search box */
  background-color: #3498db;
  /* Give it a blue color */
  color: white;
  /* Make the text white */
  border: none;
  /* Remove the default border */
  border-radius: 5px;
  /* Curve the border of the button */
  cursor: pointer;
  /* Change the cursor to pointer when hovering over the button */
  transition: background-color 0.2s ease;
  /* Smooth transition for background color change */
  font-family: 'Gotham-Bold';
}

.search-button:hover {
  background-color: #2980b9;
  /* Darken the background color when hovering over the button */
}
</style>
