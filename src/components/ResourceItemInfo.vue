<template>
  <div class="resource-item-info width-limited" v-if="resourceItem">
    <section style="margin-bottom:10px;" v-if="resourceItem.apps">
      <app-icons :apps="resourceItem.apps"></app-icons>
      &nbsp;&nbsp;<badges :badges="resourceItem.badges"></badges>
    </section>
    <section style="text-align:center;">
      <b-carousel
        style="max-width: 512px;"
        v-if="resourceItem.cover_images && resourceItem.cover_images.length > 0"
        :indicator="resourceItem.cover_images.length > 1"
        :arrow="resourceItem.cover_images.length > 1"
        :pause-info="false"
      >
        <b-carousel-item
          v-for="cover in resourceItem.cover_images"
          :key="cover"
        >
          <figure class="image is-16by9">
            <img
              loading="lazy"
              :src="cover"
              :alt="resourceItem.name"
              class="cover-image"
              data-target="modal-image2"
            />
          </figure>
        </b-carousel-item>
      </b-carousel>
    </section>

    <span style="margin-top:3px;display: block;">
      <span v-for="t in resourceItem.tags" :key="t">
        <b-tag style="cursor: pointer;" rounded>{{ t }}</b-tag>
      </span>
    </span>
    <p v-if="resourceItem.description">
      {{ resourceItem.description.slice(0, maxDescriptionLetters) }}
      <a
        v-if="resourceItem.description.length > maxDescriptionLetters"
        @click="maxDescriptionLetters = resourceItem.description.length"
        >...show all.</a
      >
    </p>
    <p
      class="authors"
      v-if="resourceItem.authors && resourceItem.authors.length > 0"
    >
      Author(s):
      <b-tooltip
        v-for="author in resourceItem.authors"
        :key="author.name"
        :label="author.affiliation"
        position="is-bottom"
      >
        <span>{{ author.name }}</span>
      </b-tooltip>
    </p>
    <br />

    <!-- <attachments
      :attachments="resourceItem.attachments"
      :focusTarget="resourceItem._focus"
    ></attachments> -->

    <div class="markdown-body width-limited">
      <h1 v-if="resourceItem.docs">Documentation</h1>
      <markdown
        v-if="resourceItem.docs"
        :baseUrl="resourceItem.baseUrl"
        :content="resourceItem.docs.slice(0, maxDocsLetters)"
      ></markdown>

      <a
        v-if="resourceItem.docs && resourceItem.docs.length > maxDocsLetters"
        style="color: #0366d6;"
        @click="maxDocsLetters = resourceItem.docs.length"
        >+ click here to see the full documentation</a
      >

      <br />
      <h3 v-if="formatedCitation" id="citation">Citation</h3>
      <ul v-if="formatedCitation" class="citation">
        <li v-for="c in formatedCitation" :key="c.text">
          {{ c.text }}
          <a v-if="c.url" :href="c.url" target="_blank">[{{ c.url_text }}]</a>
        </li>
      </ul>
    </div>
    <!-- <comment-box :title="resourceItem.name"></comment-box> -->
  </div>
  <h2 v-else>Resource not found: {{ resourceId }}</h2>
</template>

<script>
import { mapState } from "vuex";
import Badges from "@/components/Badges.vue";
import AppIcons from "@/components/AppIcons.vue";
// import Attachments from "@/components/Attachments.vue";
import Markdown from "@/components/Markdown.vue";
// import CommentBox from "@/components/CommentBox.vue";
import { randId } from "../utils";

