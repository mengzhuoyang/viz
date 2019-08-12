<template lang="pug">
#container
  .status-blob(v-if="loadingText"): p {{ loadingText }}

  .map-container
    #mymap

  left-data-panel.left-panel(v-if="!loadingText")
   .dashboard-panel
    .info-header
      h3(style="text-align: center; padding: 0.5rem 3rem; font-weight: normal;color: white;")
        | {{this.visualization.title ? this.visualization.title : this.visualization.type.toUpperCase()}}

    .info-description(style="padding: 0px 0.5rem;" v-if="this.visualization.parameters.description")
      p.description {{ this.visualization.parameters.description.value }}

    .widgets
      h4.heading Time of day:
      time-slider.time-slider(v-if="headers.length>0" :useRange='showTimeRange' :stops='headers' @change='bounceTimeSlider')
      span.checkbox
         input(type="checkbox" v-model="showTimeRange")
         | &nbsp;Show range

      h4.heading Line scale:

      scale-slider.scale-slider(:stops='scaleValues' :initialTime='1' @change='bounceScaleSlider')

      h4.heading Show totals for:
      .buttons-bar
        // {{rowName}}
        button.button(@click='clickedOrigins' :class='{"is-link": isOrigin ,"is-active": isOrigin}') Origins
        // {{colName}}
        button.button(hint="hide" @click='clickedDestinations' :class='{"is-link": !isOrigin,"is-active": !isOrigin}') Destinations

  .map-complications(v-if="!isMobile()")
    legend-box.complication(:rows="legendRows")
    scale-box.complication(:rows="scaleRows")


</template>

<script lang="ts">
'use strict'

import * as BlobUtil from 'blob-util'
import * as shapefile from 'shapefile'
import * as turf from '@turf/turf'
import colormap from 'colormap'
import { debounce } from 'debounce'
import mapboxgl, { MapMouseEvent, PositionOptions } from 'mapbox-gl'
import proj4 from 'proj4'
import vegaEmbed from 'vega-embed'
import VueSlider from 'vue-slider-component'
import { Vue, Component, Prop, Watch } from 'vue-property-decorator'

import AuthenticationStore from '@/auth/AuthenticationStore'
import Coords from '@/components/Coords'
import FileAPI from '@/communication/FileAPI'
import LeftDataPanel from '@/components/LeftDataPanel.vue'
import LegendBox from '@/visualization/aggregate-od/LegendBoxOD.vue'
import ScaleBox from '@/visualization/aggregate-od/ScaleBoxOD.vue'
import TimeSlider from '@/visualization/aggregate-od/TimeSlider2.vue'
import SharedStore from '@/SharedStore'
import ScaleSlider from '@/components/ScaleSlider.vue'
import { Visualization } from '@/entities/Entities'
import { multiPolygon } from '@turf/turf'
import { FeatureCollection, Feature } from 'geojson'
import SystemLeftBar from '../components/SystemLeftBar.vue'
import ModalVue from '../components/Modal.vue'

const TOTAL_MSG = 'All >>'
const FADED = 0.0 // 0.15

/*
const vegaChart: any = {
  $schema: 'https://vega.github.io/schema/vega-lite/v3.json',
  description: 'A simple bar chart with embedded data.',
  data: {
    values: [{ Hour: 0, 'Trips From': 0, 'Trips To': 0 }],
  },
  mark: 'bar',
  encoding: {
    x: { field: 'Hour', type: 'ordinal' },
    y: { field: 'Trips From', type: 'quantitative' },
  },
}
*/

const SCALE = [1, 3, 5, 10, 25, 50, 100, 150, 200, 300, 400, 450, 500]

const INPUTS = {
  OD_FLOWS: 'O/D Flows (.csv)',
  SHP_FILE: 'Shapefile .SHP',
  DBF_FILE: 'Shapefile .DBF',
}

@Component({
  components: {
    LeftDataPanel,
    LegendBox,
    ScaleBox,
    ScaleSlider,
    TimeSlider,
  },
})
class AggregateOD extends Vue {
  @Prop({ type: String, required: true })
  private vizId!: string

  @Prop({ type: String, required: true })
  private projectId!: string

  @Prop({ type: FileAPI, required: true })
  private fileApi!: FileAPI

  @Prop({ type: AuthenticationStore, required: true })
  private authStore!: AuthenticationStore

  // -------------------------- //

  private mySharedState = SharedStore
  private centroids: any = {}
  private centroidSource: any = {}
  private linkData: any = {}
  private spiderLinkFeatureCollection: any = {}

