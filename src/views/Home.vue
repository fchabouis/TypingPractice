<template>
  <div>
    <!-- <img alt="Vue logo" src="../assets/logo.png"> -->
    <!-- <HelloWorld msg="Welcome to Your Vue.js App"/> -->
    <div id="targetText">{{targetText.substring(position)}}</div>
    <div>{{wpm}} wpm</div>
    <div>{{accuracy}} accuracy</div>
    <div>
      {{worstAccuracy}}
    </div>
    <div>
      <Highcharts :options="chartOptions"/>
    </div>
  </div>
</template>

<script>
import axios from "axios";

let ls = window.localStorage;

const average = list => list.reduce((prev, curr) => prev + curr) / list.length;

const timeToWpm = time => Math.round(60. / ((time * 5) / 1000))

function randomChar(length) {
  let result = "";
  let characters = "abcdefghijklmnopqrstuvwxyz";
  let charactersLength = characters.length;
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
  }
  return result;
}

function sumObjectsByKey(...objs) {
  return objs.reduce((a, b) => {
    for (let k in b) {
      if (b.hasOwnProperty(k)) a[k] = (a[k] || 0) + b[k];
    }
    return a;
  }, {});
}

export default {
  name: "home",
  components: {
    // HelloWorld
  },
  data() {
    return {
      targetText: "",
      position: 0,
      time: 0,
      times: [],
      errorsN: 0,
      errors: {},
      hasStarted: false,
      typingStats: { speed: {}, errors: {}, count: {} }
    };
  },
  watch: {
    testIsDone(val) {
      if (val) {
        this.addTheStats();
      }
    }
  },
  mounted() {
    document.addEventListener("keypress", this.onPress);
    if (ls.getItem("typingStats")) {
      this.typingStats = JSON.parse(ls.getItem("typingStats"));
    }
    this.getWords();
  },
  methods: {
    onPress(event) {
      event.preventDefault();
      let targetKey = this.targetText[this.position];
      if (event.key === targetKey) {
        this.hasStarted = true;
        if (this.time) {
          let date = new Date();
          this.times.push({ key: event.key, time: date - this.time });
          this.time = date;
        } else {
          this.time = new Date();
        }
        this.position++;
      } else {
        if (!this.testIsDone) {
          this.errorsN++;
          if (targetKey in this.errors) {
            this.errors[targetKey]++;
          } else {
            this.errors[targetKey] = 1;
          }
        } else {
          if (event.key === "Enter") {
            this.startNew();
          }
        }
      }
    },
    startNew() {
      this.position = 0;
      this.time = 0;
      this.times = [];
      this.errorsN = 0;
      this.errors = {};
      this.getWords();
    },
    getWords(preferredKey) {
      let vm = this;
      let sp = `*${this.worstAccuracy}*`;
      axios
        .get(`https://api.datamuse.com/words?sp=${sp}&max=300`)
        .then(function(response) {
          const shuffled = response.data.map(el => el.word).sort(() => 0.5 - Math.random());
          let selected = shuffled.slice(0, 10);
          vm.targetText = selected.join(" ");
        })
        .catch(function(error) {
          vm.targetText = "Oooop something is wrong with the service, sorry !";
        });
    },
    addTheStats() {
      this.typingStats.speed = sumObjectsByKey(
        this.typingStats.speed,
        this.keysSpeed
      );
      this.typingStats.errors = sumObjectsByKey(
        this.typingStats.errors,
        this.errors
      );
      this.typingStats.count = sumObjectsByKey(
        this.typingStats.count,
        this.keysN
      );
      ls.setItem("typingStats", JSON.stringify(this.typingStats));
    }
  },
  computed: {
    worstAccuracy() {
      let minAccuracy = 1.
      let badKey = ''
      for (let key of Object.keys(this.typingStats.count)) {
        let accuracy = 1. - (this.typingStats.errors[key] || 0) / this.typingStats.count[key]
        console.log(key)
        if (accuracy < minAccuracy) {
          minAccuracy = accuracy
          badKey = key
        }
      }
      return badKey
    },
    testIsDone() {
      return this.position === this.targetText.length && this.hasStarted;
    },
    keysSpeed() {
      if (this.testIsDone) {
        let speedStats = this.times.reduce((acc, cv) => {
          let key = cv.key;
          if (key in acc) {
            acc[key] += cv.time;
          } else {
            acc[key] = 0;
          }
          return acc;
        }, {});
        return speedStats;
      } else {
        return {};
      }
    },
    keysN() {
      if (this.testIsDone) {
        let countStats = this.times.reduce((acc, cv) => {
          let key = cv.key;
          if (key in acc) {
            acc[key]++;
          } else {
            acc[key] = 0;
          }
          return acc;
        }, {});
        return countStats;
      } else {
        return {};
      }
    },
    seriesForScatterPlot() {
      return Object.keys(this.typingStats.count)
      .map(key => {
        return {
          name: key,
          color: "red",
          data: [
            [
              // 1,1
              timeToWpm(this.typingStats.speed[key] / this.typingStats.count[key]),
              1. - (this.typingStats.errors[key] || 0.) / this.typingStats.count[key]
            ]
          ]
        };
      });

    },
    justTimes() {
      return this.times.map(k => k.time);
    },
    wpm() {
      if (this.testIsDone) {
        return timeToWpm(average(this.justTimes))
      }
    },
    accuracy() {
      if (this.testIsDone) {
        return (1 - this.errorsN / this.targetText.length).toLocaleString(
          "en",
          { style: "percent" }
        );
      }
    },
    chartOptions() {
      return {
        chart: {
          type: "scatter",
          zoomType: "xy",
          animation: {
            duration: 2000
          }
        },
        title: {
          text: "speed / accuracy per key"
        },
        xAxis: {
          title: {
            enabled: true,
            text: "speed (wpm)"
          },
          startOnTick: true,
          endOnTick: true,
          showLastLabel: true
        },
        yAxis: {
          title: {
            text: "accuracy"
          }
        },
        plotOptions: {
          scatter: {
            marker: {
              radius: 5,
              symbol: 'circle',
              states: {
                hover: {
                  enabled: true,
                  lineColor: "rgb(100,100,100)"
                }
              }
            },
            states: {
              hover: {
                marker: {
                  enabled: false
                }
              }
            },
            tooltip: {
              headerFormat: "<b>{series.name}</b><br>",
              pointFormat: "{point.x} wpm, {point.y} accuracy"
            }
          }
        },
        series: this.seriesForScatterPlot
      };
    }
  }
};
</script>

<style scoped lang="scss">
#targetText {
  text-align: center;
  font-size: 40px;
}
</style>
