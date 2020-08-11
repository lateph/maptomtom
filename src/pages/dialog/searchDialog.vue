<template>
    <q-layout view="Lhh lpR fff" container class="bg-white">
        <q-header class="bg-primary">
            <q-toolbar>
            <q-btn flat v-close-popup round dense icon="navigate_before" />
            <q-input dark dense standout v-model="text" input-class="text-right" class="q-ml-md col-grow">
                <template v-slot:append>
                <q-icon name="search" @click="search"/>
                </template>
            </q-input>
            </q-toolbar>
        </q-header>

        <q-page-container>
            <q-page padding>
            <!-- <p v-for="n in results" :key="n">
                {{ lorem }}
            </p> -->
            <q-list>
                <template v-for="result in results">
                <q-item :key="result.id" @click="selectResult(result)" clickable v-ripple v-close-popup>
                    <q-item-section side>
                    <q-icon name="room" color="grey" />
                    </q-item-section>

                    <q-item-section>
                    <q-item-label>{{result.address.freeformAddress}}</q-item-label>
                    </q-item-section>
                </q-item>
                <q-separator spaced inset  :key="result.id+'insert'"/>
                </template>
            </q-list>
            </q-page>
        </q-page-container>
    </q-layout>
</template>

<script>
const mKey = 'ApQAkJWjATsrDgCAip5JqwHsHKLfbl0D'
import ttServices from '@tomtom-international/web-sdk-services'
import { mapState } from 'vuex'

export default {
  name: 'PageIndex',
  data () {
    return {
      text: '',
      results: []
    }
  },
  computed: {
    ...mapState('map', ['lngLat'])
  },
  mounted () {
  },
  methods: {
    search () {
      let callParameters = {
        key: mKey,
        query: this.text
      }
      // poin to malang
      callParameters = {
        ...callParameters,
        center: this.lngLat
      }
      ttServices.services.fuzzySearch(callParameters).go().then((rets) => {
        this.results = rets.results
      }).catch((e) => {
        console.log(e)
      })
      console.log('start', ttServices)
      console.log('start', callParameters)
    },
    selectResult (result) {
      console.log(result)
      console.log([result.position.lat, result.position.lng])
      this.$emit('markers', this.results, result)
      this.searchModal = false
    }
  }
}
</script>
