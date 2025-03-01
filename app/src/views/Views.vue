<template>
  <v-container>
    <BindTitle value="View history" />
    <div v-if="mostRecent">
      <Divider icon="mdi-monitor-cellphone">Most recently watched</Divider>
      <div style="max-width: 400px" class="mb-3 mx-auto d-flex align-center">
        <v-btn
          :loading="loadingItem"
          large
          icon
          outlined
          :disabled="recentIndex <= 0"
          @click="recentIndex--"
        >
          <v-icon>mdi-chevron-left</v-icon>
        </v-btn>
        <v-spacer></v-spacer>
        <div class="text-center">
          <div class="med--text subtitle-1">{{ new Date(mostRecent.date).toLocaleString() }}</div>
        </div>
        <v-spacer></v-spacer>
        <v-btn
          :loading="loadingItem"
          large
          icon
          outlined
          :disabled="recentIndex >= views.length - 1"
          @click="recentIndex++"
        >
          <v-icon>mdi-chevron-right</v-icon>
        </v-btn>
      </div>
      <div class="d-flex justify-center">
        <SceneCard style="width: 400px" v-model="mostRecent.scene" />
      </div>
    </div>

    <Divider icon="mdi-calendar">All time</Divider>

    <div class="text-center">
      <span class="headline mr-1">{{ views.length }}</span>
      <span class="subtitle-1">views</span>
    </div>

    <v-row>
      <v-col sm="12" md="12">
        <canvas ref="canvas"></canvas>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import { Component, Vue, Watch } from "vue-property-decorator";
import ApolloClient from "@/apollo";
import gql from "graphql-tag";
import SceneCard from "@/components/SceneCard.vue";
import sceneFragment from "@/fragments/scene";
import actorFragment from "@/fragments/actor";
import studioFragment from "@/fragments/studio";
import Chart from "chart.js";

@Component({
  components: { SceneCard },
})
export default class About extends Vue {
  showLog = false;
  showHttp = false;
  showWarn = true;

  views = [] as {
    scene: { _id: string; name: string };
    date: number;
  }[];

  recentIndex = -1;
  mostRecent = null as any;
  loadingItem = false;

  chart: Chart | null = null;

  @Watch("recentIndex")
  onIndexChange(val: number) {
    const time = this.views[val].date;
    this.loadingItem = true;
    this.getMostRecent(time)
      .catch((err) => {
        console.error(err.message);
      })
      .finally(() => {
        this.loadingItem = false;
      });
  }

  async getMostRecent(time: number) {
    const res = await ApolloClient.query({
      query: gql`
        query($min: Long, $max: Long) {
          getWatches(min: $min, max: $max) {
            date
            scene {
              ...SceneFragment
              actors {
                ...ActorFragment
              }
              studio {
                ...StudioFragment
              }
            }
          }
        }
        ${sceneFragment}
        ${actorFragment}
        ${studioFragment}
      `,
      variables: {
        min: time,
        max: time,
      },
    });
    this.mostRecent = res.data.getWatches[0];
  }

  async fetchAll(min?: number, max?: number) {
    const res = await ApolloClient.query({
      query: gql`
        query($min: Long, $max: Long) {
          getWatches(min: $min, max: $max) {
            date
            scene {
              _id
              name
            }
          }
        }
      `,
      variables: { min, max },
    });
    return res.data.getWatches;
  }

  get chartData() {
    return {
      labels: this.views.map((i) => new Date(i.date)),
      datasets: [
        {
          label: "Views this month",
          data: this.views.map((view, index) => ({
            t: new Date(view.date),
            y: index + 1,
          })),
          pointBackgroundColor: "#90aaff",
          fill: true,
          backgroundColor: "#90aaff22",
        },
      ],
    };
  }

  initChart() {
    const canvas = this.$refs.canvas as HTMLCanvasElement | null;
    if (!canvas) {
      console.error("Could not initialize views chart: canvas not found");
      return;
    }

    this.chart = new Chart(canvas, {
      type: "line",
      data: this.chartData,
      options: {
        tooltips: {
          position: "nearest",
          callbacks: {
            label: (tooltipItem, data) => {
              const index = tooltipItem.index;
              if (index === undefined) return "";
              return this.views[index].scene.name;
            },
          },
        },
        legend: {
          display: false,
        },
        //showLines: false,
        responsive: true,
        scales: {
          xAxes: [
            {
              type: "time",
              distribution: "linear",
            },
          ],
          yAxes: [
            {
              display: true,
              ticks: {
                suggestedMax: 5,
                beginAtZero: true,
                stepSize: 1,
              },
            },
          ],
        },
      },
    });
  }

  destroyChart() {
    this.chart?.destroy();
  }

  async getStats() {
    try {
      const items = await this.fetchAll();

      this.views = items;
      this.recentIndex = this.views.length - 1;

      this.initChart();
    } catch (err) {
      console.error("Could not retrieve views: ", err.message);
    }
  }

  mounted() {
    this.getStats();
  }

  beforeDestroy() {
    this.destroyChart();
  }
}
</script>
