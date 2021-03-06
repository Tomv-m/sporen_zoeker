<template>
  <div class="route-page">
    <Headful
      :title="`${siteName} | ${route ? route.name : 'Kaart'}`"
      :image="coverImage"
    />
    <header class="main-header" :class="{ 'oranjenassau': isOranjenassau }">
      <div class="wrapper">
        <button 
          class="right-content route-page-button route-page-info"
          @click="attributionter = true"
        >
          i
        </button>
      </div>
    </header>
    <div class="route-page-bottom">
      <div class="wrapper" v-if="route">
        <div class="route-page-actions">
          <router-link
            :to="homeRoute"
            class="route-page-button"
          >
            {{ route ? 'Terug naar routes' : 'Naar Home pagina' }}
          </router-link>
          <button
            v-if="showUserLocationBtn"
            class="route-page-button"
            @click="locateUser"
          >
            Mijn locatie
          </button>
        </div>
        <div class="route-item route-item-small" :title="route.name">
          <div
            class="route-item-image"
            :style="{ 'background-image': 'url(' + coverImage + ')' }"
          />
          <div class="route-item-info">
            <div class="route-item-info-top">
              <h2>{{ route.name }}</h2>
            </div>
            <div class="route-item-info-bottom">
              <div class="content">
                <span>{{ parseFloat(route.distance).toFixed(2) }} km</span>
                <div class="divider"></div>
                <span>{{ visualType }}</span>
              </div>
              <div class="content-icon">
                <FietsIcon v-if="route.type === 'fietsen'" />
                <LoopIcon v-if="route.type === 'lopen'" />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <transition name="fade">
      <Modal v-if="selectedLocation" @close="selectedLocation = null">
        <div class="modal-info">
          <header>
            <h2>{{ selectedLocation.name }}</h2>
            <span>{{ selectedLocation.category }}</span>
          </header>
          <ul class="contact">
            <li>
              <span class="contact-info">Adres </span>{{ selectedLocation.address }}
            </li>
            <li v-if="selectedLocation.phone">
              <span class="contact-info">Telefoon </span>{{ selectedLocation.phone }}
            </li>
            <li v-if="selectedLocation.site">
              <a :href="url(selectedLocation.site)" target="_blank">Meer info</a>
            </li>
          </ul>
        </div>
      </Modal>
    </transition>
    <transition name="fade">
      <Modal v-if="attributionter" @close="attributionter = false">
        <ul class="map-footer">
          <li><a target="_blank" href="https://toerismedebaronie.nl/">&copy; Toerisme De Baronie</a></li>
          <li><a target="_blank" href="https://www.mapbox.com/about/maps/">&copy; Mapbox</a></li>
          <li><a target="_blank" href="http://www.openstreetmap.org/about/">&copy; OpenStreetMap</a></li>
          <li><a target="_blank" href="https://www.mapbox.com/map-feedback/"><b>Improve this map</b></a></li>
        </ul>
      </Modal>
    </transition>
    <div class="route-page-loader" v-if="loading">
      <Loader />
    </div>
    <div id="map"></div>
  </div>
</template>

<script>
import firebase from 'firebase/app'
import mapbox from 'mapbox-gl'
import { ScaleOut as Loader } from 'vue-loading-spinner'

import { isOranjenassau, siteName, homeRoute } from '../../global'

import FietsIcon from '@/components/icons/FietsIcon'
import LoopIcon from '@/components/icons/LoopIcon'
import Modal from '@/components/Modal'
// Icons
import ster from '@/assets/images/icons/ster.png';
import bestek from '@/assets/images/icons/bestek.png';
import klaver from '@/assets/images/icons/klaver.png';
import bed from '@/assets/images/icons/bed.png';
import tent from '@/assets/images/icons/tent.png';
// Key Locations
import beekseBergen from '@/assets/images/icons/beekse-bergen.png';
import efteling from '@/assets/images/icons/efteling.png';

