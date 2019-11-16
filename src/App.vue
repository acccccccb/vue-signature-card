<template>
  <div id="app">
    <VueSignatureCard
            ref="sign"
            color="red"
            :disabled="false"
            :lineWidth="5"
            :width="200"
            :height="200"
            @drawing="drawing"
            @success="signSuccess">
    </VueSignatureCard>

    <VueSignatureCard
            :disabled="true"
            ref="signShow"
            color="green"
            :showButton="false"
            :lineWidth="3"
            :width="200"
            :height="200"
            @success="signSuccess">
    </VueSignatureCard>

  </div>
</template>
<script>
import VueSignatureCard from './components/VueSignatureCard'
export default {
  name: 'App',
  components: {
      VueSignatureCard
  },
    methods:{
        signSuccess:function(res){
            this.$refs.signShow.clearMap(true);
            this.$refs.signShow.showHistory(res.history,0);
        },
        drawing:function(res){
            this.$refs.signShow.showHistoryQuick(res);
        }
    }
}
</script>

<style scoped>
  #app {
    width:800px;
    margin:auto;
  }
</style>