  private zoneData: any = {} // [i][j][timePeriod] where [-1] of each is totals
  private dailyData: any = {} // [i][j]
  private marginals: any = {}
  private hoveredStateId: any = 0

  private rowName: string = ''
  private colName: string = ''
  private headers: string[] = []

  private geojson: any = {}

  private showTimeRange = false

  private isOrigin: boolean = true
  private selectedCentroid = 0
  private maxZonalTotal: number = 0

  private loadingText: string = 'Aggregate Origin/Destination Flows'
  private mymap!: mapboxgl.Map
  private project: any = {}
  private visualization!: Visualization

  private scaleFactor: any = 1
  private sliderValue: number[] = [1, 500]
  private scaleValues = SCALE
  private currentScale = SCALE[0]
  private currentTimeBin = TOTAL_MSG

  private projection!: string
  private hoverId: any

  private _mapExtentXYXY!: any
  private _maximum!: number

  private dailyFrom: any
  private dailyTo: any

  private bounceTimeSlider = debounce(this.changedTimeSlider, 100)
  private bounceScaleSlider = debounce(this.changedScale, 50)

  public async created() {
    SharedStore.setFullPage(true)
    this._mapExtentXYXY = [180, 90, -180, -90]
    this._maximum = 0
  }

  public destroyed() {
    SharedStore.setFullPage(false)
  }

  public async mounted() {
    this.projectId = (this as any).$route.params.projectId
    this.vizId = (this as any).$route.params.vizId

    await this.getVizDetails()
    SharedStore.setBreadCrumbs([
      { label: this.visualization.title, url: '/' },
      { label: this.visualization.project.name, url: '/' },
    ])
    this.setupMap()
  }

  @Watch('showTimeRange')
  private clickedRange(useRange: boolean) {
    console.log(useRange)
  }

  private async getVizDetails() {
    this.visualization = await this.fileApi.fetchVisualization(this.projectId, this.vizId)
    this.project = await this.fileApi.fetchProject(this.projectId)

    if (this.visualization.parameters.Projection) {
      this.projection = this.visualization.parameters.Projection.value
    }
    if (this.visualization.parameters['Scale Factor']) {
      this.scaleFactor = parseFloat(this.visualization.parameters['Scale Factor'].value)
    }
  }

  private get legendRows() {
    return ['#00aa66', '#880033', '↓', '↑']
  }

  private setupMap() {
    this.mymap = new mapboxgl.Map({
      container: 'mymap',
      logoPosition: 'top-left',
      style: 'mapbox://styles/mapbox/outdoors-v9',
    })

    try {
      const extent = localStorage.getItem(this.vizId + '-bounds')
      if (extent) {
        const lnglat = JSON.parse(extent)

        const mFac = this.isMobile ? 0 : 1
        const padding = { top: 50 * mFac, bottom: 100 * mFac, right: 100 * mFac, left: 300 * mFac }

        this.mymap.fitBounds(lnglat, {
          padding,
          animate: false,
        })
      }
    } catch (e) {
      // no consequence if json was weird, just drop it
    }

    this.mymap.on('click', this.handleEmptyClick)
    // Start doing stuff AFTER the MapBox library has fully initialized
    this.mymap.on('load', this.mapIsReady)
    // this.mymap.addControl(new mapboxgl.ScaleControl(), 'bottom-right')
    this.mymap.addControl(new mapboxgl.NavigationControl(), 'top-right')
  }

  private handleEmptyClick(e: mapboxgl.MapMouseEvent) {
    this.fadeUnselectedLinks(-1)
    if (this.isMobile()) {
      // do something
    }
  }

  private async mapIsReady() {
    const files = await this.loadFiles()
    if (files) {
      this.setMapExtent()
      this.geojson = await this.processInputs(files)
      this.processHourlyData(files.odFlows)
      this.marginals = this.getDailyDataSummary()
      this.buildCentroids(this.geojson)
      this.convertRegionColors(this.geojson)
      this.addGeojsonToMap(this.geojson)
      this.setMapExtent()
      this.buildSpiderLinks()
      this.setupKeyListeners()
    }

    this.loadingText = ''
  }

  private isMobile() {
    const w = window
    const d = document
    const e = d.documentElement
    const g = d.getElementsByTagName('body')[0]
    const x = w.innerWidth || e.clientWidth || g.clientWidth
    const y = w.innerHeight || e.clientHeight || g.clientHeight

    return x < 640
  }