export default {
  name: 'Route',
  components: {
    Loader,
    FietsIcon,
    LoopIcon,
    Modal
  },
  props: {
    route: Object,
    prevRoute: Object
  },
  computed: {
    visualType() {
      const type = this.route.type
      if (type === 'lopen') {
        return 'Wandelen'
      } else if (type === 'fietsen') {
        return 'Fietsen'
      } else {
        return ''
      }
    }
  },
  data() {
    return {
      loading: false,
      map: null,
      mapStyle: 'mapbox://styles/v-mtom/ckddgaaie42tu1jmwj7cbbv1f',
      attributionter: false,
      selectedLocation: null,
      userPosition: null,
      userMarker: null,
      showUserLocationBtn: false,
      coverImage: '',
      iconImages: [
        { img: ster, name: 'ster' },
        { img: bestek, name: 'bestek' },
        { img: klaver, name: 'klaver' },
        { img: bed, name: 'bed' },
        { img: tent, name: 'tent' },
      ],
      keyLocations: [
        { img: efteling, coordinates: [5.043636, 51.649820]},
        { img: beekseBergen, coordinates: [5.114222, 51.525554] }
      ], 
      siteName,
      homeRoute,
      isOranjenassau
    }
  },
  methods: {
    url(url) {
      if (!url.includes('http')) {
        url = 'http://'+url+'/'
      }
      return url
    },
    setRoute(coordinates) {
      coordinates = coordinates.map(item => [item.lng, item.lat])
      this.map.addLayer({
        "id": "route",
        "type": "line",
        "source": {
          "type": "geojson",
          "data": {
            "type": "Feature",
            "properties": {},
            "geometry": {
              "type": "LineString",
              "coordinates": coordinates
            }
          }
        },
        "layout": {
          "line-join": "round",
          "line-cap": "round"
        },
        "paint": {
          "line-color": "#008D36",
          "line-width": 6
        }
      })
      // zoom to route
      const bounds = coordinates.reduce((bounds, coord) => {
        return bounds.extend(coord)
      }, new mapbox.LngLatBounds(coordinates[0], coordinates[0]))
      this.map.fitBounds(bounds, { padding: 40 })
    },
    setRoutePoints(routePoints) {
      routePoints.forEach(point => {
        let el = document.createElement('div');
        el.className = isOranjenassau ? 'route-point hex' : 'route-point'
        el.innerHTML = point.name
        new mapbox.Marker(el)
          .setLngLat([point.lng, point.lat])
          .addTo(this.map)
      })
    },
    locateUser() {
      this.map.flyTo({
        center: this.userPosition,
        zoom: 14,
        essential: true
      })
    },
    showUserPoint() {
      if (this.userMarker === null) {
        let el = document.createElement('div')
        el.className = 'user-point'
        this.userMarker = new mapbox.Marker(el)
          .setLngLat(this.userPosition)
          .addTo(this.map)
      } else {
        this.userMarker.setLngLat(this.userPosition)
      }
    },
    setLocations(locations) {
      this.map.addSource('locations', {
        type: 'geojson',
        data: this.convertLocationsToGeojson(locations)
      })
      
      this.iconImages.forEach((icon, index) => {
        this.map.loadImage(icon.img, (err, image) => {
          if (err) throw err
          this.map.addImage(icon.name, image)
          if (index === this.iconImages.length -1) {
            this.map.addLayer({
              id: 'icons',
              type: 'symbol',
              source: 'locations',
              layout: {
                'icon-image': '{icon}',
                'icon-size': 0.25,
                'icon-allow-overlap': false
              }
            })
          }
        })
      })

      this.map.on('mouseenter', 'icons', () => {
        this.map.getCanvas().style.cursor = 'pointer';
      })
      this.map.on('mouseleave', 'icons', () => {
        this.map.getCanvas().style.cursor = ''
      })
      this.map.on('click', 'icons', e => {
        this.selectedLocation = JSON.parse(e.features[0].properties.info)
      })
    },
    setKeyLocations() {
      this.keyLocations.forEach(({img, coordinates}) => {
        this.map.loadImage(img, (err, image) => {
          if (err) throw err
          this.map.addImage(img, image)
          const source = {
            'type': 'geojson',
            'data': {
              'type': 'FeatureCollection',
              'features': [
                {
                  'type': 'Feature',
                  'geometry': {
                    'type': 'Point',
                    'coordinates': coordinates
                  }
                }
              ]
            }
          }
          this.map.addLayer({
            'id': img,
            'type': 'symbol',
            'source': source,
            'layout': {
              'icon-image': img,
              'icon-size': 0.25
            }
          });
        })
      })
      
    },
    convertLocationsToGeojson(locations) {
      const features = locations.map(location => {
        const { address, name, phone, site, category } = location
        const info = {
          address,
          name,
          phone,
          site,
          category
        }
        return  {
          'type': 'Feature',
          'properties': {
            'icon': location.type,
            'info': info
          },
          'geometry': {
            'type': 'Point',
            'coordinates': [location.coordinates.lng, location.coordinates.lat]
          }
        }
      })
      return {
        'type': 'FeatureCollection',
        'features': features
      }
    },
    getRouteData() {
      firebase.firestore().doc(this.route.data).get().then(doc => {
        if (doc.exists) {
          const data = doc.data()
          this.setRoute(data.coordinates)
          const routePoints = data.routePoints ? data.routePoints : data.bikePoints
          if (routePoints.length > 0) {
            this.setRoutePoints(routePoints)
          }
          this.loading = false
        }
      }).catch(err => {
        console.log(err)
      })
    },
    getLocations() {
      firebase.firestore().collection('locations').get().then(snapshot => {
        let locations = []
        snapshot.forEach(doc => {
          locations.push({ id: doc.id, ...doc.data() })
        })
        this.setLocations(locations)
      })
    },
    getImage() {
      const storage = firebase.storage()
      storage.ref(this.route.coverImage).getDownloadURL().then(url => {
        this.coverImage = url
      }).catch(err => {
        console.log(err)
      })
    }
  },
  created() {
    if (this.route) {
      this.getImage()
      this.loading = true
    }
  },
  mounted() {
    // initialize map
    this.map = new mapbox.Map({
      container: 'map',
      style: this.mapStyle,
      attributionControl: false,
      center: [4.9443857, 51.5416528],
      zoom: 10
    })
    this.map.on('load', () => {
      if (this.route) {
        this.getRouteData()
      } else {
        // if on /kaart page 
        this.map.scrollZoom.disable()
        this.map.addControl(new mapbox.NavigationControl(), 'bottom-right')
      }
      this.getLocations()
      if (!isOranjenassau) { this.setKeyLocations() }
    })

    // setup user location
    if (window.navigator.geolocation) {
      // watch user location
      window.navigator.geolocation.watchPosition(pos => {
        this.showUserLocationBtn = true
        this.userPosition = [pos.coords.longitude, pos.coords.latitude]
        this.showUserPoint()
      }, () => {
        this.showUserLocationBtn = false
      }, { timeout: 60000 })
    }
  }
}
</script>
