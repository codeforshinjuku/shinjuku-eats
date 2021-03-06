<template>
  <div class="home">
    <div class="home__left" :class="$store.state.displayType === 'list' ? 'hide' : ''">
      <div class="page_heading">
        <main-banner-svg />
        <h1 class="heading">{{ headerTitle }}</h1>
      </div>
      <search-item v-if="isSearch" />
      <div class="inner">
        <div
          v-for="(shop, i) in filteredShops"
          :id="'shop' + i"
          :key="i"
          class="shop_wrapper"
          @click="handleShopPoint(shop.name)"
        >
          <shop-item v-bind="shop" />
        </div>
      </div>
    </div>
    <div v-if="$store.state.displayType === 'list'" class="home__right">
      <GmapMap
        v-if="currentLocation"
        ref="mapRef"
        :center="currentLocation"
        :zoom="14"
        :options="{ draggableCursor: 'default' }"
        style="width: 100%; height: 100%"
      >
        <GmapMarker
          v-for="(shop, index) in filteredShops"
          :key="index"
          :position="{
            lat: Number(shop.latitude),
            lng: Number(shop.longitude)
          }"
          :icon="shop.icon"
          :z-index="shop.zindex"
          :animation="shop.animation"
          @click="toggleInfoWindow(shop, index)"
        />
        <GmapInfoWindow
          :position="infoWindowPosition"
          :opened="infoWindowOpen"
          :options="{
            pixelOffset: {
              width: 0,
              height: -43 // ピンの高さ分
            }
          }"
          @closeclick="infoWindowOpen = false"
        >
          <router-link
            v-if="isMobile"
            :to="{ name: 'shop_show', params: { name: shopName } }"
            class="map_window_inner"
          >{{ shopName }}</router-link>
          <a v-else v-scroll-to="'#shop' + currentShopIndex" class="map_window_inner">{{ shopName }}</a>
        </GmapInfoWindow>
      </GmapMap>
      <div v-if="notMapData" class="not_map_pin">{{ notMapData }}</div>
    </div>
  </div>
</template>

