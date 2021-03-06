<template>
  <transition name="slide-up">
    <div class="lyrics-page" :class="{ 'no-lyric': noLyric }">
      <div class="left-side">
        <div>
          <div class="cover">
            <div class="cover-container">
              <img :src="imageUrl" />
              <div
                class="shadow"
                :style="{ backgroundImage: `url(${imageUrl})` }"
              ></div>
            </div>
          </div>
          <div class="controls">
            <div class="top-part">
              <div class="track-info">
                <div class="title" :title="currentTrack.name"
                  ><router-link
                    :to="`/${player.playlistSource.type}/${player.playlistSource.id}`"
                    @click.native="toggleLyrics"
                    >{{ currentTrack.name }}</router-link
                  ></div
                >
                <div class="subtitle"
                  ><router-link
                    :to="`/artist/${currentTrack.ar[0].id}`"
                    @click.native="toggleLyrics"
                    >{{ currentTrack.ar[0].name }}</router-link
                  >
                  -
                  <router-link
                    :to="`/album/${currentTrack.al.id}`"
                    @click.native="toggleLyrics"
                    :title="currentTrack.al.name"
                    >{{ currentTrack.al.name }}</router-link
                  ></div
                >
              </div>
              <div class="buttons">
                <button-icon
                  @click.native="playerRef.likeCurrentSong"
                  :title="$t('player.like')"
                  ><svg-icon
                    :icon-class="
                      playerRef.isCurrentTrackLiked ? 'heart-solid' : 'heart'
                    "
                /></button-icon>
                <!-- <button-icon @click.native="openMenu" title="Menu"
                  ><svg-icon icon-class="more"
                /></button-icon> -->
              </div>
            </div>
            <div class="progress-bar">
              <span>{{ formatTrackTime(progress) || "0:00" }}</span>
              <div class="slider">
                <vue-slider
                  v-model="progress"
                  :min="0"
                  :max="progressMax"
                  :interval="1"
                  :drag-on-click="true"
                  :duration="0"
                  :dotSize="12"
                  :height="2"
                  :tooltipFormatter="formatTrackTime"
                  @drag-end="setSeek"
                  ref="progress"
                ></vue-slider
              ></div>
              <span>{{ formatTrackTime(progressMax) }}</span>
            </div>
            <div class="media-controls">
              <button-icon
                @click.native="playerRef.repeat"
                :title="
                  player.repeatMode === 'one'
                    ? $t('player.repeatTrack')
                    : $t('player.repeat')
                "
                :class="{ active: player.repeatMode !== 'off' }"
              >
                <svg-icon
                  icon-class="repeat"
                  v-show="player.repeatMode !== 'one'"
                />
                <svg-icon
                  icon-class="repeat-1"
                  v-show="player.repeatMode === 'one'"
                />
              </button-icon>
              <div class="middle">
                <button-icon
                  @click.native="playerRef.previous"
                  :title="$t('player.previous')"
                  ><svg-icon icon-class="previous"
                /></button-icon>
                <button-icon
                  @click.native="playerRef.play"
                  :title="$t(player.playing ? 'player.pause' : 'player.play')"
                  ><svg-icon :icon-class="playerRef.playing ? 'pause' : 'play'"
                /></button-icon>
                <button-icon
                  @click.native="playerRef.next"
                  :title="$t('player.next')"
                  ><svg-icon icon-class="next"
                /></button-icon>
              </div>
              <button-icon
                @click.native="playerRef.shuffle"
                :title="$t('player.shuffle')"
                :class="{ active: player.shuffle }"
                ><svg-icon icon-class="shuffle"
              /></button-icon>
            </div>
          </div>
        </div>
      </div>
      <div class="right-side">
        <transition name="slide-fade">
          <div class="lyrics-container" ref="lyricsContainer" v-show="!noLyric">
            <div class="line" id="line-1"></div>
            <div
              class="line"
              :class="{
                highlight: highlightLyricIndex === index,
              }"
              :style="lineStyles"
              v-for="(line, index) in lyricWithTranslation"
              :key="index"
              :id="`line${index}`"
              v-html="formatLine(line)"
            ></div>
          </div>
        </transition>
      </div>
      <div class="close-button" @click="toggleLyrics">
        <button><svg-icon icon-class="arrow-down" /></button>
      </div>
    </div>
  </transition>
</template>

<script>
// The lyrics page of Apple Music is so gorgeous, so I copy the design.
// Some of the codes are from https://github.com/sl1673495/vue-netease-music