  private get scaleRows() {
    return [
      Math.min(
        Math.round((1200 * Math.pow(this.currentScale, -1) + 20) * Math.sqrt(this.scaleFactor)),
        1000 * this.scaleFactor
      ),
    ]
  }

  private buildSpiderLinks() {
    this.spiderLinkFeatureCollection = { type: 'FeatureCollection', features: [] }

    for (const id in this.linkData) {
      if (!this.linkData.hasOwnProperty(id)) continue

      const link: any = this.linkData[id]
      try {
        const origCoord = this.centroids[link.orig].geometry.coordinates
        const destCoord = this.centroids[link.dest].geometry.coordinates
        const color = origCoord[1] - destCoord[1] > 0 ? '#00aa66' : '#880033'
        const fade = 0.7
        const properties: any = {
          id: id,
          orig: link.orig,
          dest: link.dest,
          daily: link.daily,
          color,
          fade,
        }
        ;(properties[TOTAL_MSG] = link.daily),
          link.values.forEach((value: number, i: number) => {
            properties[this.headers[i + 1]] = value ? value : 0
          })

        const feature: any = {
          type: 'Feature',
          properties,
          geometry: {
            type: 'LineString',
            coordinates: [origCoord, destCoord],
          },
        }
        this.spiderLinkFeatureCollection.features.push(feature)
      } catch (e) {
        // some dests aren't on map: z.b. 'other'
      }
    }

    this.mymap.addSource('spider-source', {
      data: this.spiderLinkFeatureCollection,
      type: 'geojson',
    } as any)

    this.mymap.addLayer(
      {
        id: 'spider-layer',
        source: 'spider-source',
        type: 'line',
        paint: {
          'line-color': ['get', 'color'],
          'line-width': ['*', (1 / 500) * this.scaleFactor, ['get', 'daily']],
          'line-offset': ['*', 0.5, ['get', 'daily']],
          'line-opacity': ['get', 'fade'],
        },
      },
      'centroid-layer'
    )

    this.changedScale(this.currentScale)

    const parent = this
    this.mymap.on('click', 'spider-layer', function(e: mapboxgl.MapMouseEvent) {
      parent.clickedOnSpiderLink(e)
    })

    // turn "hover cursor" into a pointer, so user knows they can click.
    this.mymap.on('mousemove', 'spider-layer', function(e: mapboxgl.MapMouseEvent) {
      parent.mymap.getCanvas().style.cursor = e ? 'pointer' : 'grab'
    })

    // and back to normal when they mouse away
    this.mymap.on('mouseleave', 'spider-layer', function() {
      parent.mymap.getCanvas().style.cursor = 'grab'
    })
  }

  private clickedOrigins() {
    this.isOrigin = true
    this.updateCentroidLabels()

    this.convertRegionColors(this.geojson)

    // avoiding mapbox typescript bug:
    const tsMap = this.mymap as any
    tsMap.getSource('shpsource').setData(this.geojson)
  }

  private clickedDestinations() {
    this.isOrigin = false
    this.updateCentroidLabels()

    this.convertRegionColors(this.geojson)

    // avoiding mapbox typescript bug:
    const tsMap = this.mymap as any
    tsMap.getSource('shpsource').setData(this.geojson)
  }

  private updateCentroidLabels() {
    const labels = this.isOrigin ? '{dailyFrom}' : '{dailyTo}'
    const radiusField = this.isOrigin ? 'widthFrom' : 'widthTo'

    if (this.mymap.getLayer('centroid-layer')) {
      this.mymap.removeLayer('centroid-label-layer')
      this.mymap.removeLayer('centroid-layer')
    }

    this.mymap.addLayer({
      id: 'centroid-layer',
      source: 'centroids',
      type: 'circle',
      paint: {
        'circle-color': '#ec0',
        'circle-radius': ['get', radiusField],
        'circle-stroke-width': 3,
        'circle-stroke-color': 'white',
      },
    })

    this.mymap.addLayer({
      id: 'centroid-label-layer',
      source: 'centroids',
      type: 'symbol',
      layout: {
        'text-field': labels,
        'text-size': 11,
      },
    })
  }

  private unselectAllCentroids() {
    this.fadeUnselectedLinks(-1)
    this.selectedCentroid = 0
  }

