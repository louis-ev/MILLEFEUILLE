<template>
  <div
    class="mediaContainer"
    :class="`type-${media.type}`"
    :data-context="context"
  >

    <template v-if="media.type === 'image'">
      <img :src="linkToImageThumb">
      <transition name="fade" :duration="600">
        <img v-if="is_hovered && $root.state.is_electron" :src="linkToHoveredThumb">
      </transition>
    </template>

    <template v-else-if="media.type === 'video'">
      <template v-if="context === 'preview'">
        <img :src="linkToVideoThumb">
      </template>
      <template v-else>
        <video ref="video" preload="none" :src="mediaURL" :poster="linkToVideoThumb" controls autoplay />
        <svg 
          ref="playIcon" 
          v-if="!video_is_playing"
          class="mediaContainer--videoPlay" 
          viewBox="0 0 200 200" 
          alt="Play video"
          @click="togglePlayVideo()"
        >
          <circle cx="100" cy="100" r="90" fill="#fff" stroke-width="15" stroke="#fff"></circle>
          <polygon points="70, 55 70, 145 145, 100" fill="#353535"></polygon>
        </svg>
      </template>
    </template>

    <template v-else-if="media.type === 'audio'">
      <template v-if="context === 'preview'">
        <audio controls preload="none" :src="mediaURL"  />
      </template>
      <template v-else>
        <audio controls preload="none" :src="mediaURL" autoplay />
      </template>
    </template>

    <template v-else-if="media.type === 'text'">
      <div v-if="context !== 'edit'" class="">
        <div v-if="value.length !== 0" v-html="value" />
        <p v-else v-html="'…'" />
      </div>
      <VueEditor 
        v-else
        v-model="htmlForEditor"
        ref="textField"
        autocorrect="off"
        :editorToolbar="customToolbar"
        class="mediaTextContent"
        autofocus
      />
      <!-- <textarea
        placeholder="…"
        class="mediaTextContent border-none bg-transparent"
        :value="value"
        @input="$emit('input', $event.target.value)"
        ref="textField"
        autocorrect="on"
        :readonly="read_only"
      /> -->
    </template>
    <template v-else-if="media.type === 'document'">
      <div v-if="context !== 'edit'" class="">
        <pre>
  {{ media.media_filename }}
        </pre>
      </div>
      <iframe v-else :src="mediaURL" />
    </template>

    <template v-else-if="media.type === 'other' && media.media_filename.indexOf('.') > -1">
      <div class="padding-small font-small">
        <pre>
<span v-html="$t('file:')">
</span>
{{ media.media_filename }}
        </pre>
      </div>
    </template>

  </div>
</template>
<script>
import { VueEditor } from 'vue2-editor';

// is loaded by Media and by EditMedia

export default {
  props: {
    slugFolderName: String,
    media: Object,
    subfolder: {
      type: String,
      default: ''
    },
    context: {
      type: String,
      default: 'preview'
      // preview, edit, publication
    },
    value: {
      type: String,
      default: '…'
    },
    is_hovered: Boolean,
    read_only: {
      type: Boolean,
      default: true
    },
    preview_size: {
      type: Number,
      default: 360
    }
  },
  components: {
    VueEditor
  },
  data() {
    return {
      available_resolutions: {
        preview_hovered: 600,
        default: 1600
      },
      video_is_playing: true,
      htmlForEditor: this.value,
      customToolbar: [
        [{ 'header': [false, 1, 2, 3, 4] }],
        // [{ 'header': 1 }, { 'header': 2 }, { 'header': 3 }, { 'header': 4 }],
        ['bold', 'italic', 'underline', 'link', 'blockquote'],
        [{ list: 'ordered' }, { list: 'bullet'} ],
        ['clean']  
      ]
    };
  },
  mounted() {
    if (this.context === 'edit') {
      if (Modernizr !== undefined && !Modernizr.touchevents) {
        if (this.$refs.textField !== undefined) {
          // this.$refs.textField.focus();
        }
      }
    }
  },
  beforeDestroy() {
  },
  watch: {
    'htmlForEditor': function() {
      this.$emit('input', this.htmlForEditor);
    }
  },
  computed: {
    mediaURL: function() {
      return this.$root.state.mode === 'export_publication' 
        ? `./${this.subfolder}${this.slugFolderName}/${this.media.media_filename}` 
        : `/${this.subfolder}${this.slugFolderName}/${this.media.media_filename}`;
    },
    thumbRes: function() {
      return this.context === 'preview'
        ? this.preview_size
        : this.available_resolutions.default;
    },
    thumbResHovered: function() {
      return this.available_resolutions.preview_hovered;
    },
    linkToImageThumb: function() {
      if(!this.media.hasOwnProperty('thumbs')) {
        return this.mediaURL;
      }

      let pathToSmallestThumb = this.$_.findWhere(this.media.thumbs, {
        size: this.thumbRes
      });

      if (
      // if image is gif and context is not 'preview', let’s show the original gif
        (this.context !== 'preview' &&
        this.mediaURL.toLowerCase().endsWith('.gif'))
        ||
        pathToSmallestThumb === undefined
      ) {
        return this.mediaURL;
      }

      const fullPathToThumb = pathToSmallestThumb.path;
      let url = this.$root.state.mode === 'export_publication' ? `./${fullPathToThumb}` : `/${fullPathToThumb}`;
      url += `?${(new Date()).getTime()}`;
      return url;
    },
    linkToHoveredThumb: function() {
      let pathToSmallestThumb = this.$_.findWhere(this.media.thumbs, {
        size: this.thumbResHovered
      }).path;

      const url = this.$root.state.mode === 'export_publication' ? './' + pathToSmallestThumb : '/' + pathToSmallestThumb;
      return pathToSmallestThumb !== undefined
        ? url
        : this.mediaURL;
    },
    linkToVideoThumb: function() {
      if (!this.media['thumbs'] || typeof this.media.thumbs === 'object' && this.media.thumbs.length === 0) {
        return;
      }

      let timeMark = 0;
      let timeMarkThumbs = this.$_.findWhere(this.media.thumbs, { timeMark });

      if (!timeMarkThumbs || timeMarkThumbs.length === 0) {
        return;
      }

      let pathToSmallestThumb = this.$_.findWhere(timeMarkThumbs.thumbsData, {
        size: this.thumbRes
      }).path;

      let url = this.$root.state.mode === 'export_publication' ? './' + pathToSmallestThumb : '/' + pathToSmallestThumb;
      url += `?${(new Date()).getTime()}`;
      return pathToSmallestThumb !== undefined
        ? url
        : this.mediaURL;
    }
  },
  methods: {
    togglePlayVideo() {
      if(this.video_is_playing === false) {
        this.video_is_playing = true;
        this.$refs.video.play();
        this.$refs.video.setAttribute('controls', 'controls')      
      }
    }
  }
};
</script>
<style scoped style="sass">
.mediaContainer--videoPlay {
  width: 100%;
  height: 100%;
  padding: 10px calc(50% - 50px);
  position: absolute;
  top: 0;
  left: 0;
  display: block;
  opacity: 0.95;
  cursor: pointer;
  background-image: linear-gradient(transparent, rgba(0,0,0,.1));
  transition: opacity 150ms;
}
.mediaContainer--videoPlay.is--hidden {
  opacity: 0;
}

</style>