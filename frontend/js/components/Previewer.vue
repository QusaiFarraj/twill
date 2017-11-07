<template>
  <div class="previewer" :class="{ 'previewer--loading' : loading }">
    <a17-button @click="restoreRevision" v-if="activeRevision" class="previewer__restore" variant="warning" size="small">Restore</a17-button>
    <div class="previewer__frame">
      <div class="previewer__inner">
        <div class="previewer__nav">
          <div class="previewer__revisions">
            <span class="tag tag--revision" v-if="slipScreen">Past</span>
            <a17-dropdown ref="previewRevisionsDropdown" position="bottom-left" :maxWidth="400">
              <a17-button class="previewer__trigger" @click="$refs.previewRevisionsDropdown.toggle()">
                <template v-if="activeRevision" >
                  {{ currentRevision.datetime | formatDate }} ({{ currentRevision.author }}) <span v-svg symbol="dropdown_module"></span>
                </template>
                <template v-else>
                  Last edited <timeago :auto-update="1" :since="new Date(revisions[0].datetime)"></timeago> <span v-svg symbol="dropdown_module"></span>
                </template>
              </a17-button>
              <div slot="dropdown__content">
                <button type="button" class="previewerRevision" @click="previewRevision(revision.id)" v-for="(revision, index) in revisions"  :key="revision.id">
                  <span class="previewerRevision__author">{{ revision.author }}</span>
                  <span class="previewerRevision__datetime"><span class="tag" v-if="index === 0">Current</span> {{ revision.datetime | formatDate }}</span>
                </button>
              </div>
            </a17-dropdown>
          </div>

          <ul class="previewer__breakpoints" v-if="!slipScreen">
            <li v-for="(breakpoint, index) in breakpoints" :key="breakpoint.size" class="previewer__breakpoint" :class="{ 's--active' : activeBreakpoint === breakpoint.size }" @click="resizePreview(breakpoint.size)">{{ breakpoint.name }}</li>
          </ul>

          <div class="previewer__compare" v-if="activeRevision">
            <a href="#" v-if="!slipScreen" @click.prevent="compareView">Compare view</a>
            <a href="#" v-if="slipScreen" @click.prevent="singleView">Single view</a>
          </div>
        </div>

        <div class="previewer__content">
          <div class="previewer__iframe">
            <a17-iframe :content="activeRevision ? activeContent : currentContent" :size="activeBreakpoint" @scrollDoc="setIframeScroll" :scrollPosition="scrollPosition"></a17-iframe>
          </div>
          <div class="previewer__iframe" v-if="slipScreen">
            <div class="previewer__iframeInfos"><span class="tag tag--revision">Current</span>{{ revisions[0].datetime  | formatDate }} ({{ revisions[0].author }})</div>
            <a17-iframe :content="currentContent" @scrollDoc="setIframeScroll" :scrollPosition="scrollPosition"></a17-iframe>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  // nb : UI is quite similar to https://github.com/nerijusgood/viewport-resizer

  import { mapState } from 'vuex'
  import A17PreviewerFrame from '@/components/PreviewerFrame.vue'
  import a17VueFilters from '@/utils/filters.js'

  export default {
    name: 'A17previewer',
    components: {
      'a17-iframe': A17PreviewerFrame
    },
    data: function () {
      return {
        loadedCurrent: false,
        slipScreen: false,
        activeBreakpoint: 1024,
        lastActiveBreakpoint: 1024,
        scrollPosition: 0,
        breakpoints: [
          {
            size: 320,
            name: 'Mobile'
          },
          {
            size: 768,
            name: 'Tablet'
          },
          {
            size: 1024,
            name: 'Desktop'
          },
          {
            size: 1280,
            name: 'Wide'
          }
        ]
      }
    },
    filters: a17VueFilters,
    computed: {
      activeRevision: function () {
        return Object.keys(this.currentRevision).length
      },
      ...mapState({
        loading: state => state.revision.loading,
        currentRevision: state => state.revision.active,
        activeContent: state => state.revision.activeContent,
        currentContent: state => state.revision.currentContent,
        revisions: state => state.revision.all
      })
    },
    methods: {
      getCurrentPreview: function () {
        if (this.loadedCurrent) return

        this.loadedCurrent = true
        this.$store.dispatch('getCurrentContent')
      },
      restoreRevision: function () {
        // Do something here
      },
      resizePreview: function (size) {
        this.activeBreakpoint = parseInt(size)
        this.lastActiveBreakpoint = parseInt(size)
      },
      previewRevision: function (id) {
        this.$store.commit('updateRevision', id)

        // Update preview HTML
        this.$store.dispatch('getRevisionContent')
      },
      compareView: function () {
        this.activeBreakpoint = 0
        this.slipScreen = true

        if (this.activeRevision) this.getCurrentPreview()
      },
      singleView: function () {
        this.activeBreakpoint = this.lastActiveBreakpoint
        this.slipScreen = false
      },
      setIframeScroll: function (value) {
        this.scrollPosition = value
      }
    },
    mounted: function () {
      // get preview HTML
      if (this.activeRevision) this.$store.dispatch('getRevisionContent')
      else this.getCurrentPreview()
    }
  }