  private clickedOnCentroid(e: any) {
    console.log({ CLICK: e })

    e.originalEvent.stopPropagating = true

    const centroid = e.features[0].properties
    console.log(centroid)

    const id = centroid.id

    // a second click on a centroid UNselects it.
    if (id === this.selectedCentroid) {
      this.unselectAllCentroids()
      return
    }

    this.selectedCentroid = id

    console.log(this.marginals)
    console.log(this.marginals.rowTotal[id])
    console.log(this.marginals.colTotal[id])

    /* vega chart stuff
    const values = []
    for (let i = 0; i < 24; i++) {
      values.push({ Hour: i + 1, 'Trips From': this.marginals.from[id][i], 'Trips To': this.marginals.to[id][i] })
    }
    // vegaChart.data.values = values
    // vegaEmbed('#mychart', vegaChart)
    */

    this.fadeUnselectedLinks(id)
  }

  private fadeUnselectedLinks(id: any) {
    const tsMap = this.mymap as any

    for (const feature of this.spiderLinkFeatureCollection.features) {
      const endpoints = feature.properties.id.split(':')
      let fade = endpoints[0] === String(id) || endpoints[1] === String(id) ? 0.7 : FADED
      if (id === -1) fade = 0.7
      feature.properties.fade = fade
    }
    tsMap.getSource('spider-source').setData(this.spiderLinkFeatureCollection)
  }

  private clickedOnSpiderLink(e: any) {
    if (e.originalEvent.stopPropagating) return

    console.log({ CLICK: e })

    const props = e.features[0].properties
    console.log(props)

    const trips = props.daily * this.scaleFactor
    let revTrips = 0
    const reverseDir = '' + props.dest + ':' + props.orig

    if (this.linkData[reverseDir]) revTrips = this.linkData[reverseDir].daily * this.scaleFactor

    const totalTrips = trips + revTrips

    let html = `<h1>${totalTrips} Bidirectional Trips</h1><br/>`
    html += `<p> -----------------------------</p>`
    html += `<p>${trips} trips : ${revTrips} reverse trips</p>`

    new mapboxgl.Popup({ closeOnClick: true })
      .setLngLat(e.lngLat)
      .setHTML(html)
      .addTo(this.mymap)
  }

  private convertRegionColors(geojson: FeatureCollection) {
    for (const feature of geojson.features) {
      if (!feature.properties) continue

      const daily = this.isOrigin ? feature.properties.dailyFrom : feature.properties.dailyTo
      const ratio = daily / this.maxZonalTotal

      let blue = 128 + 127 * (1.0 - ratio)
      if (!blue) blue = 255

      feature.properties.blue = blue
    }
  }

  private handleCentroidsForTimeOfDayChange(timePeriod: any) {
    const centroids: FeatureCollection = { type: 'FeatureCollection', features: [] }

    for (const feature of this.geojson.features) {
      const centroid: any = turf.centerOfMass(feature as any)

      centroid.properties.id = feature.id

      const values = this.calculateCentroidValuesForZone(timePeriod, feature)

      centroid.properties.dailyFrom = values.from * this.scaleFactor
      centroid.properties.dailyTo = values.to * this.scaleFactor

      this.dailyFrom = centroid.properties.dailyFrom
      this.dailyTo = centroid.properties.dailyTo

      centroid.properties.widthFrom = Math.min(
        70,
        Math.max(12, Math.sqrt(this.dailyFrom / this.scaleFactor) * (1.5 + this.scaleFactor / (this.scaleFactor + 50)))
      )
      centroid.properties.widthTo = Math.min(
        70,
        Math.max(12, Math.sqrt(this.dailyTo / this.scaleFactor) * (1.5 + this.scaleFactor / (this.scaleFactor + 50)))
      )

      if (!feature.properties) feature.properties = {}

      // bc/ is this correct?
      feature.properties.dailyFrom = values.from
      feature.properties.dailyTo = values.to

      if (centroid.properties.dailyFrom + centroid.properties.dailyTo > 0) {
        centroids.features.push(centroid)
        if (feature.properties) this.centroids[feature.properties.NO] = centroid
      }
    }

    this.centroidSource = centroids

    const tsMap = this.mymap as any
    tsMap.getSource('centroids').setData(this.centroidSource)
    this.updateCentroidLabels()
  }

