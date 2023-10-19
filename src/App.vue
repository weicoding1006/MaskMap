<template>
  <div class="row g-0">
    <div
      class="overflow-scroll list"
      :class="{ active: buttonClick }"
      style="max-height: 100vh"
    >
      <div class="">
        <div class="d-flex align-items-center py-2 px-2">
          <div class="me-2">
            <label for="cityName">縣市:</label>
          </div>
          <div class="flex-grow-1">
            <select name="" id="cityName" class="form-control" v-model="city">
              <option
                :value="item.CityName"
                v-for="(item, i) in cityName"
                :key="i"
              >
                {{ item.CityName }}
              </option>
            </select>
          </div>
        </div>

        <div class="d-flex align-items-center py-2 px-2">
          <div class="me-2">
            <label for="Region">地區:</label>
          </div>
          <div class="flex-grow-1">
            <select name="" id="Region" class="form-control" v-model="region">
              <option :value="item.AreaName" v-for="item in regionName">
                {{ item.AreaName }}
              </option>
            </select>
          </div>
        </div>
      </div>

      <div class="" ref="pharmacyList">
        <div
          v-for="(item, index) in data"
          class="border px-2 pt-2 option"
          @click="handleClick(item, index)"
        >
          <h3>{{ item.properties.name }}</h3>
          <p>
            成人口罩:{{ item.properties.mask_adult }} | 兒童口罩 :
            {{ item.properties.mask_child }}
          </p>
          <p>地址 : {{ item.properties.address }}</p>
        </div>
      </div>
    </div>
    <div class="col">
      <button
        class="btn btn-primary customBtn"
        @click="toggle"
        :class="{ btnActive: buttonClick }"
        v-if="isShow"
      >
        {{ buttonClick ? "關閉" : "開啟" }}
      </button>
      <div id="map"></div>
    </div>
  </div>
</template>

<script setup>
import "leaflet/dist/leaflet.css";
import L from "leaflet";
import axios from "axios";
import { onMounted, ref, watch } from "vue";
let data = ref([]);
let cityName = ref([]);
let regionName = ref([]);
let city = ref("臺北市");
let region = ref("中正區");
const coordinates = ref([]);
const pharmacyListElement = ref(null);

const pharmacyList = ref(null);
//必須在全域定義
let map = null;
let maker = null;

const buttonClick = ref(false);
const toggle = () => {
  buttonClick.value = !buttonClick.value;
};
//監聽縣市 回傳對應的鄉鎮區
watch(city, (newVal, oldVal) => {
  regionName.value = cityName.value.filter((item) => {
    //回傳相對應的縣市如:臺北市 == 臺北市
    return item.CityName == newVal;
  });
  //將鄉鎮區再次賦予
  regionName.value = regionName.value[0].AreaList;
  //地區的部分將預設第一個區
  region.value = regionName.value[0].AreaName;
});

//重撈資料 並且篩選
watch(region, (newVal, oldVal) => {
  getData();
});

//點擊事件將轉換座標
// const handleClick = (item,index) => {
//   const coordinates = [
//     item.geometry.coordinates[1],
//     item.geometry.coordinates[0],
//   ];

//   for (let i = 0; i < data.value.length; i++) {
//         maker = L.marker(coordinates)
//           .addTo(map)
//           .bindPopup(
//             `<div><h5>藥局名稱 :${data.value[i].properties.name} </h5><p class="py-1 m-0">成人口罩: ${data.value[i].properties.mask_adult}</p><p class="py-1 m-0">小孩口罩: ${data.value[i].properties.mask_child}</p><div>`
//           );
//         if (i == index) {
//           maker.openPopup();
//           //將定位點定於撈取到的資料的第一筆經緯度 位置就可以正確
//           map.setView(coordinates, 16);
//         } else {
//           maker.closePopup();
//         }
//       }
//   map.setView(coordinates, 16);
// };

