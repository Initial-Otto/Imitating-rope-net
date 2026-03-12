<script>
import emitter from './event-bus.js';
export default {
  data(){
    return{
      comment:[],
      ifClose:'none',
      collected:false,
      img:"",
      name:"",
      level:"",
      content:"",
      avatar_path:"",
      title:"",
      currentPostId: null,
      loadingComments: false,
      myComment:"",
      userId:"",
      music: "",        // 音乐文件的URL
      isPlaying: false,      // 当前播放状态
      currentTime: 0,        // 当前播放时间（秒）
      duration: 0,           // 音乐总时长（秒）
      volume: 0.7,           // 音量（0-1）
      audioPlayer: null,     // Audio对象引用
    }
  },
  methods:{
    async openPost(data) {
      this.ifClose = 'grid'
      this.currentPostId = data.id
      this.name = data.UPName
      this.img = data.img
      this.content = this.formatContent(data.text);
      this.avatar_path = data.avatar_path
      this.title = data.title
      this.level = data.level
      this.music = data.music
      await this.fetchComments(this.currentPostId);
      if (this.music) {
        await this.$nextTick(() => {
          this.initAudioPlayer();
          this.attemptAutoPlay();
        });
      }
    },
    initAudioPlayer() {
      if (!this.music) return;

      // 清理之前的音频实例
      if (this.audioPlayer) {
        this.audioPlayer.pause();
        this.audioPlayer.src = '';
      }

      // 创建新的Audio对象
      this.audioPlayer = new Audio(this.music);

      // 监听音频元数据加载完成
      this.audioPlayer.addEventListener('loadedmetadata', () => {
        this.duration = this.audioPlayer.duration;
      });

      // 监听播放时间更新
      this.audioPlayer.addEventListener('timeupdate', () => {
        this.currentTime = this.audioPlayer.currentTime;
      });

      // 监听播放结束
      this.audioPlayer.addEventListener('ended', () => {
        this.isPlaying = false;
        this.currentTime = 0;
      });

      // 设置音量
      this.audioPlayer.volume = this.volume;
    },

    // 尝试自动播放
    attemptAutoPlay() {
      if (this.audioPlayer) {
        this.audioPlayer.play().then(() => {
          this.isPlaying = true;
          console.log('音乐自动播放成功');
        }).catch(error => {
          console.log('自动播放被阻止，需要用户交互:', error);
          // 不自动播放，等待用户手动点击
        });
      }
    },

    // 播放/暂停音乐
    togglePlay() {
      if (!this.audioPlayer) {
        this.initAudioPlayer();
      }

      if (this.isPlaying) {
        this.audioPlayer.pause();
      } else {
        this.audioPlayer.play();
      }
      this.isPlaying = !this.isPlaying;
    },

    // 停止音乐
    stopMusic() {
      if (this.audioPlayer) {
        this.audioPlayer.pause();
        this.audioPlayer.currentTime = 0;
        this.isPlaying = false;
        this.currentTime = 0;
      }
    },

    // 跳转到指定时间
    seekTo(event) {
      if (!this.audioPlayer) return;

      const progressBar = event.target;
      const clickPosition = event.offsetX;
      const totalWidth = progressBar.clientWidth;
      const seekTime = (clickPosition / totalWidth) * this.duration;

      this.audioPlayer.currentTime = seekTime;
      this.currentTime = seekTime;
    },

    // 调整音量
    changeVolume(event) {
      const newVolume = parseFloat(event.target.value);
      this.volume = newVolume;

      if (this.audioPlayer) {
        this.audioPlayer.volume = newVolume;
      }
    },

    // 格式化时间显示（秒 -> MM:SS）
    formatTime(seconds) {
      if (isNaN(seconds) || !isFinite(seconds)) return '00:00';

      const mins = Math.floor(seconds / 60);
      const secs = Math.floor(seconds % 60);
      return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
    },
    formatContent(text) {
      if (!text) return '';

      // 将文本中的 [img]xxx[/img] 替换为 <br><img src="xxx">
      // 同时处理换行符，将 \n 替换为 <br>
      let formatted = text.replace(/\n/g, '<br>');

      // 替换 [img]xxx[/img] 为 <br><img src="xxx"><br>
      // 使用正则表达式全局匹配
      formatted = formatted.replace(/\[img\]([^\]]*)\[\/img\]/g, '<br><img src="$1" class="content-img"><br>');

      return formatted;
    },
    async fetchComments(postId) {
      this.loadingComments = true;

      try {
        // 调用API获取评论
        const response = await fetch('http://localhost:8000/src/php/get_comments.php', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            post_id: postId
          })
        });

        const result = await response.json();

        if (result.success) {
          // 处理评论数据
          this.comment = result.comments.map(comment => ({
            img: comment.avatar_path || "/img/head/1(1).jpeg",
            name: comment.username || "匿名用户",
            comment: comment.content_text,
            floor: comment.floor_number
          }));
        } else {
          console.error('获取评论失败:', result.message);
        }
      } catch (error) {
        console.error('请求评论失败:', error);
      } finally {
        this.loadingComments = false;
      }
    },
    async submitComment(forceCheck = false) {
      if (!this.myComment) return;
      const cachedAuth = localStorage.getItem('auth_cache');
      if (cachedAuth && !forceCheck) {
        const authData = JSON.parse(cachedAuth);
        const cacheAge = Date.now() - authData.lastCheck;

        // 如果缓存时间在2小时内，直接使用缓存
        if (cacheAge < 2 * 60 * 60 * 1000) {
          this.userId = authData.user_id
          return
        }else {
          alert("请先登录")
        }
      }

      try {
        const response = await fetch('http://localhost:8000/src/php/add_comment.php', {
          method: 'POST',
          credentials: 'include',
          body: JSON.stringify({
            post_id: this.currentPostId,
            content_text: this.myComment
          })
        });

        const result = await response.json();

        if (result.success) {
          // 评论成功，重新获取评论列表
          await this.fetchComments(this.currentPostId);
          this.myComment = ""; // 清空输入框
        } else {
          console.error('提交评论失败:', result.message);
          alert('评论失败: ' + result.message);
        }
      } catch (error) {
        console.error('提交评论请求失败:', error);
        alert('网络错误，请稍后重试');
      }
    },
    close(){
      this.ifClose='none'
      this.comment=[]
      this.stopMusic();
      this.musicUrl = null;
    },
    collect(){
      this.collected=true
    }
  },
  mounted() {
    // 使用 mitt 的 on 方法监听事件
    emitter.on('open-post-event', this.openPost);
    emitter.on('close-post-event',this.close)
  },
  beforeUnmount() { // 更正生命周期钩子名称
    // 组件卸载前，使用 mitt 的 off 方法移除事件监听，避免内存泄漏
    emitter.off('open-post-event', this.openPost);
    emitter.off('close-post-event',this.close)
    this.stopMusic()
  },
}

