<template>
  <div class="map-container">
    <div class="map-colors-info">
      <div class="map-color map-color-available"><span></span> Available</div>
      <div class="map-color map-color-conditional"><span></span> Pending</div>
      <div class="map-color map-color-sold"><span></span> Sold</div>
    </div>
    <!-- <a
      href="javascript:;"
      class="map-filters-button"
      @click="map_filters_visible = true"
      v-if="map_filters_visible === false"
      ><span v-if="loading">Loading...</span
      ><span v-if="loading === false">Map Filters</span></a
    >
    <div class="menu" v-if="map_filters_visible">
      <a
        href="javascript:;"
        style="text-align: center;"
        @click="map_filters_visible = false"
        >Close</a
      >
      <a href="javascript:;" @click="loadData('catholic_schools')"
        >Catholic Schools</a
      >
      <a href="javascript:;" @click="loadData('public_schools')"
        >Public Schools</a
      >
      <a href="javascript:;" @click="loadData('arenas')">Arenas</a>
      <a href="javascript:;" @click="loadData('recreation_centres')"
        >Recreation Centres</a
      >
    </div> -->
    <div class="quick-filters-button">
      <a-button
        :type="isFourPlusBeds ? 'primary' : 'default'"
        @click="clickFourPlusBeds"
        class="customBtn"
        >4+ Beds</a-button
      >
      <a-button
        :type="isUnder400 ? 'primary' : 'default'"
        @click="clickUnder400"
        class="customBtn"
        >Homes under 400K</a-button
      >
      <a-button
        :type="isUnder500 ? 'primary' : 'default'"
        @click="clickUnder500"
        class="customBtn"
        >Homes under 500K</a-button
      >
      <a-button
        :type="isLegalSuite ? 'primary' : 'default'"
        @click="clickLegalSuite"
        class="customBtn"
        >Homes with Legal Suite</a-button
      >
    </div>
    <div id="map" style="width: 100%; height: 100%;"></div>
  </div>
</template>

<script>
import axios from "axios";
import MarkerClusterer from "@googlemaps/markerclustererplus";
import MarkerWithLabel from "../markerwithlabel";
import { bus } from "../app.js";

