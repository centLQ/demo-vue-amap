<template>
  <div :id="mapId" class="map-box" :style="style"></div>
</template>
<script>
import {getMapPolygon} from "@/api/mapService.js";
export default {
  name: "aMap",
  props: {
    mapId: {
      type: String,
      default: function () {
        return "map_" + new Date().getTime();
      },
    },
    options: {
      type: Object,
      default() {
        return {};
      },
    },
  },
  mounted() {
    this.$nextTick((_) => this.initMap());
  },
  computed: {
    center() {
      let { center = [115.988491,36.434645] } = this.options;
      return center;
    },
    style() {
      let { width = "500px", height = "500px" } = this.options;
      return `width: ${width};height: ${height};`;
    },
  },
  data() {
    return {};
  },
  methods: {
    initMap() {
      // 解决弹窗延时dom 没生成问题
      let that = this;
      // setTimeout(() => {
        that.map = new AMap.Map(that.mapId, {
          center: that.center,
          resizeEnable: true,
          zoom: this.options.zoom || 15,
          zoomEnable: true,
          dragEnable: true,
          mapStyle: "amap://styles/0d9a77cf9972aa45c81a1240a5e2cf1f",
          // mapStyle: "amap://styles/b8630c78f8f180aa0d2621b0dc5b61b7",
        });
        that.map.on("complete", function () {
          that.$emit("mapComplete", that.map);
          if (that.options.showPolygonBorder) that.renderPolygon();
        });
      // }, 500);
    },
    renderPolygon() {
      getMapPolygon().then((result) => {
         JSON.parse(result?.data?.areaRange)?.features.forEach(t => {
            let polygonPath = t.geometry.coordinates[0].map(t => new AMap.LngLat(t[0],t[1]));
            let polygon = new AMap.Polygon({
              path: polygonPath,
              strokeColor: "#ffffff",
              strokeWeight: 1,
              fillColor: "#0086ff",
              fillOpacity: 0.6,
            });
            this.map.add(polygon);
        });
    })
    },
    clearInfoWindow() {
      this.map.clearInfoWindow();
    },
    initMaker(position) {
      let that = this;
      window.$marker = new AMap.Marker({
        map: this.map,
        icon: "https://webapi.amap.com/theme/v1.3/markers/n/mark_b.png",
        position,
        draggable: true,
      });
      $marker.on("dragging", (e) => {
        that.$emit("markerPositionChange", e.lnglat);
      });
    },
    createIcon(icon) {
      let [w, h] = icon.size;
      return new AMap.Icon({
        // 图标尺寸
        size: new AMap.Size(w, h),
        // 图标的取图地址
        image: icon.url,
        // 图标所用图片大小
        imageSize: new AMap.Size(w, h),
        // 图标取图偏移量
        // imageOffset: new AMap.Pixel(-9, -3)
      });
    },
    initLabelMarkerEvent(marker, { click, mouseover, mouseout }) {
      click && marker.on("click", click);
      mouseover && marker.on("mouseover", mouseover);
      mouseout && marker.on("mouseout", mouseout);
    },
    getLabelsLayer() {
      return new AMap.LabelsLayer({
        visible: true,
        zIndex: 10000,
        // opacity: 1,
        zooms: [3, 20],
        // 开启标注避让，默认为开启，v1.4.15 新增属性
        collision: true,
        // 开启标注淡入动画，默认为开启，v1.4.15 新增属性
        animation: true,
      });
    },
    getLabelMarker(data) {
      return {};
    },
    getIcon(icon) {
      let [w, h] = icon.size;
      return new AMap.Icon({
        // 图标尺寸
        size: new AMap.Size(w, h),
        // 图标的取图地址
        image: icon.url, 
        // 图标所用图片大小
        imageSize: new AMap.Size(w, h),
        // 图标取图偏移量
        // imageOffset: new AMap.Pixel(-9, -3)
      });
    },
    /** 渲染Marker
     * // [{
        //   position: [curData.lon, curData.lat],
        //   name: "",
        //   icon: {
        //     image: curData.icon.url,
        //     size: curData.icon.size
        //   }
        // }]
     */
    initLabelMarker(data, layer, eventCallback) {
      layer = layer || this.getLabelsLayer();
      this.map.add(layer);
      let markers = [];
      let labelMarker = null;
      let curData = null;
      for (var i = 0; i < data.length; i++) {
        curData = this.G_LODASH.cloneDeep(data[i]);
        labelMarker = new AMap.LabelMarker(curData);
        labelMarker.setExtData(curData); // 自定义数据 通过 getExtData 获取
        eventCallback && this.initLabelMarkerEvent(labelMarker, eventCallback); // 注册marker鼠标事件
        markers.push(labelMarker);
      }
      layer.add(markers);
      this.map.setFitView(markers);
      return markers;
    },
    openInfoWindow(
      content,
      { lon, lat, longitude, latitude } = position,
      offset,
      eventCallBack = {},
      options = {}
    ) {
      let opt = {
        ...options,
      };
      offset && (opt.offset = new AMap.Pixel(offset[0], offset[1])); // 设置窗口偏移量
      let p = new AMap.LngLat(lon || longitude, lat || latitude); // 地图弹窗位置
      var infoWindow = new AMap.InfoWindow(opt);
      infoWindow.setContent(content); //地图弹窗内容
      infoWindow.open(this.map, p);
      let { open, close } = eventCallBack || {};
      open && infoWindow.on("open", open); // 注册窗口打开事件
      close && infoWindow.on("close", close); // 注册窗口关闭事件
      return infoWindow;
    },
    closeInfoWindow(infoWindow, position = null) {
      infoWindow.close(this.map, position);
    },
    renderCircle({ lat, lon } = position, r, style = {}) {
      return new AMap.Circle({
        map: this.map,
        center: new AMap.LngLat(lon, lat),
        radius: r,
        strokeColor: "#F33",
        strokeOpacity: 1,
        strokeWeight: 3,
        fillColor: "ee2200",
        fillOpacity: 0.35,
        ...style,
      });
    },
    circleEditor(position, { move, adjust, end } = event, r = 500) {
      var circle = this.renderCircle(position, r, {});
      this.map.setFitView([circle]);
      let circleEditor = null;
      this.map.plugin(["AMap.CircleEditor"], () => {
        circleEditor = new AMap.CircleEditor(this.map, circle);
        circleEditor.open();
        move && circleEditor.on("move", move);
        adjust && circleEditor.on("adjust", adjust);
        end && circleEditor.on("end", end);
      });
      // 清空相关事件绑定和数据
      let clear = () => {
        move && circleEditor.off("move", move);
        adjust && circleEditor.off("adjust", adjust);
        end && circleEditor.off("end", end);
        circleEditor.close();
        this.map.remove(circle);
        circleEditor = null;
        circle = null;
      };
      // 关闭编辑
      clear.closeEdit = function () {
        circleEditor.close();
      };
      return clear;
    },
  },
};
</script>
<style lang="less">
.map-box {
  .amap-info-content {
    padding: 0;
    border-radius: 10px;
  }
  .amap-info-sharp {
    bottom: 1px;
    border-top: 8px solid #050e19;
  }
  .amap-info-content {
    box-shadow: 0px 0px 20px 0px rgba(0, 0, 0, 0.54);
  }
}
</style>