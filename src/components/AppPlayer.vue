<template>
  <div class="app-player">
    <div id="player" class="app-player__youtube"></div>

    <div class="app-player__videos" v-if="isShowPlayer">
      <ul class="app-player__list">
        <li class="app-player__list__item" v-for="(video, index) in videos" :key="video.id">
          <button class="app-player__video" @click="handleChangeVideo(video.id, index)">
            {{ video.title }}
          </button>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import {listVideos} from '@/mocs/videos.js';

const player = ref(null);
const videos = ref(listVideos || []);
const playerInstance = ref(null);
const isShowPlayer = ref(false);
const indexCurrentVideo = ref(0);

const storageName = 'VideoPlayer';
const urlYoutubeApi = 'https://www.youtube.com/iframe_api';

const onPlayerReady = () => {
  setVideoTime();
};

const setVideoTime = () => {
  let savedData = localStorage.getItem(storageName);
  const videoId = playerInstance.value.getVideoData()?.video_id || null;

  if(savedData && videoId) {
    savedData = JSON.parse(savedData);

    if(savedData[videoId]) {
      const currentTime = parseFloat(savedData[videoId]);
      playerInstance.value.seekTo(currentTime, true);
    }
  }
}

const handleChangeVideo = (videoId, index) => {
  indexCurrentVideo.value = index;
  changeVideo(videoId);
}

const changeVideo = (videoId, state = null) => {
  if (playerInstance.value) {
    if(state !== 'next') {
      saveCurrentTimeVideo();
    }

    playerInstance.value.loadVideoById(videoId);

    setTimeout(() => {
      setVideoTime();
    }, 400);
  }
};

const saveCurrentTimeVideo = (state = null) => {
  const id = playerInstance.value.getVideoData().video_id;
  const currentTime = playerInstance.value.getCurrentTime();

  let savedData = localStorage.getItem(storageName);
  let storageItem = null

  if(savedData !== null) {
    storageItem = JSON.parse(savedData);
  }

  storageItem = {
    ...storageItem,
    [id]: state === 'end' ? 0 : currentTime
  }

  localStorage.setItem(storageName, JSON.stringify(storageItem));
}

const initYoutubePlayer = () => {
  window.onYouTubeIframeAPIReady = () => {
    playerInstance.value = new YT.Player('player', {
      height: '390',
      width: '640',
      videoId: videos.value[0].id,
      events: {
        onReady: onPlayerReady,
        onStateChange: handleStateChanged,
      },
    });

    isShowPlayer.value = true;
  };
}

const playNextVideo = () => {
  if(indexCurrentVideo.value + 1 > videos.value.length - 1) {
    indexCurrentVideo.value = 0;
  } else {
    indexCurrentVideo.value += 1;
  }

  const nextVideoId = videos.value[indexCurrentVideo.value].id

  changeVideo(nextVideoId, 'next');
}

const handleStateChanged = (event) => {
  if (event.data === YT.PlayerState.ENDED) {
    saveCurrentTimeVideo('end');
    playNextVideo();
  }
}

const handleBeforeUnload = (event) => {
  saveCurrentTimeVideo();
  event.returnValue = '';
};

const loadScript = () => {
  const script = document.createElement('script');
  script.src = urlYoutubeApi;
  script.type = 'text/javascript';
  script.async = true;

  script.onload = () => {
    initYoutubePlayer();
  };

  script.onerror = () => {
    console.error('Ошибка при загрузке скрипта');
    location.reload();
  };

  document.head.appendChild(script);
}

onBeforeUnmount(() => {
  saveCurrentTimeVideo();
  window.removeEventListener('beforeunload', handleBeforeUnload);
})

onMounted(() => {
  loadScript();
  window.addEventListener('beforeunload', handleBeforeUnload);
});
</script>

<style lang="scss" scoped>
$playerHeight: 390px;

.app-player {
  display: grid;
  grid-template-columns: calc(100% - 300px) 300px;

  &__youtube {
    height: $playerHeight;
  }

  &__list {
    height: $playerHeight;
    display: flex;
    flex-direction: column;
    gap: .5rem;
    overflow: auto;
  }
}
</style>
