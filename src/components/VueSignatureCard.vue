<template>
    <div>
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
        <slot name="tool" v-if="showButton">
            <button :disabled="disabled" @click="clearMap(true)" type="button" plain >重新签</button>
            <button :disabled="disabled || pathHistory.length<10" type="button" @click="scrollBack">撤回</button>
            <button :disabled="disabled || pathHistory.length<10" type="button" @click="saveMap">提交签名</button>
        </slot>
    </div>
</template>

<script>
export default {
  name: 'VueSignatureCard',
    props:{
        disabled:{
            required:false,
            type:Boolean,
            default:false,
        },
        showButton:{
            required:false,
            type:Boolean,
            default:true,
        },
        color:{
            required:false,
            type:String,
            default:'#000',
        },
        lineWidth:{
            required:false,
            type:Number,
            default:3,
        },
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
            if(this.readonly==false || this.disabled==true) {return false;}
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
            if(this.readonly==false || this.disabled==true) {return false;}
            let x = e.changedTouches[0].clientX-this.map.offsetLeft;
            let y = e.changedTouches[0].clientY-this.map.offsetTop;
            this.drowToMap(x,y);
        },
        // pc
        handleMouseDown:function(e){
            if(this.readonly==false || this.disabled==true) {return false;}
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
            if(this.readonly==false || this.disabled==true) {return false;}
            this.mouseDown = false;
            this.map.startX = e.offsetX;
            this.map.startY = e.offsetY;
            this.pathHistory.push(-1,-1);
            this.$emit('drawing',this.pathHistory);
        },
        handleMouseMove:function(e){
            if(this.readonly==false || this.disabled==true) {return false;}
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
            this.ctx.strokeStyle=this.color;
            this.ctx.lineWidth=this.lineWidth;
            this.ctx.moveTo(this.map.startX,this.map.startY);
            this.ctx.lineTo(x,y);
            this.ctx.stroke();
            this.map.startX = x;
            this.map.startY = y;
        },
        // 保存
        saveMap:function(action,speed){
            let _this = this;
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
                if(action===true) {
                    _this.clearMap();
                    _this.showHistory(history,speed?speed:0);
                }
                _this.$emit('success',$result);
            });
        },
        // 撤回
        scrollBack:function(){
            let history = [...this.pathHistory];
            let len = history.length;
            let cut = false;
            for(var i=len-1;i>0;i--) {
                let point = history[i];
                if(point!==-1 && cut===false) {
                    cut = true;
                } else if(cut===true && point===-1) {
                    history = history.slice(0,i);
                    this.clearMap();
                    this.showHistoryQuick(history);
                    this.pathHistory = history;
                    return false;
                } else {
                    this.clearMap();
                }
            }
        },
        // 回放
        showHistoryQuick:function(history){
            history.forEach((item,i)=>{
                this.map.startX = item.x;
                this.map.startY = item.y;
                if(i>0) {
                    if(item<1) {
                        this.map.startX = item.x;
                        this.map.startY = item.y;
                    } else {
                        if(history[i-1].x>0 && history[i-1].y>0 && item.x>0 && item.y>0) {
                            this.map.startX = history[i-1].x;
                            this.map.startY = history[i-1].y;
                            this.drowToMap(item.x,item.y,true);
                        }
                    }
                }
            });
        },
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
        // 清除
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