</script>

<template>
<div class="backdrop" :style="'display:'+ifClose"></div>
<div class="postBox" :style="'display:'+ifClose">
  <div class="head">
    <div class="profile">
      <img :src="avatar_path" alt="头像">
    </div>
    <p class="name">{{ name }}</p>
    <div class="level">v{{ level }}</div>
    <div class="musicBox" :style="'display:'+ifClose" v-if="music">
      <div class="player-container">

        <!-- 播放控制 -->
        <div class="player-controls">
          <!-- 播放/暂停按钮 -->
          <button class="control-btn play-btn" @click="togglePlay">
            {{ isPlaying ? '⏸️' : '▶️' }}
          </button>

          <!-- 进度条 -->
          <div class="progress-container">
            <div class="time-display">{{ formatTime(currentTime) }}</div>
            <div class="progress-bar" @click="seekTo">
              <div class="progress" :style="{ width: (currentTime / duration * 100) + '%' }"></div>
            </div>
            <div class="time-display">{{ formatTime(duration) }}</div>
          </div>

          <!-- 音量控制 -->
          <div class="volume-control">
            <div class="volume-icon">🔊</div>
            <input
                type="range"
                class="volume-slider"
                min="0"
                max="1"
                step="0.1"
                :value="volume"
                @input="changeVolume"
            />
          </div>
        </div>
      </div>
    </div>
    <div class="collect" :class="{ clicked: collected }" @click="collect">🌟</div>
    <svg
      class="close"
      viewBox="0 0 220 100"
      xmlns="http://www.w3.org/2000/svg"
      preserveAspectRatio="none"
      @click="close"
    >
      <!-- 透明的SVG画布，两个矩形将填满此区域 -->
      <rect x="0" y="0" width="180" height="100" rx="50" ry="50" fill="#FF0000"/>
      <rect x="120" y="0" width="100" height="100" rx="25" ry="25" fill="#FF0000" transform="skewX(-30)"/>
      <text x="100" y="50"
            text-anchor="middle"
            dominant-baseline="middle"
            fill="black"
            font-family="Arial"
            font-size="50"
            font-weight="bold">
        x
      </text>
    </svg>
  </div>
  <div class="imgBox">
    <img :src="img">
  </div>
  <div class="comment">
    <div id="postDetail" class="content-container">
      <p><span class="title">{{ title }}</span><br>
        <span v-html="content" class="content-text"></span>
      </p>
    </div>
    <div class="commentBox">
      <div
          v-for="(item,index) in comment"
          :key="index"
          class="commentImg"
          :style="{gridRow:index+1}"
      >
        <img :src="item.img">
      </div>
      <p
          v-for="(item,index) in comment"
          :key="index"
          :style="{gridRow:index+1}"
      >
        <span>{{item.name}}</span><br>{{item.comment}}
      </p>
      <div
          v-for="(item,index) in comment"
          :key="index"
          class="floor"
          :style="{gridRow:index+1}"
      >
        F{{item.floor}}
      </div>
    </div>

  </div>
  <div class="toComment" >
    <textarea id="text" name="text" v-model="myComment"></textarea>
    <button type="submit" @click="submitComment">发送</button>
  </div>
