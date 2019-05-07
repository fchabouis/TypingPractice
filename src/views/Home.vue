<template>
  <div>
    <!-- <img alt="Vue logo" src="../assets/logo.png"> -->
    <!-- <HelloWorld msg="Welcome to Your Vue.js App"/> -->
    <div id="targetText">{{targetText.substring(position)}}</div>
    <div>{{wpm}} wpm</div>
    <div>{{accuracy}} accuracy</div>
    <div>
      {{ statsPerKey }}
    </div>
    <div v-for="(v,k) in statsPerKey">
      {{ k }} : {{ v.sum / v.n }}
    </div>
  </div>
</template>

<script>
// @ is an alias to /src
// import HelloWorld from '@/components/HelloWorld.vue'
import axios from 'axios'

const average = list => list.reduce((prev, curr) => prev + curr) / list.length;
function randomChar(length) {
   let result           = '';
   let characters       = 'abcdefghijklmnopqrstuvwxyz';
   let charactersLength = characters.length;
   for ( let i = 0; i < length; i++ ) {
      result += characters.charAt(Math.floor(Math.random() * charactersLength));
   }
   return result;
}


export default {
  name: "home",
  components: {
    // HelloWorld
  },
  data() {
    return {
      targetText: "You have just taken your first step toward getting involved.",
      position: 0,
      time: 0,
      times: [],
      errorsN: 0
    };
  },
  mounted() {
    document.addEventListener("keypress", this.onPress);
    this.getWords()
  },
  methods: {
    onPress(event) {
      event.preventDefault();
      console.log(event.key)
      if (event.key === this.targetText[this.position]) {
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
        } else {
          if (event.key === 'Enter') {
            this.startNew()
          }
        }
      }
    },
    startNew() {
      this.position = 0
      this.time = 0
      this.times = []
      this.errorsN = 0
    },
    getWords(preferredKey) {
      let vm = this
      let sp = `*${randomChar(1)}*`
      axios.get(`https://api.datamuse.com/words?sp=${sp}&max=10`)
        .then(function (response) {
          vm.targetText = response.data.map(el => el.word).join(' ')
        })
        .catch(function (error) {
          vm.targetText = 'Oooop something is wrong with the service, sorry !'
        })
    }
  },
  computed: {
    testIsDone() {
      return this.position === this.targetText.length;
    },
    statsPerKey() {
      if (this.testIsDone) {
        let stats = this.times.reduce((acc, cv) => {
          let key = cv.key;
          if (key in acc) {
            acc[key].sum += cv.time;
            acc[key].n++
          } else {
            acc[key] = {sum: cv.time, n: 1};
          }
          return acc
        }, {});
        return stats
      } else {
        return {}
      }
    },
    justTimes() {
      return this.times.map(k => k.time);
    },
    wpm() {
      if (this.testIsDone) {
        return Math.round(60 / ((average(this.justTimes) * 5) / 1000));
      }
    },
    accuracy() {
      if (this.testIsDone) {
        return (1 - this.errorsN / this.targetText.length).toLocaleString(
          "en",
          { style: "percent" }
        );
      }
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
