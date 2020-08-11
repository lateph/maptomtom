<template>
  <q-page class="flex column relative">
    <div id="map" style="flex: 1" />
    <div class="absolute-top full-width">
      <div class="q-pl-md q-pt-md q-pr-md q-pb-xs">
        <q-btn align="between" class="full-width text-black" color="white" label="Cari x" icon="search"  @click="mulaiSearch"/>
      </div>
      <q-scroll-area
        horizontal
        style="height: 70px;"
        class="full-width"
        :visible="false"
      >
        <div class="row q-pt-xs no-wrap">
          <q-btn
            v-for="(c, index) in categories" :key="c.id"
            rounded
            :color="category && category.id == c.id ? 'primary' : 'white'"
            :text-color="category && category.id == c.id ? 'white' : 'black'"
            class="text-no-wrap text-capitalize"
            :class="index == 0 ? 'q-ml-md' : index == categories.length-1 ? 'q-ml-xs q-mr-md' : 'q-ml-xs'"
            size="md"
            :label="c.name"
            @click="setCategory(c)"
          />
        </div>
      </q-scroll-area>
    </div>
    <div class="absolute-bottom full-width q-pa-md" v-if="openPoi">
      <q-card v-if="openPoi.type == 'POI'">
        <q-card-section>
          <q-btn
            fab
            color="primary"
            icon="place"
            class="absolute"
            style="top: 0; right: 12px; transform: translateY(-50%);"
          />
        </q-card-section>
        <q-card-section class="q-pt-none">
          <div class="text-subtitle1">
            {{openPoi.poi.brands[0].name}}
          </div>
          <div class="text-caption text-grey">
            {{openPoi.poi.categories[0]}}
          </div>
        </q-card-section>
        <q-separator />
        <q-card-actions>
          <q-btn flat round icon="event" />
          <q-btn flat color="primary" @click="direction">
            direction
          </q-btn>
        </q-card-actions>
      </q-card>
    </div>
    <q-dialog v-model="searchModal" maximized transition-show="slide-up" transition-hide="slide-down">
      <searchDialog v-on:markers="setMarkers" ref="searchDialog"/>
    </q-dialog>
  </q-page>
</template>

