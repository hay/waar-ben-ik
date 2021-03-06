<template>
  <div class="modal">
    <div class="box">
      <div class="score">
        <div class="distance">
          {{ displayDistance }}
        </div>
        <div>
          <stars :distanceToImage="distanceToImage"></stars>
        </div>
      </div>
      <div class="map" ref="map" />
      <button @click="closeClick">Volgende foto</button>
    </div>
  </div>
</template>

<script>
/* global L */
import Stars from './Stars.vue';
import distance from '@turf/distance'
import { marker, flag } from '../lib/markers'
import { formatDistance } from '../lib/util.js';

export default {
  components : { Stars },
  name: 'Results',
  props: {
    image: Object,
    submittedPoint: Object
  },
  mounted: function () {
    const element = this.$refs.map
    const map = L.map(element)

    // const tileUrl = 'https://{s}.data.amsterdam.nl/topo_wm_light/{z}/{x}/{y}.png'
    const tileUrl = 'https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}.png'

    L.tileLayer(tileUrl, {
      // subdomains: ['t1', 't2', 't3', 't4'],
      maxZoom: 19
    }).addTo(map)

    const resultsLayer =  L.geoJSON(this.geojson, {
      onEachFeature: this.createPopup,
      pointToLayer: function (feature, latLng) {
        if (feature.properties.type === 'submission') {
          return marker(latLng)
        } else {
          return flag(latLng)
        }
      },
      style: function () {
        return {
          color: 'black',
          weight: 4,
          opacity: .7,
          dashArray: '0,10',
          lineJoin: 'round'
        }
      }
    }).addTo(map)

    map.fitBounds(resultsLayer.getBounds(), {
      padding: [20, 20]
    })

    this.resultsLayer = resultsLayer
    this.map = map
  },

  methods: {
    closeClick: function () {
      this.$emit('close', this.distanceToImage);
    },
    mapUrl: function (point) {
      const coordinates = point.coordinates
      return `https://data.amsterdam.nl/data/geozoek/?modus=kaart&legenda=true&locatie=${coordinates[1]}%2C${coordinates[0]}`
    },
    createPopup: function (feature, layer) {
      if (feature.properties.type === 'submission' || feature.properties.panoramaId) {
        const mapUrl = this.mapUrl(feature.geometry)
        const link = `<a href="${mapUrl}" target="_blank">Bekijk deze locatie</a>`
        layer.bindPopup(link)
      }
    }
  },
  computed: {
    distanceToImage: function () {
      return Math.round(distance(this.image.geometry, this.submittedPoint, {
        units: 'meters'
     }))
    },
    displayDistance: function () {
      return formatDistance(this.distanceToImage);
    },
    url: function () {
      return `https://data.amsterdam.nl/data/panorama/${this.image.pano_id}/?modus=volledig`
    },
    geojson: function () {
      return {
        type: 'FeatureCollection',
        features: [
          {
            type: 'Feature',
            properties: {
              panoramaId: this.image.pano_id
            },
            geometry: this.image.geometry
          },
          {
            type: 'Feature',
            properties: {},
            geometry: {
              type: 'LineString',
              coordinates: [
                this.image.geometry.coordinates,
                this.submittedPoint.coordinates
              ]
            }
          },
          {
            type: 'Feature',
            properties: {
              type: 'submission'
            },
            geometry: this.submittedPoint
          }
        ]
      }
    }
  }
}
</script>

<style scoped>
.box {
  width: 900px;
}

.map {
  width: 100%;
  height: 500px;
}

.score {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
}

.score, .map {
  margin-bottom: 12px;
}

.distance {
  font-size: 2em;
  line-height: 1em;
  font-weight: bold;
}

@media only screen and (max-height: 568px) {
  .map {
    height: 400px;
  }
}

</style>
