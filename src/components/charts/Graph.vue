<template>
  <div class="highcharts-gauge">
    <div class="chart-legend text-center">
      <h3>{{ legend }}</h3>
    </div>
    <div :id="id"></div>
  </div>
</template>

<script>
import {graphMaker} from '@/helpers/charts';

export default {
  props: ['id', 'type', 'legend', 'title', 'subtitle', 'categories', 'data', 'yAxisText', 'xAxisText'],
  data() {
    return {
      graph: null,
    }
  },
  created() {
    this.$bus.$on('update-graph-' + this.id, data => {
      graphMaker.updatePoint(data);
    });
  },
  mounted() {
    graphMaker.create({
      id: this.id,
      type: this.type,
      data: this.data,
      yAxisText: this.yAxisText,
      xAxisText: this.xAxisText
    })
  }
}
</script>