<script>
import ShopItem from "../components/ShopItem";
import SearchItem from "../components/SearchItem";
import MainBannerSvg from "../components/MainBannerSvg";
const shuffle = ([...array]) => {
  for (let i = array.length - 1; i >= 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
};

const supportAreas = [
  "高田馬場・早稲田",
  "新宿・大久保",
  "落合",
  "四谷・市ヶ谷",
  "神楽坂"
];

export default {
  name: "Home",
  components: {
    MainBannerSvg,
    SearchItem,
    ShopItem
  },
  data: () => ({
    isMap: true,
    isSearch: false,
    currentLocation: null,
    shops: [],
    infoWindowPosition: {
      lat: 0,
      lng: 0
    },
    infoWindowOpen: false,
    shopName: "",
    currentShopIndex: "",
    isMobile: false,
    notMapData: false
  }),
  computed: {
    headerTitle: function() {
      if (!this.$route.params["name"]) return "新宿区";
      if (supportAreas.includes(this.$route.params["name"])) {
        return this.$route.params["name"];
      } else {
        return "";
      }
    },
    filteredShops: function() {
      const target = this.$route.params["name"];
      let shopsArray = [];
      if (target) {
        shopsArray = this.shops.filter(v => {
          return v.area === target;
        });
      } else {
        shopsArray = this.shops;
      }

      if (!this.$store.state.keyword) return shopsArray;
      return shopsArray.filter(v => {
        return v.name.indexOf(this.$store.state.keyword) > -1;
      });
    }
  },
  beforeCreate() {
    if (
      this.$route.params["name"] &&
      !supportAreas.includes(this.$route.params["name"])
    ) {
      this.$router.push({ name: "Home" });
    }
  },
  async created() {
    if (window.innerWidth <= 767) {
      this.isMap = false;
      this.isSearch = true;
      this.isMobile = true;
    } else {
      this.$store.dispatch("setDisplayType", "list");
    }
    const shops = await this.$http.get("data/shops.json");
    this.shops = shuffle(shops.data);
    const firstShop = this.filteredShops.find(v => v.latitude);
    this.currentLocation = {
      lat: firstShop.latitude,
      lng: firstShop.longitude
    };
  },
  methods: {
    toggleInfoWindow(shop, index) {
      this.shopName = shop.name;
      this.infoWindowPosition = {
        lat: Number(shop.latitude),
        lng: Number(shop.longitude)
      };

      if (this.currentShopIndex === index) {
        this.infoWindowOpen = !this.infoWindowOpen;
      } else {
        this.infoWindowOpen = true;
        this.currentShopIndex = index;
      }
    },
    handleShopPoint(name) {
      this.shops.map(function(v) {
        v.icon = null;
        v.zindex = null;
        v.animation = null;
      });
      let centerPoint = null;
      let notMapData = "";
      this.shops.map(function(shop) {
        if (shop.name === name) {
          if (shop.latitude && shop.longitude) {
            shop.icon = {
              url: "/images/active_pin.png",
              scaledSize: { width: 50, height: 50, f: "px", b: "px" }
            };
            shop.animation = 1;
            shop.zindex = 100;
            centerPoint = {
              lat: shop.latitude,
              lng: shop.longitude
            };
            notMapData = false;
          } else {
            shop.icon = null;
            shop.zindex = null;
            shop.animation = null;
            centerPoint = {
              lat: Number(shop.latitude),
              lng: Number(shop.longitude)
            };
            notMapData = shop.name + "の地図情報はありません";
          }
        }
      });
      this.currentLocation = centerPoint;
      this.notMapData = notMapData;
    }
  }
};
</script>
<style lang="scss">
@import "@/assets/scss/style.scss";
@include mediaMobile {
  .home {
    .search {
      margin: 0 0 3.3rem;
      &__inner {
        height: 5rem;
        &.input_active {
          display: none;
        }
      }
      &__input {
        position: static;
        width: 100%;
      }
    }
  }
}
</style>

<style scoped lang="scss">
@import "@/assets/scss/style.scss";
.home {
  display: flex;
  &__left,
  &__right {
    width: 50%;
  }
  &__left {
    height: 100vh;
    overflow-y: auto;
    .page_heading {
      position: relative;
      z-index: 15;
      width: 100%;
      margin: 0 auto 5.3rem;
      text-align: center;
      background: $white;
      .heading {
        font-size: 3.6rem;
        margin: 7.1rem 0 0;
      }
    }
    .inner {
      margin: 0 auto;
    }
    .shop_wrapper {
      cursor: pointer;
    }
  }
  &__right {
    position: relative;
    height: 100vh;
    padding: 5.1rem 0 0;
    .map_window_inner {
      color: $gray02;
    }
    .not_map_pin {
      position: absolute;
      top: 50%;
      left: 50%;
      padding: 2rem;
      transform: translate(-50%, -50%);
      animation-name: fadeOut;
      animation-duration: 5s;
      animation-fill-mode: forwards;
      border-radius: 0.4rem;
      background: $white;
    }
  }
  @keyframes fadeOut {
    0% {
      display: block;
      opacity: 100;
    }
    80% {
      opacity: 100;
    }
    100% {
      display: none;
      opacity: 0;
      z-index: -1;
    }
  }
  @include mediaMobile {
    flex-direction: column;
    &__left {
      width: 100%;
      order: 1;
      .page_heading {
        .heading {
          font-size: 2.8rem;
          margin: 2.1rem 0 0;
        }
      }
      .inner {
        width: 100%;
      }
      &.hide {
        display: none;
      }
    }
    &__right {
      width: 100%;
      order: 0;
      padding: 3.3rem 0 0;
    }
  }
  @include mediaTablet {
    &__left {
      .inner {
        width: 100%;
        padding: 0 1.5rem;
      }
    }
  }
  @include mediaPc {
    &__left {
      .inner {
        width: 62.4%;
      }
    }
  }
}
</style>
