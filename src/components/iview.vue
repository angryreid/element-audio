<template>
  <!-- <div class="di main-wrap" v-loading="audio.waiting"> -->
  <div class="di main-wrap">
    <!-- 这里设置了ref属性后，在vue组件中，就可以用this.$refs.audio来访问该dom元素 -->
    <audio
      ref="audio"
      class="dn"
      :src="url"
      :preload="audio.preload"
      @play="onPlay"
      @error="onError"
      @waiting="onWaiting"
      @pause="onPause"
      @timeupdate="onTimeupdate"
      @loadedmetadata="onLoadedmetadata"
    ></audio>
    <div class="prop-warp">
      <Button
        type="primary"
        @click="startPlayOrPause"
        shape="circle"
        class="play-btn"
        :icon="audio.playing?'ios-pause':'ios-play'"
      ></Button>

      <div class="outer-line">
        <Button
          v-show="!controlList.noSpeed"
          type="text"
          @click="changeSpeed"
        >{{audio.speed | transSpeed}}</Button>

        <div class="play-time">{{ audio.currentTime | formatSecond}}</div>

        <Slider
          v-show="!controlList.noProcess"
          v-model="sliderTime"
          :tip-format="formatProcessToolTip"
          @on-change="changeCurrentTime"
          class="slider"
        ></Slider>

        <div class="play-time">{{ audio.maxTime | formatSecond }}</div>

        <div class="muted" v-show="!controlList.noMuted" shape="circle" @click="startMutedOrNot">
          <Icon type="md-volume-up" size="18" v-if="!audio.muted" />
          <Icon type="md-volume-off" size="18" v-else />
        </div>

        <Slider
          v-show="!controlList.noVolume"
          v-model="volume"
          :tip-format="formatVolumeToolTip"
          @on-change="changeVolume"
          class="slider"
        ></Slider>
      </div>

      <!-- 暂不开放下载 -->
      <!-- <a :href="url" v-show="!controlList.noDownload" target="_blank" class="download" download>下载</a> -->
    </div>
  </div>
</template>

<script>
function realFormatSecond(second) {
  var secondType = typeof second;

  var covertNumberToTwoDigit = function(number) {
    return number.toString().replace(/^(\d)$/, "0$1");
  };

  if (secondType === "number" || secondType === "string") {
    second = parseInt(second);

    var hours = covertNumberToTwoDigit(Math.floor(second / 3600));
    second = second - hours * 3600;
    var mimute = Math.floor(second / 60);
    second = second - mimute * 60;

    return (
      hours + ":" + ("0" + mimute).slice(-2) + ":" + ("0" + second).slice(-2)
    );
  } else {
    return "00:00:00";
  }
}

export default {
  props: {
    theUrl: {
      type: String,
      required: true
    },
    theSpeeds: {
      type: Array,
      default() {
        return [1, 1.5, 2];
      }
    },
    theControlList: {
      type: String,
      default: ""
    }
  },
  name: "VueAudio",
  data() {
    return {
      url: this.theUrl || "http://devtest.qiniudn.com/secret base~.mp3",
      audio: {
        currentTime: 0,
        maxTime: 0,
        playing: false,
        muted: false,
        speed: 1,
        waiting: true,
        preload: "auto"
      },

      sliderTime: 0,
      volume: 50,
      speeds: this.theSpeeds,

      controlList: {
        // 不显示下载
        noDownload: false,
        // 不显示静音
        noMuted: false,
        // 不显示音量条
        noVolume: false,
        // 不显示进度条
        noProcess: false,
        // 只能播放一个
        onlyOnePlaying: false,
        // 不要快进按钮
        noSpeed: false
      }
    };
  },
  methods: {
    setControlList() {
      let controlList = this.theControlList.split(" ");
      controlList.forEach(item => {
        if (this.controlList[item] !== undefined) {
          this.controlList[item] = true;
        }
      });
    },
    changeSpeed() {
      let index = this.speeds.indexOf(this.audio.speed) + 1;
      this.audio.speed = this.speeds[index % this.speeds.length];
      this.$refs.audio.playbackRate = this.audio.speed;
    },
    startMutedOrNot() {
      this.$refs.audio.muted = !this.$refs.audio.muted;
      this.audio.muted = this.$refs.audio.muted;
    },
    // 音量条toolTip
    formatVolumeToolTip(index) {
      return "音量条: " + index;
    },
    // 进度条toolTip
    formatProcessToolTip(index = 0) {
      index = parseInt((this.audio.maxTime / 100) * index);
      return "进度条: " + realFormatSecond(index);
    },
    // 音量改变
    changeVolume(index = 0) {
      this.$refs.audio.volume = index / 100;
      this.volume = index;
    },
    // 播放跳转
    changeCurrentTime(index) {
      console.log(index);
      this.$refs.audio.currentTime = parseInt(
        (index / 100) * this.audio.maxTime
      );
    },
    startPlayOrPause() {
      return this.audio.playing ? this.pausePlay() : this.startPlay();
    },
    // 开始播放
    startPlay() {
      this.$refs.audio.play();
    },
    // 暂停
    pausePlay() {
      this.$refs.audio.pause();
    },
    // 当音频暂停
    onPause() {
      this.audio.playing = false;
    },
    // 当发生错误, 就出现loading状态
    onError() {
      this.audio.waiting = true;
    },
    // 当音频开始等待
    onWaiting(res) {
      console.log(res);
    },
    // 当音频开始播放
    onPlay(res) {
      this.audio.playing = true;
      this.audio.loading = false;

      if (!this.controlList.onlyOnePlaying) {
        return;
      }

      let target = res.target;

      let audios = document.getElementsByTagName("audio");

      [...audios].forEach(item => {
        if (item !== target) {
          item.pause();
        }
      });
    },
    // 当timeupdate事件大概每秒一次，用来更新音频流的当前播放时间
    onTimeupdate(res) {
      // console.log('timeupdate')
      // console.log(res)
      this.audio.currentTime = res.target.currentTime;
      this.sliderTime = parseInt(
        (this.audio.currentTime / this.audio.maxTime) * 100
      );
    },
    // 当加载语音流元数据完成后，会触发该事件的回调函数
    // 语音元数据主要是语音的长度之类的数据
    onLoadedmetadata(res) {
      console.log("loadedmetadata");
      console.log(res);
      this.audio.waiting = false;
      this.audio.maxTime = parseInt(res.target.duration);
    }
  },
  filters: {
    formatSecond(second = 0) {
      return realFormatSecond(second);
    },
    transPlayPause(value) {
      return value ? "暂停" : "播放";
    },
    transMutedOrNot(value) {
      return value ? "放音" : "静音";
    },
    transSpeed(value) {
      return "倍速: x" + value;
    }
  },
  created() {
    this.setControlList();
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.main-wrap {
  padding: 10px 15px;
}
.prop-warp {
  display: flex;
  align-items: center;
}
.play-btn {
  width: 48px;
  height: 48px;
  font-size: 24px;
  font-weight: bolder;
}
.slider {
  display: inline-block;
  width: 100px;
  margin: 0 15px;
}

.di {
  display: inline-block;
}

.download {
  color: #409eff;
  margin-left: 15px;
}

.dn {
  display: none;
}
.muted {
  display: inline-block;
  cursor: pointer;
  margin-left: 15px;
}

.outer-line {
  display: flex;
  align-items: center;
  height: 38px;
  padding: 0 12px;
  border: 1px solid #e8e8e8;
  border-radius: 24px;
  margin: 0 0 0 5px;
}

.play-time {
  color: rgba(84, 84, 84, 0.5);
  font-size: 12px;
}
</style>