</script>

<style lang="scss" scoped>
  @import "../../scss/setup/variables.scss";
  @import "../../scss/setup/colors.scss";
  @import "../../scss/setup/mixins.scss";

  $height__nav: 80px;

  .previewer {
    display: block;
    width: 100%;
    padding: 0;
    position:relative;
    flex-grow:1;
    background-color:$color__text;
  }

  .previewer__restore {
    position:fixed;
    right:20px;
    top:13px;
    z-index:$zindex__overlay + 1;
  }

  .tag--revision {
    color:$color__text;
    position:absolute;
    top: 17px;
    left: 0;
    margin: 0;
    opacity:0.5;
  }

  .previewer__nav {
    display: flex;
    flex-direction: row;
    height:$height__nav;
    opacity:1;
    transition: opacity .3s ease;
  }

  .previewer__frame {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    display:flex;
    flex-flow: column nowrap;
  }

  .previewer__inner {
    position: relative;
    width: 100%;
    overflow: hidden;
    flex-grow: 1;
    display:flex;
    flex-flow: column nowrap;
  }

  .previewer__trigger {
    height:auto;
    line-height: inherit;
  }

  .previewer__trigger,
  .previewer__compare {
    color:$color__text--light;
    padding-left:0;
    padding-right:0;

    &:hover,
    &:focus {
      color:$color__background;
    }

    a {
      text-decoration:none;
    }

    .icon {
      margin-left:6px;
    }
  }

  .previewer__revisions,
  .previewer__compare {
    margin-left:20px;
    margin-right:20px;
    padding-top: 40px;
  }

  .previewer__revisions {
    padding-top: 40px;
    flex-grow:1;
    position:relative;
  }

  .previewer__breakpoints {
    margin: 0 auto;
    position:absolute;
    top: 0;
    left: 50%;
    transform:translateX(-50%);
    height:$height__nav;
    line-height:$height__nav;
  }

  .previewer__breakpoint {
    cursor:pointer;
    display: inline;
    color:$color__text--light;
    padding:0 20px;

    &.s--active {
      color:$color__background;
    }
  }

  .previewer__content {
    width: 100%;
    height: 100%;
    display:flex;
    flex-grow:1;
    flex-flow: row nowrap;
  }

  .previewer__iframe {
    width: 100%;
    opacity:1;
    transition: opacity .3s ease, width .3s ease;
    position: relative;
    display: flex;
    flex-grow: 1;
  }

  .previewer--loading {
    .previewer__nav,
    .previewer__iframe {
      opacity:0;
      pointer-events: none;
    }

    .previewer__content {
      &::after {
        content: 'Loading preview...';
        position:absolute;
        top:25%;
        left:50%;
        width:200px;
        margin-left:-100px;
        text-align:center;
        color:$color__text--light;
      }
    }
  }

  .previewer__iframeInfos {
    height:80px;
    margin-top:-80px;
    position:absolute;
    color:$color__text--light;
    top:0;
    left:10px;
    padding-top:40px
  }

  button.previewerRevision {
    display:flex;
    padding:0 15px;
  }

  .previewerRevision__author {
    padding-right:30px;
    flex-grow: 1;
  }

  .previewerRevision__datetime {
    color:$color__link;
  }
</style>