<template>
  <div v-if="routeFound">
    <div v-if="!loading">
      <Routes
        v-if="route.data === null"
        :group="route"
      />
      <Route
        v-else
        :route="route"
        :prevRoute="from"
      />
    </div>
  </div>
  <NotFound v-else/>
</template>

<script>
import firebase from 'firebase/app'

import { routesCollection } from '../global'

import Routes from '@/components/subViews/Routes'
import Route from '@/components/subViews/Route'
import NotFound from '@/components/NotFound'

export default {
  name: 'RouteRoutes',
  components: {
    Routes,
    Route,
    NotFound
  },
  data() {
    return {
      loading: true,
      from: null,
      route: null,
      routeFound: true
    }
  },
  methods: {
    getRouteRoutes() {
      const slug = this.$route.params.slug.toLowerCase()
      firebase.firestore().collection(routesCollection).where('slug', '==', slug).get().then(snapshot => {
        this.loading = false
        if (!snapshot.empty) {
          const doc = snapshot.docs[0]
          const data = doc.data()
          if (data.group === null) {
            this.route = { id: doc.id, ...data }
          } else {
            this.routeFound = false
          }
        } else {
          this.routeFound = false
        }
      }).catch(err => {
        this.loading = false
        this.routeFound = false
      })
    }
  },
  beforeRouteEnter(to, from, next) {
    next((vm) => {
      vm.from = from
    })
  },
  created() {
    this.getRouteRoutes()
  },
  watch: {
    '$route.params.slug'() {
      this.getRouteRoutes()
    }
  }
}
</script>
