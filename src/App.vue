  <script >
  import LineChart from './components/LineChart.vue'

  export default {
    components: {
      LineChart,
    },
    data() {
      return {
        customData: [
          { x: 1, y: 50 },
          { x: 2, y: 80 },
          { x: 3, y: 25 },
          { x: 6, y: 1000 },
        ],
      };
    },
  };
  </script>

  <template>
    <div class="chart-container">
      <LineChart 
        :width="800" 
        :height="400" 
        :margin="{ top: 30, right: 30, bottom: 50, left: 60 }" 
        :series="[
          { data: Array.from({ length: 10 }, (_, i) => ({ x: i + 1, y: Math.random() * 100 })), color: 'red', width: 3 },
          { data: Array.from({ length: 10 }, (_, i) => ({ x: i + 1, y: Math.random() * 100 })), color: 'green', width: 2 },
        ]"
        xLabel="Tempo"
        yLabel="Valor"
        :hasGradient="false"
        curveType="cardinal"
        :enableCrosshair="true"
        crosshairType="cross"
        axisColor="#000"
        :grid="false"
        gridColor="#e2e"
        :enablePoint="true"
        :pointRadius="5"
        pointColor="#fff"
        pointBorderColor="#000"
        :pointBorderWidth="1"
        :enableArea="true"
        :areaOpacity="0.2"
        crosshairColor="#000"
        :crossHairDashArray="5"
    />
    </div>
  </template>

  <style scoped>
    header {
      line-height: 1.5;
    }

  .logo {
    display: block;
    margin: 0 auto 2rem;
  }

  .chart-container {
    background-color: white;
    border-radius: 10px;
  }

  @media (min-width: 1024px) {
    header {
      display: flex;
      place-items: center;
      padding-right: calc(var(--section-gap) / 2);
    }

    .logo {
      margin: 0 2rem 0 0;
    }

    header .wrapper {
      display: flex;
      place-items: flex-start;
      flex-wrap: wrap;
    }
  }
  </style>