  private calculateCentroidValuesForZone(timePeriod: any, feature: any) {
    let from = 0
    let to = 0

    // daily
    if (timePeriod === 'All >>') {
      from = Math.round(this.marginals.rowTotal[feature.id as any])
      to = Math.round(this.marginals.colTotal[feature.id as any])
      return { from, to }
    }

    // time range
    if (timePeriod.constructor === Array) {
      let hourFrom = parseInt(timePeriod[0], 10)
      if (!hourFrom) hourFrom = 1

      const hourTo = parseInt(timePeriod[1], 10)

      for (let i = hourFrom; i <= hourTo; i++) {
        from += Math.round(this.marginals.from[feature.id as any][i])
        to += Math.round(this.marginals.to[feature.id as any][i])
      }
      return { from, to }
    }

    // single point in time
    const hour = parseInt(timePeriod, 10)
    from = Math.round(this.marginals.from[feature.id as any][hour])
    to = Math.round(this.marginals.to[feature.id as any][hour])
    return { from, to }
  }

  private buildCentroids(geojson: FeatureCollection) {
    const centroids: FeatureCollection = { type: 'FeatureCollection', features: [] }

    for (const feature of geojson.features) {
      const centroid: any = turf.centerOfMass(feature as any)

      centroid.properties.id = feature.id
      const dailyFrom = Math.round(this.marginals.rowTotal[feature.id as any])
      const dailyTo = Math.round(this.marginals.colTotal[feature.id as any])

      centroid.properties.dailyFrom = dailyFrom * this.scaleFactor
      centroid.properties.dailyTo = dailyTo * this.scaleFactor

      this.dailyFrom = centroid.properties.dailyFrom
      this.dailyTo = centroid.properties.dailyTo

      centroid.properties.widthFrom = Math.min(
        70,
        Math.max(12, Math.sqrt(this.dailyFrom / this.scaleFactor) * (1.5 + this.scaleFactor / (this.scaleFactor + 50)))
      )
      centroid.properties.widthTo = Math.min(
        70,
        Math.max(12, Math.sqrt(this.dailyTo / this.scaleFactor) * (1.5 + this.scaleFactor / (this.scaleFactor + 50)))
      )

      if (dailyFrom) this.maxZonalTotal = Math.max(this.maxZonalTotal, dailyFrom)
      if (dailyTo) this.maxZonalTotal = Math.max(this.maxZonalTotal, dailyTo)

      if (!feature.properties) feature.properties = {}
      feature.properties.dailyFrom = dailyFrom
      feature.properties.dailyTo = dailyTo

      if (centroid.properties.dailyFrom + centroid.properties.dailyTo > 0) {
        centroids.features.push(centroid)
        if (feature.properties) this.centroids[feature.properties.NO] = centroid
        this.updateMapExtent(centroid.geometry.coordinates)
      }
    }

    this.centroidSource = centroids

    console.log({ CENTROIDS: this.centroids })

    this.mymap.addSource('centroids', {
      data: this.centroidSource,
      type: 'geojson',
    } as any)

    this.updateCentroidLabels()

    const parent = this

    this.mymap.on('click', 'centroid-layer', function(e: mapboxgl.MapMouseEvent) {
      parent.clickedOnCentroid(e)
    })

    // turn "hover cursor" into a pointer, so user knows they can click.
    this.mymap.on('mousemove', 'centroid-layer', function(e: mapboxgl.MapMouseEvent) {
      parent.mymap.getCanvas().style.cursor = e ? 'pointer' : 'grab'
    })

    // and back to normal when they mouse away
    this.mymap.on('mouseleave', 'centroid-layer', function() {
      parent.mymap.getCanvas().style.cursor = 'grab'
    })
  }

  private setMapExtent() {
    localStorage.setItem(this.vizId + '-bounds', JSON.stringify(this._mapExtentXYXY))
    this.mymap.fitBounds(this._mapExtentXYXY, {
      padding: { top: 50, bottom: 100, right: 100, left: 300 },
      animate: false,
    })
  }

  private setupKeyListeners() {
    const parent = this
    window.addEventListener('keyup', function(event) {
      if (event.keyCode === 27) {
        // ESC
        parent.pressedEscape()
      }
    })
    window.addEventListener('keydown', function(event) {
      if (event.keyCode === 38) {
        // UP
        parent.pressedArrowKey(-1)
      }
      if (event.keyCode === 40) {
        // DOWN
        parent.pressedArrowKey(+1)
      }
    })
  }

