<template>
    <div>
        <div class="content">
            <ul class="output" v-loading="loading">
                <li v-for="message in messages" :key="message.index" :class="[message.username==user.name? 'right' : 'left']">
                    <div class="time">{{message.date}}</div>
                    <img :src="message.picUrl" class="pic">
                    <div class="username">{{message.username}}</div>
                    <div class="msg" v-html="message.msg"></div>
                </li>
            </ul>
            <!-- <textarea v-model="text" class="input" autofocus
            @keydown.enter.exact="sendMessage()"
            @keydown.ctrl.enter="addNewLine()">
            </textarea> -->
            <upload class="up" v-if="upload_show" @closeFile='closeFile' @sendFile='sendFile'></upload>
            <emoji class="emoji" v-if="emoji_show" @addEmoji="addEmoji"></emoji>
            <!-- <div v-html="text" class="input" contenteditable="true" @input="changeText($event)"
            @keydown.enter.exact="sendMessage()"
            @keydown.ctrl.enter="addNewLine()">
            </div> -->
            <divInput class="input" v-model="text" @send="sendMessage"></divInput>
            <div class="control">
                <el-button size="small" type="primary" class="el-btn" @click="emoji()">表情</el-button>
                <el-button size="small" type="primary" class="el-btn" @click="sendPic()">图片</el-button>
                <el-button size="small" type="primary" class="el-btn" @click="openFile()">文件</el-button>
                <input type="file" accept="image/*" id="upload" @change="getFileName($event)">
                <el-button size="small" type="primary" class="el-btn" @click="sendMessage()">发送</el-button>
            </div>
        </div>
    </div>
</template>

<script>
import divInput from './divInput'
import emoji from './emoji'
import upload from './upload'
import dateFormat from '../utils/date'
import {history,uploadImg} from '../api/api'
export default {
    data(){
        return{
            text: '',
            socket: '',
            emoji_show: false,
            loading: true,
            user: {
                name:JSON.parse(sessionStorage.getItem('user')).name,
                uid: JSON.parse(sessionStorage.getItem('user')).uid
            },
            friend: {
                name:JSON.parse(sessionStorage.getItem('friend')).name,
                uid: JSON.parse(sessionStorage.getItem('friend')).uid,
                type: JSON.parse(sessionStorage.getItem('friend')).type//0表示群聊
            },
            messages: [],
            upload_show: false,
            bridge: []
        }
    },
    computed:{
        
    },
    methods:{
        conWebSocket(){
            let vm = this;
            if(this.friend.type!=0)
                this.bridge=[this.user.uid,this.friend.uid];
            if(window.WebSocket){
                vm.socket = new WebSocket('ws://localhost:4000');
                let socket = vm.socket;
                socket.onopen = function(e){
                    console.log("连接服务器成功");
                    vm.socket.send(JSON.stringify({
                        uid: vm.user.uid,
                        type: -1,//连接
                        username: vm.user.name,
                        msg: '',
                        bridge: [],
                        groupId: ''
                    }))
                }
                socket.onclose = function(e){
                    console.log("服务器关闭");
                }
                socket.onerror = function(){
                    console.log("连接出错");
                }
                // 接收服务器的消息
                socket.onmessage = async function(e){
                    let message = JSON.parse(e.data);
                    if(message.msg=="")
                        return;
                    if(message.bridge.sort().join(',')==vm.bridge.sort().join(',')){
                        await vm.$store.dispatch('getUserInfo',message.username);
                        let picUrl = {picUrl:vm.$store.state.info.picUrl};
                        message = {...message,...picUrl};
                        vm.messages.push(message);
                        vm.$previewRefresh();
                        vm.scrollTop();
                    }    
                } 
            }
        },
        scrollTop(){//滚动条自动下移
            var output=document.querySelector('.output');
            setTimeout(()=>{
                output.scrollTop=output.scrollHeight;
            },50);
        },
        sendMessage(e){
            // if(window.event){
            //     window.event.returnValue=false;
            // }else{
            //     e.preventDefault();
            // }
            if(!this.text){
                return this.$message({
                    message:'内容为空',
                    type:'error'
                });
            }
            this.post(this.text);
            this.scrollTop();
            this.text='';           
        },
        post(data){
            let vm=this;
            if(this.friend.type==0){
                this.socket.send(JSON.stringify({
                    username:vm.user.name,
                    uid:vm.uid,
                    msg:data,
                    type:0,
                    bridge:[]
                }))
                return;
            }
            this.socket.send(JSON.stringify({
                uid: vm.user.uid,
                type: 1,
                username: vm.user.name,
                friend: vm.friend.name,
                msg: data,
                bridge: vm.bridge,
            }))
        },
        emoji(){
            this.emoji_show=this.emoji_show==true?false:true;
        },
        addEmoji(data){
            this.text+=data;
            this.emoji();
        },
        async gethistory(){
            let param=new URLSearchParams();
            param.append('type',this.friend.type);
            param.append('username',this.user.name);
            param.append('friend',this.friend.name);
            let data=await history(param);
            if(data=='net'){
                this.loading=false;
                return;
            }
            for(let i=0;i<data.data.length;i++){
                let message={};
                await this.$store.dispatch('getUserInfo',data.data[i].username);
                let date=dateFormat(new Date(data.data[i].date),'yyyy-MM-dd HH:mm:ss');
                message={
                    username:data.data[i].username,
                    msg:data.data[i].msg,
                    date,
                    picUrl:this.$store.state.info.picUrl
                };
                this.messages.push(message);
            }
            this.$previewRefresh();
            this.scrollTop();
            this.loading=false;     
        },
        sendPic(){
            document.querySelector('#upload').click();
        },
        openFile(){
            this.upload_show=true;
        },
        closeFile(){
            this.upload_show=false;
        },
        sendFile(data){
            //console.log(data);
            let msg=`<a href="${data.url}" download><i class="iconfont icon-wenjian" style="color:yellow;font-size:2rem"></i>${data.fileName}</a>`;
            this.post(msg);
        },
        getFileName(e){
            // var fileName=document.getElementById("upload").files[0];
            // var reader = new FileReader();
            // reader.onload = e=>{
            //     this.text+=`<img src=${this.result} height="200px" width="240px">`;
            // };
            // reader.readAsDataURL(fileName);

            let tFiles = e.target.files[0];
            let param = new FormData();
            param.append('image',tFiles,tFiles.name);
            let config = {
                    //添加请求头 
                    headers: {
                        "Content-Type": "multipart/form-data"
                    },
            };
            uploadImg(param,config).then(response=>{
                //console.log(response.data);
                let msg=`<img src="${response.data.url}" height="200px" width="240px" preview="1" preview-text=""></img>`;
                this.post(msg);
            }).catch(e=>{
                console.log(e);
            })
        }
    },
    mounted(){
        this.conWebSocket();
        this.gethistory();
    },
    components:{
        emoji,
        divInput,
        upload
    }
}
</script>

