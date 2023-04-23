<template>
  <div id="map" style="width: 1000px;height: 800px;"></div>
</template>

<script>
import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css'
import mapboxdraw from "@mapbox/mapbox-gl-draw"
import "@mapbox/mapbox-gl-draw/dist/mapbox-gl-draw.css"
import * as turf from '@turf/turf'

import { onMounted, ref } from 'vue'

export default {
  name: 'App',
  setup() {
    let drawCoordinates = ref([])
    onMounted(() => {
      mapboxgl.accessToken = 'pk.eyJ1Ijoib25lZ2lzZXIiLCJhIjoiY2xnb2pscDQ5MDg1MDNlczgzNXpqbmsyYSJ9.xx0-MRvzdw8keBhtxr4qEA';
      const map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v11',
        center: [-103.5917, 40.6699],
        zoom: 3
      });
      // console.log(map)
      map.on('load', () => {
        map.addSource('earthquakes', {
          type: 'geojson',
          data: 'https://docs.mapbox.com/mapbox-gl-js/assets/earthquakes.geojson',
          cluster: true,
          clusterMaxZoom: 14, // Max zoom to cluster points on
          clusterRadius: 50 // Radius of each cluster when clustering points (defaults to 50)
        });

        map.addLayer({
          id: 'clusters',
          type: 'circle',
          source: 'earthquakes',
          filter: ['has', 'point_count'],
          paint: {
            'circle-color': [
              'step',
              ['get', 'point_count'],
              '#51bbd6',
              100,
              '#f1f075',
              750,
              '#f28cb1'
            ],
            'circle-radius': [
              'step',
              ['get', 'point_count'],
              20,
              100,
              30,
              750,
              40
            ]
          }
        });
      })

      let draw = new mapboxdraw({
        displayControlsDefault: false,
        controls: {
          polygon: true,
          trash: true
        }
      });
      map.addControl(draw);
      map.on('draw.create', function (e) {
        var features = draw.getAll();
        drawCoordinates.value = features.features[0].geometry.coordinates
        var features_Layer = map.queryRenderedFeatures(e.point, {
          layers: ['clusters']
        });
        var selectedFeatures = []
        drawCoordinates.value.forEach(coordinate => {
          features_Layer.forEach(feature => {
            const point = turf.point(feature.geometry.coordinates) // 将要素转为 Turf.js 中的 Point 对象
            if (turf.booleanPointInPolygon(point, turf.polygon([coordinate]))) { // 判断要素是否在绘制的区域内
              selectedFeatures.push(feature) // 如果在区域内，则添加到选中要素数组中
            }
          })
        })
        // 将选中的要素添加到目标数组 便于后续高亮显示
        const source = {
          type: 'geojson',
          data: {
            type: 'FeatureCollection',
            features: selectedFeatures
          }
        }
        map.addLayer({
          id: 'selected',
          type: 'circle',
          source: source,
          paint: {
            'circle-radius': 15,
            'circle-color': '#6800F0'
          }
        })
        // console.log(selectedFeatures)
      })
      map.on('draw.delete', function (e) {
        // console.log(e)
        map.removeLayer('selected')
      });
    })
  }

}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