export default {
  name: "ResourceItemInfo",
  props: ["resourceId"],
  components: {
    markdown: Markdown,
    badges: Badges,
    // attachments: Attachments,
    "app-icons": AppIcons
    // "comment-box": CommentBox
  },
  data() {
    return {
      resourceItem: null,
      maxDescriptionLetters: 100,
      maxDocsLetters: 500,
      showSource: false
    };
  },
  mounted() {
    this.init();
  },
  computed: {
    formatedCitation: function() {
      let cites = this.resourceItem.cite;
      if (!cites) return null;
      if (this.resourceItem.cite && !Array.isArray(this.resourceItem.cite)) {
        cites = [this.resourceItem.cite];
      }
      const citations = [];
      for (let c of cites) {
        if (typeof c === "string") {
          citations.push({
            text: c
          });
        } else {
          let url = c.url;
          let url_text = "link";
          if (c.doi) {
            if (c.doi.startsWith("http")) url = c.doi;
            else url = "https://doi.org/" + c.doi;
            url_text = "doi";
          }
          citations.push({
            text: c.text,
            url,
            url_text
          });
        }
      }
      return citations;
    },
    ...mapState({
      siteConfig: state => state.siteConfig,
      loadedUrl: state => state.loadedUrl,
      resourceItems: state => state.resourceItems
    })
  },
  methods: {
    async init() {
      if (!this.loadedUrl) {
        const repo = this.siteConfig.rdf_root_repo;
        let manifest_url = this.siteConfig.manifest_url;
        await this.$store.dispatch("fetchResourceItems", {
          repo,
          manifest_url
        });
      }
      const resourceItem = this.resourceItems.filter(item => {
        return item.id === this.resourceId;
      })[0];
      if (!resourceItem) {
        alert("Item not found: " + this.resourceId);
        throw new Error("Item not found: " + this.resourceId);
      }
      this.resourceItem = resourceItem;
      const focus = () => {
        if (this.resourceItem._focus) {
          const el = document.getElementById(this.resourceItem._focus);
          if (el) {
            el.parentNode.scrollTop = el.offsetTop - 40;
          }
        }
      };
      if (this.resourceItem.documentation)
        this.getDocs(this.resourceItem).then(focus);
    },
    async getDocs(resourceItem) {
      resourceItem.docs = "@loading...";
      try {
        let docsUrl;
        if (!resourceItem.documentation.startsWith("http"))
          docsUrl = new URL(resourceItem.documentation, resourceItem.root_url)
            .href;
        else {
          docsUrl = resourceItem.documentation;
        }
        if (docsUrl.includes("github.")) docsUrl = docsUrl + "?" + randId();
        const response = await fetch(docsUrl);
        if (response.status == 200) {
          const raw_docs = await response.text();
          let baseUrl;
          if (!this.resourceItem.documentation.startsWith("http")) {
            const temp = (
              this.resourceItem.root_url +
              "/" +
              this.resourceItem.documentation
            ).split("/");
            baseUrl = temp.slice(0, temp.length - 1).join("/");
          } else {
            const temp = this.resourceItem.documentation.split("/");
            baseUrl = temp.slice(0, temp.length - 1).join("/");
          }
          if (resourceItem.documentation.endsWith(".md")) {
            resourceItem.baseUrl = baseUrl;
            resourceItem.docs = raw_docs;
          } else if (resourceItem.documentation) {
            resourceItem.docs = `### [Documentation](${resourceItem.documentation})`;
          }
        } else {
          resourceItem.docs = null;
        }
        this.$forceUpdate();
      } catch (e) {
        resourceItem.docs = null;
        this.$forceUpdate();
      } finally {
        if (this.resourceItem.docs)
          this.maxDocsLetters = this.resourceItem.docs
            .split("\n")
            .slice(0, 5)
            .join("\n").length;
      }
    }
  }
};
</script>
<style scoped>
.resource-item-info {
  padding: 20px;
  height: calc(100% - 50px);
  overflow: auto;
  overscroll-behavior: contain;
}

.width-limited {
  max-width: 1080px;
  margin-left: auto !important;
  margin-right: auto !important;
  float: none !important;
}

@media screen and (max-width: 768px) {
  .resource-item-info {
    padding: 5px;
  }
}
.card-image {
  max-height: 500px;
}
.citation {
  list-style-type: circle;
}
.cover-image {
  object-fit: contain;
}
.badge {
  display: inline-block;
  padding: 1px;
  margin-top: -5px;
  margin-bottom: -5px;
}
.tags:not(:last-child) {
  margin-bottom: -10px;
}
</style>
