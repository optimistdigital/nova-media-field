<template>
  <div :class="`mime-type-icon mime-type-${mimeStartsWith}`">
    <img
      v-if="isImageFile && src && !isImageFileMissing"
      draggable="false"
      :src="src"
      @error="isImageFileMissing = true"
    />
    <missing-file-icon v-else-if="isImageFile && isImageFileMissing" />
    <audio-icon v-else-if="isAudioFile" />
    <thumbnail-video-icon v-else-if="isVideoFile && !showVideo" icon="video-icon" />
    <video v-else-if="isVideoFile && showVideo" controls>
      <source :src="src" :type="mimeType" />
    </video>
    <document-icon v-else class="p-2" />
  </div>
</template>
<script>
import AudioIcon from '../icons/AudioIcon';
import DocumentIcon from '../icons/DocumentIcon';
import MissingFileIcon from '../icons/MissingFileIcon';

export default {
  name: 'mime-type-icon',

  props: ['mimeType', 'src', 'showVideo'],

  components: {
    AudioIcon,
    DocumentIcon,
    MissingFileIcon,
  },

  data() {
    return {
      isImageFileMissing: false,
    };
  },

  computed: {
    mimeStartsWith() {
      const split = this.mimeType?.split('/');
      return split.length ? split[0] : null;
    },
    isImageFile() {
      return this.mimeType?.indexOf('image/') === 0;
    },
    isAudioFile() {
      return this.mimeType?.indexOf('audio/') === 0;
    },
    isVideoFile() {
      return this.mimeType?.indexOf('video/') === 0;
    },
  },
};
</script>

<style lang="scss" scoped>
.mime-type-icon {
  width: 100%;
}

video {
  width: 100%;
  max-height: 100px;
}
</style>
