<template>
  <div class="mime-type-icon-wrapper">
    <mime-type-icon :src="fileThumbnail" :mime-type="mimeType" />
  </div>
</template>
<script>
import MissingFileIcon from '../icons/MissingFileIcon';
import DocumentIcon from '../icons/DocumentIcon';
import MimeTypeIcon from './MimeTypeIcon';
export default {
  name: 'index-file-field',
  components: { MimeTypeIcon, DocumentIcon, MissingFileIcon },
  props: ['field'],

  data() {
    const isImageFile = this.field.value.mime_type.indexOf('image/') === 0;
    return {
      fileIsMissing: false,
      isImageFile,
    };
  },

  computed: {
    file() {
      return this.field?.value || {};
    },
    previewImg() {
      return this.file.image_sizes?.thumbnail?.url || '';
    },
    mimeType() {
      return this.file?.mime_type;
    },
    fileThumbnail() {
      if (this.file?.mime_type?.indexOf('video') === 0) return this.file.data.thumbnail || '';
      if (this.file?.mime_type?.indexOf('image') === 0) {
        if (this.file?.image_sizes?.thumbnail) return this.file.image_sizes.thumbnail.url;
      }
      return '';
    },
  },
};
</script>
<style lang="scss">
.mime-type-icon-wrapper {
  display: flex;
  justify-content: center;
  align-content: center;
  align-items: center;

  .mime-type-icon {
    width: 100%;
    max-width: 50px;
  }

  .mime-type-video {
    padding: 6px;
  }

  .mime-type-audio {
    padding: 8px;
  }

  img,
  svg {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }
}
</style>