</div>

</template>

<style scoped>
.postBox{
  width: 90%;
  height: 80%;
  display: none;
  grid-template-columns: 10px minmax(200px,1fr) 3fr 10px;
  grid-template-rows: 80px 20px 1fr 80px 20px;
  gap:5px 20px;
  position: fixed;
  top:15%;
  left:5%;
  background-color: white/*rgb(40,40,40)*/;
  border-radius: 20px;
  border:10px /*rgb(60,60,60)*/lightcoral solid;
  z-index: 10;
  backdrop-filter: blur(10px);
}
.head{
  grid-row:1;
  grid-column-start: 1;
  grid-column-end: 6;
  background-color: lightpink;
  border-radius: 10px 10px 0 0;
  display: grid;
  grid-template-rows: 40px 40px;
  grid-template-columns: 80px 200px 1fr 280px;
}
.musicBox {
  grid-row-start: 1;
  grid-row-end: 3;
  grid-column: 3;
  width: 100%;
  height: 80%;
  background: lightpink;
  border-radius: 20px;
  border: 3px solid lightcoral;
  z-index: 10;
  display: none;
  box-shadow: 0 8px 25px rgba(255, 105, 180, 0.3);
  margin: 5px 0 5px 0;
}

.player-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: rgba(0,0,0,0);
}

.player-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-grow: 1;
  background-color: rgba(0,0,0,0);
}

.control-btn {
  background: rgba(255, 255, 255, 0);
  border: none;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  margin: 0 0 0 10px;
  font-size: 20px;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.control-btn:hover {
  transform: scale(1.1);
  background: white;
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
}

.play-btn{
  background: pink;
  color: white;
  box-shadow: 2px 2px 2px rgba(0, 0, 0, 0.3);
}

.progress-container {
  display: flex;
  align-items: center;
  flex-grow: 1;
  margin: 0 20px;
  background-color: rgba(0,0,0,0);
  box-shadow: 2px 2px 2px rgba(0, 0, 0, 0.3);
  border-radius: 5px;
}

.time-display {
  color: white;
  background-color: rgba(0,0,0,0);
  font-size: 14px;
  min-width: 40px;
  margin: 0 5px 0 5px;
  text-align: center;
  font-family: 'Courier New', monospace;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
}

.progress-bar {
  flex-grow: 1;
  height: 8px;
  background: rgba(255, 255, 255, 0);
  border-radius: 4px;
  margin: 0 15px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

.progress {
  height: 100%;
  background: linear-gradient(90deg, white, lightcoral);
  border-radius: 4px;
  transition: width 0.1s linear;
  position: relative;
}

.progress-bar:hover .progress::after {
  content: '';
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
  width: 12px;
  height: 12px;
  background: rgba(0,0,0,0);
  border-radius: 50%;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
}

.volume-control {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0);
  border-radius: 20px;
  padding: 5px 15px;
  box-shadow: 2px 2px 2px rgba(0, 0, 0, 0.3);
  margin: 0 10px 0 0;
}

.volume-icon {
  color: white;
  background-color: rgba(0,0,0,0);
  font-size: 18px;
  margin-right: 10px;
}

.volume-slider {
  width: 100px;
  height: 6px;
  -webkit-appearance: none;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 3px;
  outline: none;
}

.volume-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
}