  private async loadFiles() {
    try {
      this.loadingText = 'Loading files...'

      if (SharedStore.debug) console.log(this.visualization.inputFiles)

      const shpfileID = this.visualization.inputFiles[INPUTS.SHP_FILE].fileEntry.id
      const shpBlob = await this.fileApi.downloadFile(shpfileID, this.projectId)
      const shpFile: ArrayBuffer = await BlobUtil.blobToArrayBuffer(shpBlob)

      const dbfID = this.visualization.inputFiles[INPUTS.DBF_FILE].fileEntry.id
      const dbfBlob = await this.fileApi.downloadFile(dbfID, this.projectId)
      const dbfFile: ArrayBuffer = await BlobUtil.blobToArrayBuffer(dbfBlob)

      const odFlowsID = this.visualization.inputFiles[INPUTS.OD_FLOWS].fileEntry.id
      const odBlob = await this.fileApi.downloadFile(odFlowsID, this.projectId)
      const odFlows: string = await BlobUtil.blobToBinaryString(odBlob)

      return { shpFile, dbfFile, odFlows }
    } catch (e) {
      console.error({ e })
      this.loadingText = '' + e
      return null
    }
  }

  private async processInputs(files: any) {
    this.loadingText = 'Converting to GeoJSON...'
    const geojson = await shapefile.read(files.shpFile, files.dbfFile)

    this.loadingText = 'Converting coordinates...'
    for (const feature of geojson.features) {
      // save id
      if (feature.properties) feature.id = feature.properties.NO

      try {
        if (feature.geometry.type === 'MultiPolygon') {
          this.convertMultiPolygonCoordinatesToWGS84(feature)
        } else {
          this.convertPolygonCoordinatesToWGS84(feature)
        }
      } catch (e) {
        console.error('ERR with feature: ' + feature)
        console.error(e)
      }
    }

    console.log(geojson)
    return geojson
  }

  private convertPolygonCoordinatesToWGS84(polygon: any) {
    for (const origCoords of polygon.geometry.coordinates) {
      const newCoords: any = []
      for (const p of origCoords) {
        const lnglat = Coords.toLngLat(this.projection, p) as any
        newCoords.push(lnglat)
      }

      // replace existing coords
      origCoords.length = 0
      origCoords.push(...newCoords)
    }
  }

  private convertMultiPolygonCoordinatesToWGS84(multipolygon: any) {
    for (const origCoords of multipolygon.geometry.coordinates) {
      const coordinates = origCoords[0] // multipolygons have an extra array[0] added

      const newCoords: any = []
      for (const p of coordinates) {
        const lnglat = proj4(this.projection, 'WGS84', p) as any
        newCoords.push(lnglat)
      }

      origCoords[0] = newCoords
    }
  }

  private getDailyDataSummary() {
    const rowTotals: any = []
    const colTotals: any = []
    const fromCentroid: any = {}
    const toCentroid: any = {}

    for (const row in this.zoneData) {
      if (!this.zoneData.hasOwnProperty(row)) continue
      if (!row) continue

      fromCentroid[row] = []

      for (const col in this.zoneData[row]) {
        if (!this.zoneData[row].hasOwnProperty(col)) continue
        if (!col) continue

        // daily totals
        if (!rowTotals[row]) rowTotals[row] = 0
        rowTotals[row] += this.dailyData[row][col]

        if (!colTotals[col]) colTotals[col] = 0
        colTotals[col] += this.dailyData[row][col]

        if (!toCentroid[col]) toCentroid[col] = []

        // time-of-day details
        for (let i = 0; i < 24; i++) {
          if (!fromCentroid[row][i + 1]) fromCentroid[row][i + 1] = 0
          fromCentroid[row][i + 1] += this.zoneData[row][col][i]

          if (!toCentroid[col][i + 1]) toCentroid[col][i + 1] = 0
          toCentroid[col][i + 1] += this.zoneData[row][col][i]
        }
      }
    }
    return { rowTotal: rowTotals, colTotal: colTotals, from: fromCentroid, to: toCentroid }
  }

  private processHourlyData(csvData: string) {
    const lines = csvData.split('\n')
    const separator = lines[0].indexOf(';') > 0 ? ';' : ','

    this.loadingText = 'Processing hourly data...'

    // data is in format: o,d, value[1], value[2], value[3]...
    const headers = lines[0].split(separator).map(a => a.trim())
    this.rowName = headers[0]
    this.colName = headers[1]
    this.headers = [TOTAL_MSG].concat(headers.slice(2))

    console.log(this.headers)

    for (const row of lines.slice(1)) {
      const columns = row.split(separator)
      const values = columns.slice(2).map(a => parseFloat(a))

      // build zone matrix
      const i = columns[0]
      const j = columns[1]
      if (!this.zoneData[i]) this.zoneData[i] = {}
      this.zoneData[i][j] = values

      // calculate daily/total values
      const daily = values.reduce((a, b) => a + b, 0)

      if (!this.dailyData[i]) this.dailyData[i] = {}
      this.dailyData[i][j] = daily

      // save total on the links too
      if (daily !== 0) {
        const rowName = String(columns[0]) + ':' + String(columns[1])
        this.linkData[rowName] = { orig: columns[0], dest: columns[1], daily, values }
      }
    }
    console.log({ LINKS: this.linkData, ZONES: this.zoneData })
  }

