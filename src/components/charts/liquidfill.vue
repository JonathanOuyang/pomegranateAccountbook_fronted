<!--  -->
<template>
  <div :id="elId"
       class="chart-liquidfill"
       :style="{width: radius+'px', height: radius+'px'}"></div>
</template>

<script>
import echarts from "echarts/lib/echarts";
import echartsLiquidfill from "echarts-liquidfill";
//引入uuid文件
import uuid from "uuid/v1";

export default {
  name: "chart-liquidfill",
  props: {
    data: {
      type: Number,
      default: 0
      // required: true
    },
    radius: {
      type: Number,
      default: 60
    }
  },
  data() {
    return {
      elId: "",
      chartObj: {}
    };
  },
  created() {
    this.elId = uuid(); //获取随机id
  },
  mounted() {
    this.$nextTick(() => {
      this.chartObj = echarts.init(document.getElementById(this.elId));
      this.render();
    });
  },
  computed: {},

  watch: {
    data: {
      deep: true,
      handler: function() {
        this.render();
      }
    }
  },

  methods: {
    render() {
      const option = {
        series: [
          {
            type: "liquidFill",
            data: [this.data],
            radius: '100%',
            amplitude: '3%',
            waveLength: '120%',
            outline: {
              show: false
            },
            backgroundStyle: {
              color: 'rgba(246, 113, 125, 0.1)'
            },
            itemStyle: {
              color: 'rgb(246, 113, 125)'              
            },
            label: {
              fontSize: this.radius/4,
              color: 'rgb(246, 113, 125)'  
            }
          }
        ]
      };
      this.chartObj.setOption(option);
    }
  }
};
</script>

<style lang='less' scoped>
@import "../../assets/variable.less";
.chart-liquidfill {}
</style>