.volume-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
  border: none;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
}
.profile{
  grid-row-start: 1;
  grid-row-end: 2;
  grid-column: 1;
  width: 60px;
  height: 60px;
  margin: 10px;
  border-radius: 50%;
  background-color: white;
  overflow: hidden;
}
.profile img{
  width: 100%;
}
.name{
  color:rgb(80,80,80);
  grid-column: 2;
  grid-row: 1;
  margin: 20px 0 0 5px;
  background-color: lightpink;
}
.level{
  border-radius: 10px;
  grid-column: 2;
  grid-row: 2;
  width: 50px;
  height: 20px;
  margin: 5px;
  color:black;
  background-color: lightcoral/*rgb(50,50,50)*/;
  text-align: center;
}
.imgBox{
  grid-row-start:3;
  grid-row-end: 5;
  grid-column: 2;
  background-color: lightpink;
  border-radius: 20px;
  border: 3px /*rgb(60,60,60)*/white solid;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  overflow: hidden;
}
.comment{
  grid-row:3;
  grid-column-start: 3;
  grid-column-end: 5;
  background-color: lightpink;
  border-radius: 20px;
  border: 3px /*rgb(60,60,60)*/white solid;
  overflow: auto;
  position: relative;
  transform: translateZ(0);
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: min-content min-content;
}
.collect{
  width: 40px;
  height: 40px;
  border-radius: 20px;
  position: absolute;
  top:20px;
  right: 125px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 30px;
}
.collect.clicked{
  background-color: yellow;
}
.close{
  position: absolute;
  top:20px;
  right: 15px;
  width: 100px;
  height: 40px;
  background-color: rgba(0,0,0,0);
  transition: transform 0.5s;
  cursor: pointer;
}
.close:hover{
  transform: scale(1.05);
}
.imgBox img{
  width: 100%;
}
.commentBox{
  width: 100%;
  height: auto;
  min-height: 100px;
  grid-row: 2;
  display: grid;
  grid-template-columns: 70px 1fr 70px;
  grid-auto-flow: row;
  grid-auto-rows: min-content;
  gap: 20px;
  margin: 30px 0 0 0;
  background-color: lightpink;
}
.content-container {
  grid-row:1;
  width: 90%;
  height: auto;
  min-height: 100px;
  padding: 20px 20px 0 20px;
  background-color: lightpink;
  border-radius: 10px;
}

.content-container * {
  background-color: inherit; /* 继承父元素背景 */
  word-wrap: break-word;
  word-break: break-all;
  white-space: pre-line;
}
#postDetail >>> .content-img {
  max-width: 90%;
  height: auto;
  margin: 10px 0;
  border-radius: 10px;
  display: block;
}
.title {
  font-size: 20px;
  color: rgb(140,140,140);
}

.content-text {
  color: rgb(100,100,100);
  line-height: 1.5; /* 添加行高改善阅读体验 */
}

.commentImg{
  width: 60px;
  height: auto;
  left: 10px;
  position: relative;
  grid-column: 1;
  background-color: lightpink;
}
.commentImg img{
  width: 60px;
  height: 60px;
  border-radius: 50%;
}
.commentBox p span{
  color: rgb(140,140,140);
  background-color: lightpink;
}
.commentBox p{
  position: relative;
  left: 20px;
  color: rgb(100,100,100);
  grid-column: 2;
  border-bottom: 3px solid white;
  word-wrap: break-word; /* 允许长单词换行 */
  word-break: break-all; /* 强制换行 */
  white-space: pre-line; /* 保留换行符并正常换行 */
  margin: 0;
  padding-bottom: 1rem;
  background-color: lightpink;
}
.commentBox .floor{
  position: relative;
  margin-left: auto;
  right: 10px;
  top: 5px;
  width: 40px;
  height: 20px;
  border-radius: 0 10px 10px 10px;
  background-color: lightcoral/*rgb(100,100,100)*/;
  text-align: center;
  grid-column: 3;
}
.toComment{
  grid-row: 4;
  grid-column-start: 3;
  grid-column-end: 5;
  background-color: lightpink;
  border-radius: 20px;
  border: 3px /*rgb(60,60,60)*/white solid;
  display: flex;
  /* 确保宽度计算包含padding和border */
  box-sizing: border-box;
  justify-content: space-between;
  padding: 7px 5%;
}
.toComment textarea{
  background-color: lightcoral/*rgb(140,140,140)*/;
  width: 80%;
  height: 60px;
  border-radius: 10px;
  padding: 0;
  border: none;
  overflow: auto;
  resize: none;
  font-size: 20px;
}
.toComment button{
  background-color: /*rgb(100,100,100)*/lightcoral;
  width: 18%;
  height: 60px;
  border-radius: 10px;
  border: 0;
  padding: 0;
  margin: 0;
  flex-shrink: 0;
}
.backdrop {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  backdrop-filter: blur(10px);
  background-color: rgba(0, 0, 0, 0.5); /* 半透明黑色 */
  z-index: 10; /* 低于POSTPage但高于其他内容 */
}

</style>