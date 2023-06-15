<template>
  <div>
    <div ref="mapContainer" class="map-container"></div>
    <div class="list">
      <div
        class="list__item"
        :class="{'list__item--active' : item.properties.name === selectedDistrict}"
        v-for="item in Geo.features"
        :key="item.properties.name"
        @click="selectDistrict(item.geometry.coordinates, item.properties.name)"
      >
        {{ item.properties.name }}
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, Ref } from 'vue';
import mapboxgl, { Map, LngLatBounds, LngLatBoundsLike } from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';
import * as Geo from '@/assets/file.json';


const mapContainer: Ref<HTMLElement | null> = ref(null);
const mapInstance: Ref<Map | null> = ref(null);
const selectedDistrict: Ref<string> = ref('');
const districtBounds: Ref<LngLatBoundsLike | null> = ref(null);


onMounted(() => {
  mapboxgl.accessToken = 'pk.eyJ1IjoidGlta2EyNTA1MDAiLCJhIjoiY2txNmoza2UxMGx5bDJwbXRxendkNnczOSJ9.8fcNQdaFl-V2kBOI7yP_wQ';

  const map = new mapboxgl.Map({
    container: mapContainer.value ?? '',
    style: 'mapbox://styles/mapbox/streets-v11',
    zoom: 8,
  });

  mapInstance.value = map;

  
  map.on('load', () => {

    map.addSource('polygon', {
      type: 'geojson',
      data: {
        type: 'FeatureCollection',
        features: Geo.features,
      },
    });

    map.addLayer({
      id: 'line',
      type: 'line',
      source: 'polygon',
      paint: {
        'line-width': 2,
        'line-color': '#0080ff'
      },
    });

    map.addLayer({
      id: 'fill',
      type: 'fill',
      source: 'polygon',
      paint: {
        'fill-color': '#0080ff',
        'fill-opacity': 0.1,
      },
    });

    calculateDistrictBounds()

    if(districtBounds.value) map.fitBounds(districtBounds.value, { padding: 20, maxZoom: 10, duration: 0 });
  });
});


/**
 * Формирование корректных гео-данных
 * @param {array} coord координаты
 * @return {array}
 */
function formatingData(coord: any[]): any[] {
  if (typeof coord?.[0]?.[0] === 'number') {
    return formatingData(coord[0]);
  } else {
    return coord?.length === 1
      ? [...coord[0]] :
      coord.reduce((accumulator, currentValue) => {
        const correctValue = typeof currentValue?.[0]?.[0] === 'number'
          ? currentValue
          : currentValue[0]
        accumulator = [...accumulator, ...correctValue]
        return accumulator
      }, [])
  }
}

/**
 * Вычисление крайних координат
 */
function calculateDistrictBounds() {
  const bounds = Geo.features.reduce((acc: LngLatBounds, feature: { geometry: { coordinates: any; }; }) => {
    const coordinates = feature.geometry.coordinates;
    const featureBounds = formatingData(coordinates).reduce((innerAcc, coord) => {
      return innerAcc.extend(coord);
    }, new mapboxgl.LngLatBounds());
    acc.extend(featureBounds);
    return acc;
  }, new mapboxgl.LngLatBounds());
  districtBounds.value = bounds;
}


/**
 * Позиционирование карты
 * @param {array} coord координаты
 * @param {boolean} isClick вызвано ли действие при клике
 */
function fitToBounds(coord: any[], isClick: boolean) {
  const map = mapInstance.value;
  const bounds = formatingData(coord).reduce((bounds, coordinates) => {
    return bounds.extend(coordinates);
  }, new mapboxgl.LngLatBounds());

  map?.fitBounds(bounds, { padding: 20, zoom: isClick ? 9 : 7});
}


/**
 * Обработка клика по району
 * @param {array} coord координаты
 */
function selectDistrict(coord: any[], name: string) {
  fitToBounds(coord, true);
  selectedDistrict.value = name;
  const fillLayer = mapInstance.value?.getLayer('fill');
  if (fillLayer) {
    mapInstance.value?.setPaintProperty('fill', 'fill-color', [
      'case',
      ['==', ['get', 'name'], selectedDistrict.value],
      '#ff0000',
      '#0080ff',
    ]);
    mapInstance.value?.setPaintProperty('line', 'line-width', [
      'case',
      ['==', ['get', 'name'], selectedDistrict.value],
      5,
      2,
    ]);
  }
}

</script>

<style lang="scss" scoped>
.map-container {
  width: 100vw;
  height: 100vh;
}
.list {
  height: 50%;
  padding: 15px;
  background-color: rgba(0,	128,	255, 0.7);
  position: fixed;
  top: 0;
  left: 0;
  overflow: auto;

  &__item {
    cursor: pointer;
    margin-bottom: 5px;
    color: #000;
    &:hover {
      color: #fff;;
    }
    &--active {
      color: #fff;
    }
    &:last-child {
      margin-bottom: 0;
    }
  }
}
</style>