import { mapState, mapMutations } from "vuex";
import VueSlider from "vue-slider-component";
import { formatTrackTime } from "@/utils/common";
import { getLyric } from "@/api/track";
import { lyricParser } from "@/utils/lyrics";
import ButtonIcon from "@/components/ButtonIcon.vue";

export default {
  name: "Lyrics",
  components: {
    VueSlider,
    ButtonIcon,
  },
  data() {
    return {
      lyricsInterval: null,
      lyric: [],
      tlyric: [],
      highlightLyricIndex: -1,
      minimize: true,
    };
  },
  computed: {
    ...mapState(["player"]),
    currentTrack() {
      return this.player.currentTrack;
    },
    imageUrl() {
      return this.player.currentTrack.al.picUrl + "?param=1024y1024";
    },
    progress: {
      get() {
        return this.$parent.$refs.player.progress;
      },
      set(value) {
        this.$parent.$refs.player.setProgress(value);
      },
    },
    progressMax() {
      return this.$parent.$refs.player.progressMax;
    },
    lyricWithTranslation() {
      let ret = [];
      // 空内容的去除
      const lyricFiltered = this.lyric.filter(({ content }) =>
        Boolean(content)
      );
      // content统一转换数组形式
      if (lyricFiltered.length) {
        lyricFiltered.forEach((l) => {
          const { rawTime, time, content } = l;
          const lyricItem = { time, content, contents: [content] };
          const sameTimeTLyric = this.tlyric.find(
            ({ rawTime: tLyricRawTime }) => tLyricRawTime === rawTime
          );
          if (sameTimeTLyric) {
            const { content: tLyricContent } = sameTimeTLyric;
            if (content) {
              lyricItem.contents.push(tLyricContent);
            }
          }
          ret.push(lyricItem);
        });
      } else {
        ret = lyricFiltered.map(({ time, content }) => ({
          time,
          content,
          contents: [content],
        }));
      }
      return ret;
    },
    haveTranslation() {
      return this.tlyric.length > 0;
    },
    lineStyles() {
      return {
        fontSize: this.haveTranslation ? "28px" : "36px",
      };
    },
    playerRef() {
      return this.$parent.$refs.player;
    },
    showLyrics() {
      return this.$store.state.showLyrics;
    },
    noLyric() {
      return this.lyric.length == 0;
    },
  },
  created() {
    this.getLyric();
  },
  destroyed() {
    clearInterval(this.lyricsInterval);
  },
  methods: {
    ...mapMutations(["toggleLyrics"]),
    getLyric() {
      return getLyric(this.currentTrack.id).then((data) => {
        if (!data?.lrc?.lyric) {
          this.lyric = [];
          this.tlyric = [];
          return false;
        } else {
          let { lyric, tlyric } = lyricParser(data);
          this.lyric = lyric;
          this.tlyric = tlyric;
          return true;
        }
      });
    },
    formatTrackTime(value) {
      return formatTrackTime(value);
    },
    setSeek() {
      let value = this.$refs.progress.getValue();
      this.$parent.$refs.player.setProgress(value);
      this.$parent.$refs.player.player.seek(value);
    },
    setLyricsInterval() {
      this.lyricsInterval = setInterval(() => {
        const progress = this.player.seek() ?? 0;
        let oldHighlightLyricIndex = this.highlightLyricIndex;
        this.highlightLyricIndex = this.lyric.findIndex((l, index) => {
          const nextLyric = this.lyric[index + 1];
          return (
            progress >= l.time && (nextLyric ? progress < nextLyric.time : true)
          );
        });
        if (oldHighlightLyricIndex !== this.highlightLyricIndex) {
          const el = document.getElementById(`line${this.highlightLyricIndex}`);
          if (el)
            el.scrollIntoView({
              behavior: "smooth",
              block: "center",
            });
        }
      }, 50);
    },
    formatLine(line) {
      const showLyricsTranslation = this.$store.state.settings
        .showLyricsTranslation;
      if (showLyricsTranslation && line.contents[1]) {
        return line.contents[0] + "<br/>" + line.contents[1];
      } else {
        return line.contents[0];
      }
    },
  },
  watch: {
    currentTrack() {
      this.getLyric();
    },
    showLyrics(show) {
      if (show) {
        this.setLyricsInterval();
      } else {
        clearInterval(this.lyricsInterval);
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.lyrics-page {
  position: fixed;
  top: 0;
  right: 0;
  left: 0;
  bottom: 0;
  z-index: 200;
  background: var(--color-body-bg);
  display: flex;
}

.left-side {
  flex: 1;
  display: flex;
  justify-content: flex-end;
  margin-right: 32px;
  margin-top: 24px;
  align-items: center;
  transition: all 0.5s;
  .controls {
    max-width: 54vh;
    margin-top: 24px;
    color: var(--color-text);
    .title {
      margin-top: 8px;
      font-size: 1.4rem;
      font-weight: 600;
      opacity: 0.88;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 1;
      overflow: hidden;
    }
    .subtitle {
      margin-top: 4px;
      font-size: 1rem;
      opacity: 0.58;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 1;
      overflow: hidden;
    }

    .top-part {
      display: flex;
      justify-content: space-between;
      .buttons {
        display: flex;
        align-items: center;
        button {
          margin: 0 0 0 4px;
        }
        .svg-icon {
          opacity: 0.58;
          height: 18px;
          width: 18px;
        }
      }
    }
    .progress-bar {
      margin-top: 22px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      .slider {
        width: 100%;
        flex-grow: grow;
        padding: 0 10px;
      }
      span {
        font-size: 15px;
        opacity: 0.58;
        min-width: 28px;
      }
    }
    .media-controls {
      display: flex;
      justify-content: center;
      margin-top: 18px;
      align-items: center;
      button {
        margin: 0;
      }
      .svg-icon {
        opacity: 0.38;
        height: 14px;
        width: 14px;
      }
      .active .svg-icon {
        opacity: 0.88;
      }
      .middle {
        padding: 0 16px;
        display: flex;
        align-items: center;
        button {
          margin: 0 8px;
        }
        button:nth-child(2) .svg-icon {
          height: 28px;
          width: 28px;
          padding: 2px;
        }
        .svg-icon {
          opacity: 0.88;
          height: 22px;
          width: 22px;
        }
      }
    }
  }
}

.cover {
  position: relative;

  .cover-container {
    position: relative;
  }
  img {
    border-radius: 0.75em;
    width: 54vh;
    height: 54vh;
    user-select: none;
    object-fit: cover;
  }
  .shadow {
    position: absolute;
    top: 12px;
    height: 54vh;
    width: 54vh;
    filter: blur(16px) opacity(0.6);
    transform: scale(0.92, 0.96);
    z-index: -1;
    background-size: cover;
    border-radius: 0.75em;
  }
}

.right-side {
  flex: 1;
  font-weight: 600;
  color: var(--color-text);
  margin-right: 24px;
  .lyrics-container {
    height: 100%;
    display: flex;
    flex-direction: column;
    padding-left: 96px;
    max-width: 460px;
    overflow-y: auto;
    transition: 0.5s;
    .line {
      margin-top: 38px;
      opacity: 0.28;
    }
    .highlight {
      opacity: 0.98;
      transition: 0.5s;
    }
  }
  ::-webkit-scrollbar {
    display: none;
  }
  .lyrics-container .line:first-child {
    margin-top: 50vh;
  }
  .lyrics-container .line:last-child {
    margin-bottom: calc(50vh - 128px);
  }
}

.close-button {
  position: fixed;
  top: 24px;
  right: 24px;
  z-index: 300;
  border-radius: 0.75rem;
  height: 44px;
  width: 44px;
  display: flex;
  justify-content: center;
  align-items: center;
  opacity: 0.28;
  transition: 0.2s;
  -webkit-app-region: no-drag;
  .svg-icon {
    color: var(--color-text);
    padding-top: 5px;
    height: 22px;
    width: 22px;
  }
  &:hover {
    background: var(--color-secondary-bg);
    opacity: 0.88;
  }
}

.lyrics-page.no-lyric {
  .left-side {
    transition: all 0.5s;
    transform: translateX(27vh);
    margin-right: 0;
  }
}

.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.4s;
}
.slide-up-enter, .slide-up-leave-to /* .fade-leave-active below version 2.1.8 */ {
  transform: translateY(100%);
}

.slide-fade-enter-active {
  transition: all 0.5s ease;
}
.slide-fade-leave-active {
  transition: all 0.5s cubic-bezier(1, 0.5, 0.8, 1);
}
.slide-fade-enter,
.slide-fade-leave-to {
  transform: translateX(27vh);
  opacity: 0;
}
</style>
