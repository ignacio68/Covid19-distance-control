<template>
  <GridLayout>
    <Frame id="map-page">
      <MapComponent
        row="0"
        height="100%"
        :accessToken="accessToken"
        :zoomLevel="15"
        :userLatitude="initialLocation.lat"
        :userLongitude="initialLocation.lng"
        @on-map-ready="onMapReady($event)"
        @set-center="centerCamera()"
      />
    </Frame>
    <Frame
      id="bottomSheet"
      ref="bottomSheet"
      borderTopLeftRadius="16"
      borderTopRightRadius="16"
      backgroundColor="white"
      verticalAlignment="top"
      @loaded="loadBottomSheet"
    >
      <StackLayout>
        <NewMarker
          v-if="isNewMarkerMenuShowing"
          class="newMarker m-16"
          backgroundColor="white"
          @enabled-fab="isEnabledFAB"
          @on-new-marker-cancel="hideBottomSheet()"
          @on-new-marker-done="newMarker"
        />
        <NewArea
          v-if="isNewAreaMenuShowing"
          class="newArea m-16"
          backgroundColor="white"
          @enabled-fab="isEnabledFAB"
          @on-new-area-cancel="hideBottomSheet()"
          @on-new-area-done="newSecurityArea"
        />
      </StackLayout>
    </Frame>
  </GridLayout>
  <!-- <Label>
        <FormattedString>
          <Span text="Latitude: " />
          <Span :text="currentUserLocation.lat" />
        </FormattedString>
      </Label>
      <Label>
        <FormattedString>
          <Span text="Longitude: " />
          <Span :text="currentUserLocation.lng" />
        </FormattedString>
      </Label> -->
</template>

<script lang="ts">
import Vue from 'vue'

import { mapToken } from '@/setup/map'

import {  addSource,
          setCenter,
          addMarker,
          updateMarker,
          setSecurityArea,
          showSecurityArea,
          removeSecurityArea
        } from '@/api/map'
// import * as map from '@/api/map'

// import { setStorage } from '@/api/storage'
import { Color } from '@nativescript/core/color'
import { screen } from '@nativescript/core/platform'

import { CubicBezierAnimationCurve } from  '@nativescript/core/ui/animation'

import { Marker, PolygonOptions } from '@/utils/types'

import { getMap as map } from '@/store/map'
import {
          getOriginLocation as initialLocation,
          getCurrentUserLocation as userLocation
        } from '@/store/userLocation'
import securityArea from '@/store/securityArea'

import MapComponent from '@/components/Map/MapComponent.vue'
import NewMarker from '@/components/Map/NewMarker.vue'
import NewArea from '@/components/Map/NewArea.vue'

export default Vue.extend({
  name: 'Map',

  components: {
    MapComponent,
    NewMarker,
    NewArea
  },

  props:{
    isVisible: {
      type: Boolean,
      default: true
    },
    isNewMarkerMenuShowing: {
      type: Boolean,
      default: false
    },
    isNewAreaMenuShowing: {
      type: Boolean,
      default: false
    }
  },

  data() {
    return {
      accessToken: mapToken,
      // initialLocation: {
      //   lat: '0',
      //   lng: '0',
      // },
      // map: void 0,
      screenHeight: screen.mainScreen.heightDIPs,
      radius: 1,
      fillOpacity: 5,
      activeUser: null,
    }
  },

  computed: {
    map,
    initialLocation,
    // FIXME: the current coordinates are '0'
    userLocation,
    bottomSheet() {
      return this.$refs.bottomSheet.nativeView
    },
    getRadius: {
      get: function() { return this.radius },
      set: function (value: number) {
        this.radius = value
      }
    },
    getFillOpacity: {
      get: function () {return this.fillOpacity},
      set: function (value: number) {
        this.fillOpacity = value
      }
    }
  },

  watch: {
    isVisible: function(newValue) {
      this.showSecurityArea('user', newValue)
    },
    isNewMarkerMenuShowing(newValue){
      newValue === true ? this.showBottomSheet() : this.hideBottomSheet()
    },
    isNewAreaMenuShowing(newValue){
      newValue === true ? this.showBottomSheet() : this.hideBottomSheet()
    }
  },

  mounted() {
    console.log('mounted()')
    this.$root.$on('remove-security-area', value => this.removeSecurityArea(value.name))
  },

  methods: {
    isEnabledFAB(bool) {
      console.log(`isEnabledFAB(): ${bool}`)
      this.$emit('enabled-fab', bool)
    },
    // onRadiusChange(value) {
    //    this.getRadius = value
    // },

    /***** BOTTOM SHEET *****/
    loadBottomSheet() {
      this.bottomSheet.translateY = this.screenHeight
    },
    showBottomSheet() {
      this.animationBottomSheet(600)
    },
    hideBottomSheet() {
      this.animationBottomSheet(0)
    },
    animationBottomSheet(height: number) {
      this.bottomSheet.animate({
        duration: 1000,
        translate: { x: 0, y: this.screenHeight - height },
        curve: new CubicBezierAnimationCurve(.44, .63, 0, 1)
      })
    },

    /***** MAP *****/

    onMapReady() {
      console.log('onMapReady()')
      // this.map = args.map
      addSource(this.map, 'main') // TODO: Change source name
      // this.showMarkers()
    },

    centerCamera() {
      console.log('setCenter()')
      setCenter(this.map)
    },

    /***** markers *****/
    newMarker(values) {
      console.log(`newMarker()`)
      // if(!values.id || !values.coordinates) {
      //   this.hasNewMarkerError = true
      //   return
      // }
      // TODO: activate when fix userLoacation
      // const coordinates = this.userLocation
      const marker: Marker = {
        id: values.id,
        group: values.group,
        lat: values.coordinates.lat,
        lng: values.coordinates.lng,
        title: values.id,
        selected: true,
        onTap: () => this.newSecurityArea(values.id, values.color),
      }
      // TODO: sustituir por el codigo de abajo
      addMarker(this.map, marker)
      this.hideBottomSheet()
      // setStorage(marker.id, marker).then(success => {
      //   console.log(`setStorage? ${success}`)
      //   if(success){
      //     addMarker(this.map, marker)
      //   }
      // })

    },
    updateMarker(id: string) {
      updateMarker(this.map, id)
    },
    removeMarker(id: string) {
      this.map.removeMarkers(this.map, id)
    },
    showMarkers() {
      this.activeUser = 'user'
      this.newMarker(this.activeUser)
      console.log('showMarkers')
      console.dir(JSON.stringify(this.markers))
    },

    /***** SECURITY AREA ******/

    newSecurityArea(id: string, color: any) {
      if (securityArea.getSecurityArea(id)) {
        console.log(`${id} exist, choose another name`)
        return
      }
      const polygonOptions: PolygonOptions = {
        id: id,
        radius: this.radius,
        fillColor: new Color(color),
        fillOpacity: this.fillOpacity / 10,
        isVisible: true
      }
      setSecurityArea(this.map, polygonOptions)
    },

    showSecurityArea(id: string, value:boolean) {
      showSecurityArea(this.map, id, value)
    },

    removeSecurityArea(id: string) {
      console.log(`remove polygon: ${id}`)
      removeSecurityArea(this.map, id)
    }
  }
})
</script>
<style lang="scss" scoped>
@import '../../app-variables';

.newMarker {
  color: $primary-dark;
}
</style>