  private updateMapExtent(coordinates: any) {
    this._mapExtentXYXY[0] = Math.min(this._mapExtentXYXY[0], coordinates[0])
    this._mapExtentXYXY[1] = Math.min(this._mapExtentXYXY[1], coordinates[1])
    this._mapExtentXYXY[2] = Math.max(this._mapExtentXYXY[2], coordinates[0])
    this._mapExtentXYXY[3] = Math.max(this._mapExtentXYXY[3], coordinates[1])
  }

  private addGeojsonToMap(geojson: any) {
    this.addGeojsonLayers(geojson)
    this.addNeighborhoodHoverEffects()
  }

  private addGeojsonLayers(geojson: any) {
    this.mymap.addSource('shpsource', {
      data: geojson,
      type: 'geojson',
    } as any)

    this.mymap.addLayer(
      {
        id: 'shplayer-fill',
        source: 'shpsource',
        type: 'fill',
        paint: {
          'fill-color': ['rgb', ['get', 'blue'], ['get', 'blue'], 255],
          'fill-opacity': 0.7,
        },
      },
      'road-primary'
    )

    this.mymap.addLayer(
      {
        id: 'shplayer-border',
        source: 'shpsource',
        type: 'line',
        paint: {
          'line-color': '#66f',
          'line-opacity': 0.5,
          'line-width': ['case', ['boolean', ['feature-state', 'hover'], false], 3, 1],
        },
      },
      'centroid-layer'
    )
  }

  private addNeighborhoodHoverEffects() {
    const parent = this
    this.mymap.on('mousemove', 'shplayer-fill', function(e: any) {
      // typescript definitions and mapbox-gl are out of sync at the moment :-(
      // so setFeatureState is missing
      const tsMap = parent.mymap as any
      if (e.features.length > 0) {
        if (parent.hoveredStateId) {
          tsMap.setFeatureState({ source: 'shpsource', id: parent.hoveredStateId }, { hover: false })
        }
        parent.hoveredStateId = e.features[0].properties.NO
        tsMap.setFeatureState({ source: 'shpsource', id: parent.hoveredStateId }, { hover: true })
      }
    })

    // When the mouse leaves the state-fill layer, update the feature state of the
    // previously hovered feature.
    this.mymap.on('mouseleave', 'shplayer-fill', function() {
      const tsMap = parent.mymap as any
      if (parent.hoveredStateId) {
        tsMap.setFeatureState({ source: 'shpsource', id: parent.hoveredStateId }, { hover: false })
      }
      parent.hoveredStateId = null
    })
  }

  private offsetLineByMeters(line: any, metersToTheRight: number) {
    try {
      const offsetLine = turf.lineOffset(line, metersToTheRight, { units: 'meters' })
      return offsetLine
    } catch (e) {
      // offset can fail if points are exactly on top of each other; ignore.
    }
    return line
  }

  private pressedEscape() {
    this.unselectAllCentroids()
  }

  private pressedArrowKey(delta: number) {}

  private changedTimeSlider(value: any) {
    this.currentTimeBin = value
    const widthFactor = (this.currentScale / 500) * this.scaleFactor

    if (!this.showTimeRange) {
      this.mymap.setPaintProperty('spider-layer', 'line-width', ['*', widthFactor, ['get', value]])
      this.mymap.setPaintProperty('spider-layer', 'line-offset', ['*', 0.5 * widthFactor, ['get', value]])
    } else {
      const sumElements: any = ['+']

      // build the summation expressions: e.g. ['+', ['get', '1'], ['get', '2']]
      let include = false
      for (const header of this.headers) {
        if (header === value[0]) include = true

        // don't double-count the total
        if (header === TOTAL_MSG) continue

        if (include) sumElements.push(['get', header])

        if (header === value[1]) include = false
      }

      this.mymap.setPaintProperty('spider-layer', 'line-width', ['*', widthFactor, sumElements])
      this.mymap.setPaintProperty('spider-layer', 'line-offset', ['*', 0.5 * widthFactor, sumElements])
    }

    this.handleCentroidsForTimeOfDayChange(value)
  }