<script>
import tt from '@tomtom-international/web-sdk-maps'
import ttServices from '@tomtom-international/web-sdk-services'
const SearchIconCreator = require('src/components/SearchIconCreator')
const mKey = 'ApQAkJWjATsrDgCAip5JqwHsHKLfbl0D'
import { mapMutations, mapState } from 'vuex'
export default {
  name: 'PageIndex',
  components: {
    searchDialog: require('src/pages/dialog/searchDialog').default
  },
  data () {
    return {
      searchModal: false,
      text: '',
      results: [],
      markers: [],
      map: null,
      userMarker: null,
      openPoi: null,
      category: null,
      categories: [
        {
          id: '7311',
          name: 'Gas Station'
        },
        {
          id: '9361023',
          name: 'Grocery Store'
        },
        {
          id: '7314',
          name: 'Hotel/Motel'
        },
        {
          id: '7321',
          name: 'Hospital'
        },
        {
          id: '7315',
          name: 'Restaurant'
        },
        {
          id: '7310',
          name: 'Repair Shop'
        }
      ]
    }
  },
  computed: {
    ...mapState('map', ['lngLat'])
  },
  mounted () {
    // console.log(iconCreator)
    // Raplaces div#map with TomTom map
    this.map = tt.map({
      key: mKey,
      style: 'tomtom://vector/1/basic-main',
      container: 'map'
    })
    ttServices.services.poiCategories({
      key: mKey
    })
      .go()
      .then((rets) => {
        console.log('cat', rets)
      })
    this.map.on('moveend', () => {
      this.loadPOICat()
    })
    window.addEventListener('keyboardDidHide', () => {
      setTimeout(() => {
        this.map.resize()
      }, 200)
    })
    navigator.geolocation.watchPosition((position) => {
      if (this.userMarker == null) {
        var element = document.createElement('div')
        element.className = 'tt-icon-marker-black'
        var innerElement = document.createElement('div')
        innerElement.className = 'marker-inner tt-icon-' + 'ic_map_poi_110-white'
        element.appendChild(innerElement)
        this.userMarker = new tt.Marker({ element: element })
        this.userMarker.setLngLat([
          position.coords.longitude,
          position.coords.latitude
        ])
        this.userMarker.addTo(this.map)
        this.map.jumpTo({ center: this.userMarker.getLngLat(), zoom: 13 })
      }
      this.setLngLat([
        position.coords.longitude,
        position.coords.latitude
      ])
      this.userMarker.setLngLat([
        position.coords.longitude,
        position.coords.latitude
      ])
    }, (e) => {
      console.log('error', e)
    }, { timeout: 30000 })
  },
  methods: {
    ...mapMutations('map', ['setLngLat']),
    async loadPOICat () {
      if (this.category) {
        const data = await ttServices.services.poiSearch({
          key: mKey,
          categorySet: this.category.id,
          boundingBox: this.map.getBounds()
        })
          .go()
        this.setMarkers(data.results, null)
      }
    },
    setCategory (c) {
      this.category = c
      this.loadPOICat()
    },
    mulaiSearch () {
      if (this.lngLat) {
        this.searchModal = true
      }
    },
    clearMarkers () {
      for (var markerId in this.markers) {
        var marker = this.markers[markerId]
        marker.remove()
      }
    },
    direction () {
      const startPos = this.lngLat

      var lon = this.openPoi.position.lng || this.openPoi.position.lon
      const finalPos = [
        lon,
        this.openPoi.position.lat
      ]
      ttServices.services.calculateRoute({
        key: mKey,
        traffic: false,
        locations: startPos + ':' + finalPos
      })
        .go()
        .then((response) => {
          var geojson = response.toGeoJson()
          this.map.addLayer({
            id: 'route',
            type: 'line',
            source: {
              type: 'geojson',
              data: geojson
            },
            paint: {
              'line-color': '#2faaff',
              'line-width': 8
            }
          })
          // var coordinates = geojson.features[0].geometry.coordinates;
          // this.updateRoutesBounds(coordinates);
        })
        .catch(() => {
          console.log('error cuk')
        })
    },
    setMarkers (data, result) {
      console.log('set markers', data)
      // var bounds = new tt.LngLatBounds()
      this.clearMarkers()
      for (let i = 0; i < data.length; i++) {
        const poi = data[i]
        var markerId = poi.id
        var countryCodeISO3 = poi.address.countryCodeISO3 ? ', ' + poi.address.countryCodeISO3 : ''
        var poiOpts = {
          name: poi.poi ? poi.poi.name : undefined,
          address: poi.address.freeformAddress + countryCodeISO3,
          distance: poi.dist,
          classification: poi.poi ? poi.poi.classifications[0].code : undefined,
          position: poi.position,
          entryPoints: poi.entryPoints
        }

        // console.log(iconCreator)
        // console.log(SearchIconCreator)
        // console.log()

        var element = document.createElement('div')
        element.className = 'tt-icon-marker-black'
        var innerElement = document.createElement('div')
        innerElement.className = 'marker-inner ' + new SearchIconCreator('white', poi).getIcon()
        element.appendChild(innerElement)
        element.onclick = () => {
          console.log(poi)
          this.openPoi = poi
        }

        const marker = new tt.Marker({ element: element })
        // var lon = this.poiData.position.lng || this.poiData.position.lon;
        var lon = poiOpts.position.lng || poiOpts.position.lon
        marker.setLngLat([
          lon,
          poiOpts.position.lat
        ])

        marker.addTo(this.map)
        this.markers[markerId] = marker

        if (result && result.id === poi.id) {
          // bounds.extend(marker.getLngLat())
          this.map.jumpTo({ center: marker.getLngLat(), zoom: 14 })
          this.openPoi = poi
        }
      }
      // this.map.fitBounds(bounds, { padding: 50 })
    }
  }
}
</script>
