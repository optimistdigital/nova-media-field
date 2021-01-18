<template>
  <od-modal ref="isModalOpen" v-if="isModalOpen" :name="'isModalOpen'" :align="'flex justify-end'" width="1315">
    <div slot="container">
      <div class="modal-header flex flex-wrap justify-between mb-6">
        <h2 class="text-90 font-normal text-xl">
          <span v-if="!uploadOnly">Browse media library</span>
          <span v-if="uploadOnly">Upload media</span>
          {{ currentCollection ? `(${currentCollectionData.label})` : '' }}
        </h2>

        <div class="collection-select" v-if="!uploadOnly">
          <div>
            <span>Search</span>
            <input
              class="w-full form-control form-input form-input-bordered"
              id="search"
              dusk="search"
              @input="onSearchInput"
            />
          </div>
          <div>
            <span>Collection</span>
            <select-control
              :id="collectionField.attribute"
              :dusk="collectionField.attribute"
              :fieldName="collectionField.fieldName"
              :options="collectionField.options"
              v-model="currentCollection"
              :disabled="displayCollection != null"
              class="w-full form-control form-select"
            >
              <option value="" selected>{{ __('All collections') }}</option>
            </select-control>
          </div>
        </div>
      </div>

      <media-modal-constraints
        :field="field"
        :currentCollectionData="currentCollectionData"
        :currentCollection="currentCollection"
      />

      <div :class="`flex mb-6`" id="media-dropzone" v-if="!loadingMediaFiles">
        <div class="img-collection" @scroll="scrollEventListener" ref="imgCollectionRef">
          <div class="empty-message" v-if="files.length === 0 && !uploadOnly">
            <p>There are currently no media files in this library</p>
            <p>Drag and drop files here to upload them</p>
          </div>

          <div class="empty-message" v-if="files.length === 0 && uploadOnly">
            <p>Drag and drop files here to upload them</p>
          </div>

          <uploaded-file
            v-for="file in fileList.filter(filterUploadedFiles)"
            :selected="stateSelectedFiles.find(item => item.processed && item.data.id === file.data.id) !== void 0"
            :active="file.processed && stateActiveFile && file.data.id === stateActiveFile.data.id"
            @click.native="toggleFileSelect(file)"
            :key="file.id"
            :file="file.processed ? file.data : {}"
            :progress="file.uploading ? file.uploadProgress : -1"
          />
        </div>

        <div class="image-editor">
          <edit-image v-if="stateActiveFile !== void 0" :file="stateActiveFile.data" />
        </div>

        <div
          :class="`input-dropzone-wrap ${draggingFile || (showUploadArea && listenUploadArea) ? 'visible' : ''} ${
            draggingFile && !draggingOverDropzone ? 'pulse' : ''
          }`"
        >
          <p>
            <svg class="fill-current w-4 h-4 mx-auto" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
              <path d="M13 8V2H7v6H2l8 8 8-8h-5zM0 18h20v2H0v-2z" />
            </svg>
          </p>
          <p>To upload files you can simply drag and drop them in the area or click it to open file browser.</p>
          <input
            type="file"
            ref="uploadFiles"
            name="media"
            class="input-dropzone"
            @change="fileBrowserSelectListener"
            multiple
          />
        </div>
      </div>

      <label v-if="showUploadArea || draggingFile" for="generate-thumbnails" class="flex">
        <input id="generate-thumbnails" type="checkbox" v-model="withThumbnails" autofocus/>
        <span class="ml-1">Generate thumbnails</span>
      </label>

      <div class="loader-container" v-if="loadingMediaFiles">
        <div class="loader" />
        <div class="small-loader" />
      </div>
    </div>
    <div slot="buttons" class="w-full flex">
      <button
        type="button"
        v-if="
          (uploadOnly && !(showUploadArea && listenUploadArea)) || !(showUploadArea && listenUploadArea) || draggingFile
        "
        v-on:click="openModalWithUpload"
        class="btn btn-default btn-primary whitespace-no-wrap"
      >
        {{ __('Upload files') }}
      </button>
      <button
        type="button"
        v-if="!(!(showUploadArea && listenUploadArea) || draggingFile)"
        v-on:click="showMediaLibrary"
        class="btn btn-default btn-primary whitespace-no-wrap"
      >
        {{ __('Back') }}
      </button>

      <div class="flex w-full justify-end">
        <button
          type="button"
          v-if="(!(showUploadArea && listenUploadArea) || draggingFile) && !uploadOnly"
          @click.prevent="closeModalAndSave"
          class="btn btn-default btn-primary mr-3"
        >
          {{ __('Apply and close') }}
        </button>
        <button type="button" @click.prevent="closeModal" class="btn btn-default btn-danger">
          {{ __('Close') }}
        </button>
      </div>
    </div>
  </od-modal>
</template>

<script>
import debounce from './../debounce';

export default {
  props: [
    'field',
    'isModalOpen',
    'chosenCollection',
    'activeFile',
    'selectedFiles',
    'updateMedia',
    'files',
    'multipleSelect',
    'loadingMediaFiles',
    'showUploadArea',
    'uploadOnly',
  ],

  data() {
    return {
      draggingOverDropzone: false,
      draggingFile: false,
      searchValue: '',
      listenUploadArea: false,
      stateActiveFile: void 0,
      stateSelectedFiles: [],
      withThumbnails: this.field.withThumbnails ?? true
    };
  },

  computed: {
    fileList() {
      return this.files.sort(this.sortByDate).sort(this.sortUploadedFiles);
    },

    currentCollection: {
      get: function () {
        return this.field.displayCollection || this.chosenCollection;
      },
      set: function (value) {
        this.$emit('update:chosenCollection', value);
      },
    },

    currentCollectionData: {
      get: function () {
        if (!this.currentCollection) return null;
        if (this.field.collections[this.currentCollection]) return this.field.collections[this.currentCollection];

        return {
          label: this.currentCollection.replace('_', ' ').replace('-', ' '),
          name: this.currentCollection,
          constraints: [],
        };
      },
    },

    displayCollection() {
      return this.field.displayCollection;
    },

    collectionField() {
      const { collections } = this.field;
      if (!collections || !Object.keys(collections).length) return false;
      return {
        attribute: 'display-collection',
        fieldName: 'display-collection',
        options: Object.keys(collections).map(collectionKey => {
          return {
            label: collections[collectionKey].label,
            value: collectionKey,
          };
        }),
      };
    },
  },

  watch: {
    isModalOpen(newVal, oldVal) {
      // watch it

      if (this.showUploadArea) {
        this.listenUploadArea = true;
      }

      if (newVal) {
        this.stateActiveFile = this.activeFile ? { ...this.activeFile } : void 0;
        this.stateSelectedFiles = Array.isArray(this.selectedFiles) ? [...this.selectedFiles] : [];
        this.addEventListeners();
      } else {
        this.clearEventListeners();
      }
    },
  },

  methods: {
    onSearchInput(event) {
      const oldValue = this.searchValue;
      const newValue = event.target.value;

      this.searchValue = newValue;
      if (oldValue === '' || (oldValue !== '' && newValue === '')) {
        this.$refs.imgCollectionRef.scrollTop = 0;
      }

      this.$emit('search', this.searchValue);
    },

    openModal() {
      this.$emit('update:isModalOpen', true);
    },

    closeModal() {
      this.listenUploadArea = false;
      this.$emit('update:isModalOpen', false);
    },

    closeModalAndSave() {
      this.listenUploadArea = false;
      this.$emit('update:selectedFiles', [...this.stateSelectedFiles]);
      this.$emit('update:activeFile', this.stateActiveFile ? { ...this.stateActiveFile } : void 0);
      this.$emit('update:isModalOpen', false);
    },

    filterUploadedFiles(file) {
      if (file.uploading || !file.processed) {
        return true;
      }

      return (
        !this.currentCollection || (file.data.collection_name && file.data.collection_name === this.currentCollection)
      );
    },
    sortByDate(a, b) {
      if (Date.parse(a.data.created_at) < Date.parse(b.data.created_at)) return 1;
      if (Date.parse(a.data.created_at) > Date.parse(b.data.created_at)) return -1;
      return 0;
    },
    sortUploadedFiles(a, b) {
      for (const selectedFile of this.selectedFiles) {
        if (selectedFile.data.id === a.data.id) {
          return -1;
        }
      }

      return 0;
    },

    fileBrowserSelectListener(e) {
      e.preventDefault();
      e.stopPropagation();

      this.draggingFile = false;
      this.draggingOverDropzone = false;

      for (const fileKey of Object.keys(this.$refs.uploadFiles.files)) {
        this.files.unshift({
          fileInput: this.$refs.uploadFiles.files[fileKey],
          collection: this.currentCollection || null,
          data: {},
          uploadProgress: 0,
          uploading: false,
          processed: false,
        });
      }

      this.listenUploadArea = false;
      this.$emit('update:files', this.files);
      this.uploadFiles();
    },

    dropEventListener(e) {
      e.preventDefault();
      e.stopPropagation();

      this.draggingFile = false;
      this.draggingOverDropzone = false;

      for (const fileKey of Object.keys(e.dataTransfer.files)) {
        this.files.unshift({
          fileInput: e.dataTransfer.files[fileKey],
          collection: this.currentCollection || null,
          data: {},
          uploadProgress: 0,
          uploading: false,
          processed: false,
        });
      }

      this.listenUploadArea = false;
      this.$emit('update:files', this.files);
      this.uploadFiles();
    },

    dragEndListener(e) {
      e.preventDefault();
      this.endDrag = true;
      this.setEndDrag();
    },

    dragLeaveListener(e) {
      e.preventDefault();

      if (e.target.matches('.input-dropzone')) {
        this.draggingOverDropzone = false;
      } else {
        this.endDrag = true;
        this.setEndDrag();
      }
    },

    dragOverListener(e) {
      e.preventDefault();
      this.draggingFile = true;
      this.endDrag = false;

      if (e.target.matches('.input-dropzone')) this.draggingOverDropzone = true;
    },

    scrollEventListener(e) {
      if (window.mediaLibrary.fetching) return;

      if (e.target.scrollTop + e.target.clientHeight >= e.target.scrollHeight - 300) {
        this.$emit('loadImages');
      }
    },

    setEndDrag: debounce(function () {
      if (this.endDrag) {
        this.draggingFile = false;
        this.draggingOverDropzone = false;
      }
    }, 1000),

    toggleFileSelect(file) {
      if (this.stateActiveFile && this.stateActiveFile.data.id === file.data.id) {
        this.stateActiveFile = void 0;

        if (!this.multipleSelect) {
          this.stateSelectedFiles = [];
          // this.$emit('update:stateSelectedFiles', this.stateSelectedFiles);
          return;
        }
      }

      if (
        this.multipleSelect &&
        !this.stateSelectedFiles.find(item => item.processed && item.data.id === file.data.id)
      ) {
        this.stateSelectedFiles.push(file);
      } else if (
        this.multipleSelect &&
        this.stateSelectedFiles.find(item => item.processed && item.data.id === file.data.id)
      ) {
        const i = this.stateSelectedFiles.findIndex(item => item.processed && +item.data.id === +file.data.id);

        if (i > -1) {
          this.stateSelectedFiles.splice(i, 1);
          // this.$emit('update:stateActiveFile', this.stateActiveFile);
          // this.$emit('update:stateSelectedFiles', this.stateSelectedFiles);
        }

        return;
      } else if (!this.multipleSelect) {
        this.stateSelectedFiles = [file];
      }

      this.stateActiveFile = file;
    },

    uploadFiles() {
      for (const fileInfo of this.files) {
        if (!fileInfo.fileInput || fileInfo.uploading || fileInfo.processed) {
          continue;
        }

        const form = new FormData();

        form.append('withThumbnails', this.withThumbnails)
        form.append('file', fileInfo.fileInput);

        if (fileInfo.collection) {
          form.append('collection', fileInfo.collection);
        }

        fileInfo.uploading = true;

        axios
          .post('/api/media/upload', form, {
            headers: {
              'Content-Type': 'multipart/form-data',
            },
            onUploadProgress: e => {
              fileInfo.uploadProgress = (e.loaded / e.total) * 100;
            },
          })
          .then(response => {
            if (this.uploadOnly) {
              this.$emit('updateFiles', {
                uploading: false,
                processed: true,
                data: response.data,
              });
            } else {
              window.mediaLibrary.files.unshift({
                uploading: false,
                processed: true,
                data: response.data,
              });

              this.$emit('update:files', window.mediaLibrary.files);

              this.updateMedia();
            }
          })
          .catch(error => {
            if (!error.response) {
              Nova.$emit('error', 'Failed to upload image.');
              this.updateMedia();
              return;
            }

            const response = error.response.data;

            if (Array.isArray(response && response.errors && response.errors.file)) {
              Nova.$emit('error', response.errors.file[0]);
            } else if (response.message) {
              Nova.$emit('error', response.message);
            } else {
              Nova.$emit('error', 'Failed to upload image.');
            }

            if (window.mediaLibrary && window.mediaLibrary.files) {
              window.mediaLibrary.files.splice(0, 1);
            }

            this.updateMedia();
          });
      }
    },

    addEventListeners() {
      window.addEventListener('dragover', this.dragOverListener);
      window.addEventListener('dragleave', this.dragLeaveListener);
      window.addEventListener('dragend', this.dragEndListener);
      window.addEventListener('drop', this.dropEventListener);
    },

    clearEventListeners() {
      window.removeEventListener('dragover', this.dragOverListener);
      window.removeEventListener('dragleave', this.dragLeaveListener);
      window.removeEventListener('dragend', this.dragEndListener);
      window.removeEventListener('drop', this.dropEventListener);
    },

    openModalWithUpload() {
      this.listenUploadArea = true;
      this.$emit('update:isModalOpen', true);
      this.$emit('update:showUploadArea', true);
    },

    showMediaLibrary() {
      this.listenUploadArea = false;
      this.$emit('update:isModalOpen', true);
      this.$emit('update:showUploadArea', false);
    },
  },
};
</script>