export default {
  props: {
    type: {
      type: String,
      default: "home",
    },
  },
  data() {
    return {
      isFourPlusBeds: false,
      isUnder400: false,
      isUnder500: false,
      isLegalSuite: false,
      topMenuHeight: 0,
      initialMarginSize: 0,
      loading: false,
      map_filters_visible: false,
      info_window: null,
      map: null,
      marker_clusterer: null,
      markers: [],
      items: [],
      cached_marker_items: [],
      scrollIntance: null,
      resizeInstance: null,
      filter: {},
    };
  },
  mounted() {
    this.setUpMap();
  },
  created() {
    // this.tabs = await new appConnections().data;
    // let $this = this;
    bus.$on("openMarker", (item) => {
      console.log("markers", this.markers);
      var filterMarkers = this.markers.filter((e) => e.itemId == item.id);
      var selectedMarker = filterMarkers[0];
      console.log("openMarker", selectedMarker);
      google.maps.event.trigger(selectedMarker, "click");
      // google.maps.event.addListener(selectedMarker, "click", function() {
      //   // do something with this marker ...
      //   console.log("$this-", $this, $this.map);
      //   $this.info_window.open($this.map, selectedMarker);
      // });
      // selectedMarker.addListener("click", function() {
      //   // define actions to occur when specified marker is clicked
      //   console.log("Specified marker clicked!");
      // });
    });
  },
  methods: {
    clickFourPlusBeds() {
      this.isFourPlusBeds = !this.isFourPlusBeds;
      this.quickFilter();
    },
    clickUnder400() {
      this.isUnder400 = !this.isUnder400;
      this.quickFilter();
    },
    clickUnder500() {
      this.isUnder500 = !this.isUnder500;
      this.quickFilter();
    },
    clickLegalSuite() {
      this.isLegalSuite = !this.isLegalSuite;
      this.quickFilter();
    },
    quickFilter() {
      console.log(
        "quickfilter--",
        this.isFourPlusBeds,
        this.isUnder400,
        this.isUnder500,
        this.isLegalSuite
      );
      this.$router.push({});
      this.$router.push({
        query: Object.assign({}, this.$route.query, {
          isFourPlusBeds: this.isFourPlusBeds,
          isUnder400: this.isUnder400,
          isUnder500: this.isUnder500,
          isLegalSuite: this.isLegalSuite,
        }),
      });
      this.filter = this.$store.state.filter;
      this.filter.isFourPlusBeds = this.isFourPlusBeds;
      this.filter.isUnder400 = this.isUnder400;
      this.filter.isUnder500 = this.isUnder500;
      this.filter.isLegalSuite = this.isLegalSuite;
      console.log("dd", this.filter);
      // this.filter = {
      //   key: null,
      //   type: null,
      //   communities: [],
      //   price_range: [],
      //   released_market: [],
      //   beds: 0,
      //   baths: 0,
      //   possession_date: null,
      //   home_size_range: [],
      //   home_types: [],
      //   model_ids: [],
      //   features: [],
      //   front_exposure: [],
      //   garage_parking_spaces: [],
      //   lot_size: 0,
      //   lot_shape: "",
      //   lot_location: "",
      //   walkout_lot: "",
      //   colourboard: "",
      //   is_price_reduced: false,
      //   is_promotion: false,
      //   is_guaranted: 0,
      //   isUnder400: this.isUnder400,
      //   isUnder500: this.isUnder500,
      //   isLegalSuite: this.isLegalSuite,
      // };
      this.$store.dispatch("saveFilter", {
        routeName: this.$route.name,
        filter: this.filter,
      });
      this.$emit("onRefresh");
    },
    setUpMap() {
      let $this = this;
      let origin = new google.maps.LatLng(
        53.54067477803275,
        -113.49848717382815
      );
      this.map = new google.maps.Map(document.getElementById("map"), {
        zoom: 9,
        center: origin,
        fullscreenControl: false,
        mapTypeControl: false,
        streetViewControl: false,
        styles: [
          {
            stylers: [
              {
                saturation: -100,
              },
            ],
          },
        ],
      });
      this.info_window = new google.maps.InfoWindow({
        content: "",
      });
      this.marker_clusterer = new MarkerClusterer(this.map, [], {
        imagePath:
          "https://developers.google.com/maps/documentation/javascript/examples/markerclusterer/m",
        maxZoom: 11,
      });
      this.map.addListener("bounds_changed", function(event) {
        let bounds = $this.map.getBounds();
        let itemIds = [];
        for (let i = 0; i < $this.markers.length; i++) {
          if (bounds.contains($this.markers[i].getPosition())) {
            itemIds.push($this.markers[i].itemId);
          }
        }
        console.log("itemsids--", itemIds);
        $this.$emit("onVisibleItems", itemIds);
      });
    },
    loadData(type) {
      let $this = this;
      this.loading = true;
      this.map_filters_visible = false;
      this.items.forEach((item) => {
        item.setMap(null);
      });
      this.items = [];
      axios
        .get("/wp-json/templatev2/v1/map-data", { params: { type: type } })
        .then((response) => {
          response.data.forEach((item) => {
            console.log("item---", item);
            this.loading = false;
            let latLng = new google.maps.LatLng(item.lat, item.lng);
            let marker = new google.maps.Marker({
              position: latLng,
              title: item.name,
              icon:
                "http://maps.google.com/mapfiles/kml/shapes/placemark_circle.png",
            });
            marker.addListener("click", () => {
              let content = "<h4>" + item.name + "</h4>";
              if (item.hasOwnProperty("grade_level")) {
                content += item.grade_level + "<br />";
              }
              if (item.hasOwnProperty("street")) {
                content += item.street;
              }
              $this.info_window.setContent(content);
              $this.info_window.open($this.map, marker);
            });
            $this.items.push(marker);
            $this.refreshMarkers();
          });
        });
    },
    refreshMarkers(items) {
      if (this.type === "home") {
        this.refreshHomeMarkers(items);
      } else {
        this.refreshCommunityMarkers(items);
      }
    },
    refreshHomeMarkers(items) {
      console.log("refreshhomemarkers-", items);
      let $this = this;
      if (items == null) {
        items = this.cached_marker_items;
      } else {
        this.cached_marker_items = items;
      }
      this.marker_clusterer.clearMarkers();
      this.markers = [];
      let bounds = new google.maps.LatLngBounds();
      items.forEach((item) => {
        if (item.lat && item.lng) {
          let content = `
                            <h4>${item.title}</h4>
                            <a href="${
                              item.link
                            }" style="padding: 5px; cursor: pointer; display: block">
                                <img src="${
                                  item.medium_image
                                }" style="height: 60px; display: block" />
                                ${item.sq_ft} sqft, ${item.bedrooms} Beds, ${
            item.bathrooms
          } Baths <br/>
                                ${item.street}, ${item.community_name}
                            </a>
                        `;
          let latLng = new google.maps.LatLng(item.lat, item.lng);
          let color = "#52b788";
          if (item.offer_status === "Sold") {
            color = "#FC384A";
          } else if (item.offer_status === "Conditional") {
            color = "#ffba08";
          }
          let marker = new MarkerWithLabel({
            position: latLng,
            title: item.title,
            icon: {
              path: google.maps.SymbolPath.CIRCLE,
              scale: 7,
              fillColor: color,
              fillOpacity: 0.8,
              strokeColor: "#fff",
              strokeWeight: 1.8,
            },
            itemId: item.id,
          });
          marker.addListener("click", () => {
            $this.info_window.setContent(content);
            $this.info_window.open($this.map, marker);
          });
          marker.addListener("mouseover", () => {
            $this.info_window.setContent(content);
            $this.info_window.open($this.map, marker);
            marker.setIcon({
              path: google.maps.SymbolPath.CIRCLE,
              scale: 7,
              fillColor: color,
              fillOpacity: 0.5,
              strokeColor: "#fff",
              strokeWeight: 1.8,
            });
          });
          marker.addListener("mouseout", () => {
            marker.setIcon({
              path: google.maps.SymbolPath.CIRCLE,
              scale: 7,
              fillColor: color,
              fillOpacity: 0.8,
              strokeColor: "#fff",
              strokeWeight: 1.8,
            });
          });
          bounds.extend(latLng);
          $this.markers.push(marker);
        }
      });
      this.marker_clusterer.addMarkers(this.markers);
      if (this.markers.length > 0) {
        this.map.fitBounds(bounds);
      }
    },
    refreshCommunityMarkers(items) {
      let $this = this;
      if (items == null) {
        items = this.cached_marker_items;
      } else {
        this.cached_marker_items = items;
      }
      this.markers.forEach((marker) => {
        marker.setMap(null);
      });
      this.markers = [];
      let bounds = new google.maps.LatLngBounds();
      items.forEach((item) => {
        if (item.lat && item.lng) {
          let latLng = new google.maps.LatLng(item.lat, item.lng);
          let marker = new google.maps.Marker({
            position: latLng,
            title: item.name,
          });
          marker.setMap($this.map);
          marker.addListener("click", () => {
            let content =
              "<h4>" +
              item.name +
              "</h4>" +
              '<div style="margin-bottom: 10px;cursor: pointer"><a href="/communities/#/communities/' +
              item.area +
              "/" +
              item.id +
              '"><img src="' +
              item.image +
              '" style="width: 100px;" /></a></div>';
            $this.info_window.setContent(content);
            $this.info_window.open($this.map, marker);
          });
          bounds.extend(latLng);
          $this.markers.push(marker);
        }
      });
      if (this.markers.length > 0) {
        if (this.markers.length === 1) {
          this.map.setCenter(bounds.getCenter());
          this.map.setZoom(13);
        } else {
          this.map.fitBounds(bounds);
        }
      }
    },
  },
};
</script>

<style lang="scss" scoped>
@import "../app.scss";

.property-price {
  color: #00ff00;
}
.customBtn {
  border: 2px solid #40a9ff;
  color: #40a9ff;
}
.customBtn:hover {
  border: 2px solid #40a9ff;
  color: white;
  background: #40a9ff;
}
.customBtn.ant-btn-primary {
  color: white;
}

@media all and (max-width: 1050px) {
  .main-container {
    .map-container {
      position: fixed !important;
      left: 0 !important;
      right: 0 !important;
      top: 180px !important;
      bottom: 0 !important;
    }
  }
}
</style>