//gpt太帥了 解決上面程式碼是不斷新增節點的問題
const handleClick = (item, index) => {
  const coordinates = [
    item.geometry.coordinates[1],
    item.geometry.coordinates[0],
  ];

  // 清除之前的標記
  map.eachLayer((layer) => {
    if (layer instanceof L.Marker) {
      map.removeLayer(layer);
    }
  });

  // 添加新標記
  const customIcon = L.icon({
    iconUrl:
      "https://github.com/weicoding1006/Image/blob/main/Mask.png?raw=true",
    iconSize: [70, 70],
    iconAnchor: [35, 90],
    popupAnchor: [-3, -76],
  });
  for (let i = 0; i < data.value.length; i++) {
    const markerCoordinates = [
      data.value[i].geometry.coordinates[1],
      data.value[i].geometry.coordinates[0],
    ];

    const maker = L.marker(markerCoordinates, { icon: customIcon })
      .addTo(map)
      .bindPopup(
        `<div><h5>藥局名稱 :${data.value[i].properties.name} </h5><p class="py-1 m-0">成人口罩: ${data.value[i].properties.mask_adult}</p><p class="py-1 m-0">小孩口罩: ${data.value[i].properties.mask_child}</p><div>`
      );

    console.log(maker);
    if (i === index) {
      maker.openPopup();
      // 這裡就是使用item帶進來的新座標
      map.setView(coordinates, 16);
    } else {
      maker.closePopup();
    }
  }
};

const isShow = ref(false);
onMounted(() => {
  //如果將let map寫在該區域會有問題
  if(window.innerWidth > 768){
      isShow.value = false
    }else{
      isShow.value = true;
    }
  window.addEventListener('resize',() => {
    if(window.innerWidth > 768){
      isShow.value = false
    }else{
      isShow.value = true;
    }
  })
  map = L.map("map").setView([25.032361, 121.518267], 10);
  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribuztion:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
  }).addTo(map);
  getData();
  getRegionData();
});

const getData = () => {
  //每次觸發前都先刪除先前打的點
  map.eachLayer((layer) => {
    if (layer instanceof L.Marker) {
      map.removeLayer(layer);
    }
  });

  return axios
    .get(
      "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json"
    )
    .then((res) => {
      data.value = res.data.features;
      data.value = data.value.filter((item) => {
        return (
          item.properties.town == region.value &&
          item.properties.county == city.value
        );
      });

      const customIcon = L.icon({
        iconUrl:
          "https://github.com/weicoding1006/Image/blob/main/Mask.png?raw=true",
        iconSize: [70, 70],
        iconAnchor: [35, 90],
        popupAnchor: [-3, -76],
      });
      for (let i = 0; i < data.value.length; i++) {
        //將經緯度單獨拆出來
        const coordinates = [
          data.value[i].geometry.coordinates[1],
          data.value[i].geometry.coordinates[0],
        ];

        maker = L.marker(coordinates, { icon: customIcon })
          .addTo(map)
          .bindPopup(
            `<div><h5>藥局名稱 :${data.value[i].properties.name} </h5><p class="py-1 m-0">成人口罩: ${data.value[i].properties.mask_adult}</p><p class="py-1 m-0">小孩口罩: ${data.value[i].properties.mask_child}</p><div>`
          );
        if (i == 0) {
          maker.openPopup();
          //將定位點定於撈取到的資料的第一筆經緯度 位置就可以正確
          map.setView(coordinates, 16);
        } else {
          maker.closePopup();
        }
      }
    });
};

//第一次撈取
const getRegionData = () => {
  axios
    .get(
      "https://raw.githubusercontent.com/donma/TaiwanAddressCityAreaRoadChineseEnglishJSON/master/AllData.json"
    )
    .then((res) => {
      cityName.value = res.data;
      // console.log(cityName.value);
      regionName.value = cityName.value.filter((item) => {
        //回傳相對應的縣市如:臺北市 == 臺北市
        return item.CityName == "臺北市";
      });
      //將鄉鎮區再次賦予
      regionName.value = regionName.value[0].AreaList;
    });
};
</script>

<style>
.row {
  height: 100% !important;
  width: 100% !important;
}

#map {
  height: 100vh;
}

.overflow-scroll {
  overflow-x: hidden !important;
}
.option:hover {
  background-color: rgb(154, 255, 147);
  cursor: pointer;
}
.list {
  width: 350px;
}
@media (max-width: 768px) {
  .list {
    background-color: white;
    width: 250px;
    left: -250px;
    position: absolute;
    z-index: 10000;
  }
  .active {
    left: 0px !important;
  }

  .customBtn {
    z-index: 10000;
    position: absolute;
    height: 100px;
    width: 80px;
    font-size: 20px;
    top: 150px;
    border-radius: 0;
  }
  .btnActive {
    left: 248px !important;
  }
}
</style>
