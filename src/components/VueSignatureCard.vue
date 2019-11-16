<template>
  <div>
    <div ref="scrollRef">
      <div ref="signBox">
        <div class="canvasBox" :style="'width:'+ width +'px;height:'+height+'px;'">
          <canvas
                  :disabled="true"
                  class="canvas"
                  :width="canvasWidth"
                  :height="canvasHeight"
                  @mousedown="handleMouseDown"
                  @mouseup="handleMouseUp"
                  @mouseleave="handleMouseUp"
                  @mousemove="handleMouseMove"
                  @touchstart="handleTouchStart"
                  @touchmove="handleTouchMove"
                  ref="canvas">您当前浏览器不支持canvas，建议更换浏览器！</canvas>
        </div>
        <button :disabled="btnDisable" @click="clearMap(true)" type="button" plain >重新签</button>
        <button :disabled="btnDisable || pathHistory.length<10" type="button" @click="scrollBack">撤回</button>
        <button :disabled="btnDisable || pathHistory.length<10" type="button" @click="saveMap">提交签名</button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'VueSignatureCard',
    props:{
        width:{
            required:false,
            type:Number,
            default:200,
        },
        height:{
            required:false,
            type:Number,
            default:200,
        },

        history:{
            required:false,
            type:String
        }
    },
    data(){
        return {
            readonly:true,
            btnDisable:false,
            timmer:null,
            canvasWidth:0,
            canvasHeight:0,
            canvas:null,
            ctx:null,
            mouseDown:false,
            map:{
                startX:0,
                startY:0,
                offsetTop:0,// y轴偏移
                offsetLeft:0,// x轴偏移
            },
            pathHistory:[]
        }
    },
    created(){
        this.canvasWidth = this.width;
        this.canvasHeight = this.height;
    },
    mounted(){
        let canvas = this.$refs['canvas'];
        let ctx=canvas.getContext("2d");
        this.canvas = canvas;
        this.ctx = ctx;
        this.map.offsetTop = canvas.getBoundingClientRect().top;
        this.map.offsetLeft = canvas.getBoundingClientRect().left;
        // 抗锯齿
        let width = canvas.width;
        let height = canvas.height;
        if (window.devicePixelRatio) {
            this.canvas.style.width = width + "px";
            this.canvas.style.height = height + "px";
            this.canvas.height = height * window.devicePixelRatio;
            this.canvas.width = width * window.devicePixelRatio;
            ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
        }
        if(this.history) {
            this.showHistory(this.history);
        }
    },
    methods:{
        // 手机
        handleTouchStart:function(e){
            if(this.readonly==false) {return false;}
            this.map.startX = e.changedTouches[0].clientX-this.map.offsetLeft;
            this.map.startY = e.changedTouches[0].clientY-this.map.offsetTop;
            if(this.pathHistory.length>0) {
                this.pathHistory.push({
                    x:-1,
                    y:-1
                });
            }
            this.pathHistory.push({
                x:e.changedTouches[0].clientX-this.map.offsetLeft,
                y:e.changedTouches[0].clientY-this.map.offsetTop
            });
        },
        handleTouchMove:function(e){
            if(this.readonly==false) {return false;}
            let x = e.changedTouches[0].clientX-this.map.offsetLeft;
            let y = e.changedTouches[0].clientY-this.map.offsetTop;
            this.drowToMap(x,y);
        },
        clearMap:function(action){
            this.pathHistory = [];
            this.canvas.width = this.canvasWidth;
            this.canvas.height = this.canvasHeight;
            this.map.startX = 0;
            this.map.startY = 0;
            clearInterval(this.timmer);
            this.timmer = null;
            if(action===true) {
                this.readonly = true;
            }
        },
        // pc
        handleMouseDown:function(e){
            if(this.readonly==false) {return false;}
            this.map.startX = e.offsetX;
            this.map.startY = e.offsetY;
            if(this.pathHistory.length>0) {
                this.pathHistory.push({
                    x:-1,
                    y:-1
                });
            }
            this.pathHistory.push({
                x:e.offsetX,
                y:e.offsetY
            });
            this.mouseDown = true;
        },
        handleMouseUp:function(e){
            if(this.readonly==false) {return false;}
            this.mouseDown = false;
            this.map.startX = e.offsetX;
            this.map.startY = e.offsetY;
            this.pathHistory.push(-1,-1);
        },
        handleMouseMove:function(e){
            if(this.readonly==false) {return false;}
            if(this.mouseDown) {
                this.drowToMap(e.offsetX,e.offsetY);
            }
        },
        // 绘制
        drowToMap:function(x,y,replay){
            if(!replay) {
                this.pathHistory.push({
                    x:x,
                    y:y
                });
            }
            this.ctx.strokeStyle="#10aeff";
            this.ctx.lineWidth=4;
            this.ctx.moveTo(this.map.startX,this.map.startY);
            this.ctx.lineTo(x,y);
            this.ctx.stroke();
            this.map.startX = x;
            this.map.startY = y;
        },
        // 保存
        saveMap:function(){
            let _this = this;
            // this.btnDisable = true;
            this.readonly = false;
            this.canvas.toBlob(function(blob) {
                console.log(blob);
                let $result = {
                    history:_this.pathHistory,
                    canvasSize:{
                        width:_this.canvasWidth,
                        height:_this.canvasHeight,
                    },
                    blob:blob,
                };
                let history = [..._this.pathHistory];
                _this.clearMap();
                _this.showHistory(history,20);
                console.log($result);
            });
        },
        // 撤回
        scrollBack:function(){
            let history = [...this.pathHistory];
            console.log(history);
            let len = history.length;
            // for(var i=len;i>0;i--) {
            //     let point = history[i];
            //     if(point===-1) {
            //         console.log('#');
            //         return false;
            //     } else {
            //         console.log(i);
            //     }
            // }
            this.clearMap(true);
            history.forEach((i,item)=>{
                this.map.startX = item.x;
                this.map.startY = item.y;
                if(item<1) {
                    this.map.startX = item.x;
                    this.map.startY = item.y;
                } else {
                    if(item.x>0 && item.y>0 && item.x>0 && item.y>0) {
                        this.map.startX = history[i-1].x;
                        this.map.startY = history[i-1].y;
                        this.drowToMap(item.x,item.y,true);
                    }
                }
            });
            // history = history.slice(0,len);
            // this.showHistory(history);
        },
        // 回放
        showHistory:function (historyObj,speed) {
            this.pathHistory = historyObj;
            let len = historyObj.length;
            let i = 0;
            this.timmer = setInterval(()=>{
                if(i>len-1) {
                    clearInterval(this.timmer);
                } else {
                    this.map.startX = historyObj[i].x;
                    this.map.startY = historyObj[i].y;
                    if(i<1) {
                        this.map.startX = historyObj[i].x;
                        this.map.startY = historyObj[i].y;
                    } else {
                        if(historyObj[i-1].x>0 && historyObj[i-1].y>0 && historyObj[i].x>0 && historyObj[i].y>0) {
                            this.map.startX = historyObj[i-1].x;
                            this.map.startY = historyObj[i-1].y;
                            this.drowToMap(historyObj[i].x,historyObj[i].y,true);
                        }
                    }
                    i++;
                }
            },speed?speed:0);
        },
    }
}
</script>
<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .scroll{
    position:absolute;
    overflow:scroll;
    -webkit-overflow-scrolling: touch;
    top:0;
    left:0;
    bottom:0;
    right:0;
  }

  .canvasBox {
    box-sizing:border-box;
    background:#fff;
    margin-bottom:20px;
    border:1px solid #dedede;
  }
</style>