  private changedScale(value: any) {
    console.log({ slider: value, timebin: this.currentTimeBin })
    this.currentScale = value
    this.changedTimeSlider(this.currentTimeBin)
  }
}

// register component with the SharedStore
SharedStore.addVisualizationType({
  component: AggregateOD,
  typeName: 'aggregate-od',
  prettyName: 'Origin/Destination Patterns',
  description: 'Depicts aggregate O/D flows between areas.',
  requiredFileKeys: [INPUTS.OD_FLOWS, INPUTS.SHP_FILE, INPUTS.DBF_FILE],
  requiredParamKeys: ['Projection', 'Scale Factor'],
})

export default AggregateOD
</script>

<style scoped>
h3 {
  margin: 0px 0px;
  font-size: 16px;
}

h4 {
  margin-left: 3px;
}

#container {
  width: 100%;
  display: grid;
  grid-template-columns: auto 1fr;
  grid-template-rows: auto 1fr;
}

.status-blob {
  background-color: white;
  box-shadow: 0 0 8px #00000040;
  margin: auto 0px auto -10px;
  padding: 3rem 0px;
  text-align: center;
  grid-column: 1 / 3;
  grid-row: 1 / 3;
  z-index: 99;
  border-top: solid 1px #479ccc;
  border-bottom: solid 1px #479ccc;
}

.map-container {
  background-color: #eee;
  grid-column: 1 / 3;
  grid-row: 1 / 3;
  display: flex;
  flex-direction: column;
}

#mymap {
  height: 100%;
  width: 100%;
}

.mytitle {
  margin-left: 10px;
  color: white;
}

.details {
  font-size: 12px;
  margin-bottom: auto;
  margin-top: auto;
}

.bigtitle {
  font-weight: bold;
  font-style: italic;
  font-size: 20px;
  margin: 20px 0px;
}

.info-header {
  background-color: #097c43;
  padding: 0.5rem 0rem;
  border-top: solid 1px #888;
  border-bottom: solid 1px #888;
  margin-bottom: 1rem;
}

.project-summary-block {
  width: 16rem;
  grid-column: 1 / 2;
  grid-row: 1 / 2;
  margin: 0px auto 0px 0px;
  z-index: 10;
}

.widgets {
  display: flex;
  flex-direction: column;
  padding: 0px 0.5rem;
  margin-top: auto;
  margin-bottom: 0.5rem;
}

.status-blob p {
  color: #555;
}

.map-complications {
  display: flex;
  grid-column: 1/3;
  grid-row: 1/3;
  margin: auto 0.5rem 1.1rem auto;
  z-index: 4;
}

.complication {
  margin: 0rem 0rem 0.1rem 0.5rem;
}

.buttons-bar {
  display: flex;
  flex-direction: row;
  text-align: center;
}

.buttons-bar button {
  flex-grow: 1;
  width: 50%;
  margin: 0px 2px;
}

.time-slider {
  background-color: white;
  border: solid 1px;
  border-color: #ccc;
  border-radius: 4px;
  margin: 0rem 0px auto 0px;
}

.scale-slider {
  background-color: white;
  border: solid 1px;
  border-color: #ccc;
  border-radius: 4px;
  margin: 0rem 0px auto 0px;
}

.heading {
  text-align: left;
  color: black;
  margin-top: 2rem;
}

.checkbox {
  font-size: 12px;
  margin-top: 0.25rem;
  margin-left: auto;
  margin-right: 0.5rem;
  color: rgba(31, 151, 199, 0.932);
}

.description {
  text-align: center;
  margin-top: 0rem;
  padding: 0rem 0.25rem;
  font-size: 0.8rem;
  color: #555;
}

.hide-button {
  grid-column: 1/2;
  grid-row: 2/3;
  margin: auto auto 0.5rem 16.5rem;
  z-index: 20;
}

.hide-toggle-button {
  margin-left: 0.25rem;
}

.left-panel {
  grid-column: 1 / 3;
  grid-row: 1 / 3;
  display: flex;
  flex-direction: column;
  width: 16rem;
}

.dashboard-panel {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

.mapboxgl-popup-content {
  padding: 0px 20px 0px 0px;
  opacity: 0.95;
  box-shadow: 0 0 3px #00000080;
}

@media only screen and (max-width: 640px) {
}
</style>