<style lang="less" scoped>
    .content{
        height: calc(~"100vh - 80px");
        // width: calc(~"100vw - 220px");
        width: 100vw;
        // float: right;
       .output{
            border-bottom: 2px solid #e4e4e4; 
            height: calc(~"100vh - 216px");
            overflow: auto; 
            .left{
                .pic{
                width: 50px;
                height: 50px;
                border-radius: 50%;
                float: left;
                }
                .username{
                    opacity: 0.3;
                }
                .msg{
                    display: inline-block;
                    margin-left: 0.5rem;
                    position: relative;
                    background: aqua;  
                    max-width: calc(~"100vw - 50px");
                    border-radius: 10px; 
                    padding: 0.5rem;
                    white-space: pre-wrap;  //换行输出
                }
                .time{
                    text-align: center;
                    opacity: 0.4;
                }
            }
            .right{
                float: right;
                width: 100%;
                overflow: hidden;
                .pic{
                    width: 50px;
                    height: 50px;
                    border-radius: 50%;
                    float: right;
                }
                .username{
                    float: right;
                    opacity: 0.3;
                }
                .msg{
                    float: right;
                    display: inline-block;
                    background: aqua;  
                    max-width: calc(~"100vw - 50px");
                    margin-top: 1.5rem;
                    border-radius: 10px; 
                    padding: 0.5rem;
                    white-space: pre-wrap;  //换行输出
                }
                .time{
                    text-align: center;
                    opacity: 0.4;
                }
            }     
        } 
        .up{
            position: absolute;
            top: 40%;
            left: 50%;
            transform: translateX(-50%);
        }
        .emoji{
            position: absolute;
            top: 50%;
            margin-top: 5rem;
        }
        .input{
            height: 130px;
            width: 100%;
            border: none;
            resize: none;
            outline: none;
            border-bottom: 1px solid #e4e4e4;
        }
        .control{
            float: right;
            #upload{
                display: none;
            }
        }
    }
</style>