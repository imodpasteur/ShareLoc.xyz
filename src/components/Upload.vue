<template>
  <div class="upload">
    <b-notification
      v-if="client.isSandbox"
      type="is-warning"
      has-icon
      aria-close-label="Close notification"
      role="alert"
    >
      You are using the development mode of the upload feature, this means files
      will be uploaded to the sandbox version of Zenodo
      (https://sandbox.zenodo.org). The uploaded files can be removed from
      Zenodo at any time without notice. This is temporary. In the future, the
      upload feature will connect to the main Zenodo storage and allow permanent
      storage of your data.
    </b-notification>
    <b-steps
      style="margin-top: 20px;"
      class="width-limited"
      position="left"
      :has-navigation="false"
      v-model="stepIndex"
      label-position="right"
    >
      <b-step-item :disabled="rdfYaml" label="Start" icon="file">
        <b-button
          v-if="imjoy"
          style="text-transform:none;"
          class="button is-fullwidth"
          @click="gettingStarted"
          expanded
          >New to ShareLoc.XYZ? Watch the Step-by-step Guide</b-button
        >
        <br />
        <b-field
          v-if="!client.credential"
          label="Please login or sign up to Zenodo.org"
          message="ShareLoc.XYZ uses https://zenodo.org as storage service, you will need to sign up or login to Zenodo, and allow ShareLoc.XYZ to upload files to zenodo on your behalf."
          expanded
        >
          <b-button
            style="text-transform:none;"
            class="button is-fullwidth is-primary"
            @click="login()"
            expanded
            icon-left="login"
            >Login to Zenodo</b-button
          >
        </b-field>
        <b-field
          v-else
          label="You have already logged in via Zenodo"
          message="ShareLoc.XYZ uses https://zenodo.org as storage service, you will need to sign up or login to Zenodo, and allow ShareLoc.XYZ to upload files to zenodo on your behalf."
          expanded
        >
          <b-button
            style="text-transform:none;"
            class="button is-small"
            @click="client.logout()"
            icon-left="logout"
            >Logout</b-button
          >
        </b-field>
        <b-field
          v-if="client.credential"
          label="Option 1: Create a new deposit"
          expanded
          message="With this option, you can upload a dataset or application to ShareLoc (via Zenodo), please make sure you have prepared a set of files. Also note that for each dataset can contain files with the total size up to 50GB, if you have more than that, you should either split the files into several deposits or contact Zenodo to increase the quota."
        >
          <b-button
            style="text-transform:none;"
            class="button is-fullwidth"
            @click="startUpload"
            expanded
            :disabled="!client.credential"
            >Start Upload</b-button
          >
        </b-field>
        <b-field
          v-if="client.credential"
          label="Option 2: Update an existing deposit"
          message="With this option, you can update an existing dataset or application. A Zenodo DOI or URL should be provided."
        >
          <b-input
            type="url"
            placeholder="Type a DOI or URL here"
            v-model="URI4Load"
          >
          </b-input>
        </b-field>

        <b-button
          v-if="client.credential"
          style="text-transform:none;"
          class="button is-fullwidth"
          @click="loadRdfFromURL(URI4Load)"
          expanded
          :disabled="!client.credential"
          >Load</b-button
        >
        <br />
        <b-notification aria-close-label="Close notification">
          <h1>
            You can also create an application card for your software tool!
          </h1>
          <p>
            If you developed a software tool for facilitating data acquisiton or
            analysis of SMLM data, please consider adding an `application` card
            to ShareLoc.XYZ so people who used your tool can link their datasets
            to your tool. You can also make an
            <a target="_blank" href="https://imjoy.io">ImJoy</a> plugin for your
            tool so users can directly run it from within ShareLoc. For detailed
            instructions on how to contribute an application, please
            <a
              target="_blank"
              href="https://github.com/imodpasteur/ShareLoc.XYZ/blob/main/docs/contribute-applications.md"
              >read here</a
            >.
          </p>
        </b-notification>
      </b-step-item>

      <b-step-item label="Edit" icon="pencil" :disabled="!rdfYaml">
        <section v-if="stepIndex == 1">
          <upload-form @submit="submitRDF" :init-rdf="rdf"></upload-form>
        </section>
      </b-step-item>

      <b-step-item label="Upload" icon="upload">
        <b-field
          label="Upload Summary"
          style="height: 260px; overflow: auto;"
          v-if="rdfYaml"
        >
          <markdown
            v-if="rdfYaml"
            baseUrl=""
            :content="formatedModelYaml"
          ></markdown>
        </b-field>
        <b-field
          v-if="editedFiles"
          label="Files"
          message="These files will be uploaded or updated"
        >
          <b-taglist attached rounded>
            <b-tag
              v-for="file in editedFiles"
              :key="file.sampleName + '/' + file.name"
              rounded
              >{{ file.sampleName + "/" + file.name }}</b-tag
            >
          </b-taglist>
        </b-field>
        <br />
        <div v-if="similarDeposits && similarDeposits.length > 0">
          <label class="label">Similar Existing Items</label>
          <p>
            The following published deposit(s) are similar to yours (matched by
            name), please make sure you are using distinctive names to avoid
            confusion to the users.
          </p>
          <b-notification
            v-for="item in similarDeposits"
            :key="item.id"
            :type="item.name === rdf.name ? 'is-danger' : null"
            aria-close-label="Close notification"
          >
            <h1>
              <a :href="item.config._deposit.links.html" target="_blank">{{
                item.name
              }}</a>
            </h1>
            <p>{{ item.description.slice(0, 200) }}</p>

            <p>
              Authors:
              {{
                item.authors &&
                  item.authors
                    .map(author => author.name.split(";")[0])
                    .join(",")
              }}
            </p>
            <p>Uploaded: {{ item.config._deposit.updated }}</p>
            <br />
            <div class="columns">
              <b-button
                v-if="
                  userId &&
                    item.config._deposit &&
                    item.config._deposit.owners.includes(userId)
                "
                @click="createOrUpdateDeposit(item.config._deposit.id, false)"
                class="column button is-primary is-light is-fullwidth"
                expanded
                icon-left="autorenew"
              >
                <span>Update as a new version</span>
              </b-button>
            </div>
          </b-notification>
          <b-button
            style="text-transform:none;"
            class="button is-fullwidth"
            @click="stepIndex = 1"
            expanded
            :class="{
              'is-primary': sameNameDeposits && sameNameDeposits.length > 0
            }"
            icon-left="arrow-left"
            >Go back to rename</b-button
          >
        </div>
        <br />
        <b-field>
          <b-switch v-model="requestedJoinCommunity">
            Apply for listing in the
            <a
              :href="
                client.baseURL +
                  '/communities/' +
                  siteConfig.zenodo_config.community
              "
              target="_blank"
              >ShareLoc.XYZ community list</a
            >
          </b-switch>
        </b-field>
        <p v-if="uploadStatus">{{ uploadStatus }}</p>
        <b-progress
          v-if="uploadProgress"
          type="is-primary"
          :value="uploadProgress"
          size="is-small"
          expanded
        >
        </b-progress>
        <div class="columns">
          <div v-if="client && editedFiles" class="column">
            <b-button
              :disabled="
                uploadProgress ||
                  (sameNameDeposits && sameNameDeposits.length > 0)
              "
              @click="createOrUpdateDeposit()"
              class="button is-primary is-light is-fullwidth"
              expanded
              icon-left="plus"
            >
              <span>Upload as new deposit</span>
            </b-button>
          </div>
          <div v-if="client && editedFiles && depositId" class="column">
            <b-button
              :disabled="uploadProgress"
              @click="createOrUpdateDeposit(depositId, false)"
              class="button is-primary is-light is-fullwidth"
              expanded
              icon-left="autorenew"
            >
              <span>Add new version to deposit</span>
            </b-button>
          </div>
        </div>
      </b-step-item>
      <b-step-item label="Publish" icon="share-variant" disabled>
        <b-notification
          v-if="publishedUrl"
          type="is-success"
          has-icon
          aria-close-label="Close notification"
        >
          <h1>Published on Zenodo</h1>
          <h2>DOI: {{ publishedDOI }}</h2>
          <h2>
            <a :href="publishedUrl" target="_blank">{{ publishedUrl }}</a>
          </h2>
          <p>
            Note: Newly uploaded item may not appear immediately in the resource
            list.
          </p>
          <p v-if="requestedJoinCommunity">
            To be listed as part of the verified ShareLoc.XYZ community list, a
            notification will be sent to the admin team aand we will review
            request soon.
          </p>
        </b-notification>
        <b-notification
          v-else-if="prereserveDOI"
          has-icon
          aria-close-label="Close notification"
        >
          <h1>Ready to publish</h1>
          <h2>Preserved DOI: {{ prereserveDOI.doi }}</h2>
          <h2>
            You can also review and add additional information on Zenodo:
            <a :href="prereserveUrl" target="_blank">{{ prereserveUrl }}</a>
          </h2>
          <p>
            Note: Please check carefully before publishing. It is generally not
            possible to remove items after they have been published. Changes
            will be added as a new version, but will not erase the previous
            version.
          </p>
        </b-notification>

        <b-notification
          v-if="uploadMode === 'update'"
          has-icon
          type="is-info"
          aria-close-label="Close notification"
        >
          <p>
            Note: After publishing the updated deposit, you may see the item
            disappear from the website, this is normal because Zenodo need some
            time to index the new version. It should combe back in a while.
          </p>
        </b-notification>

        <b-button
          v-if="client && client.credential && uploaded && !publishedUrl"
          @click="publishDeposition()"
          class="button is-primary is-fullwidth"
          expanded
        >
          <b-icon icon="upload"></b-icon>
          <span>Publish</span>
        </b-button>
      </b-step-item>
    </b-steps>

    <section style="padding: 10px;"></section>
  </div>
</template>

<script>
import { mapState } from "vuex";
import yaml from "js-yaml";

import {
  rdfToMetadata,
  resolveDOI,
  getFullRdfFromDeposit,
  fetchFile
} from "../utils";
import Markdown from "@/components/Markdown.vue";
import TagInputField from "@/components/TagInputField.vue";
import DropFilesField from "@/components/DropFilesField.vue";
import UploadForm from "./UploadForm.vue";
import doiRegex from "doi-regex";
import marked from "marked";
import DOMPurify from "dompurify";

export default {
  name: "upload",
  props: ["updateDepositId"],
  components: {
    markdown: Markdown,
    // eslint-disable-next-line vue/no-unused-components
    TagInputField,
    // eslint-disable-next-line vue/no-unused-components
    DropFilesField,
    UploadForm
  },
  mounted() {
    this.dropFiles = null;
    this.uploadStatus = "";
    this.uploadProgress = 0;
    if (!this.loadedUrl) {
      const repo = this.siteConfig.rdf_root_repo;
      let manifest_url = this.siteConfig.manifest_url;
      this.$store.dispatch("fetchResourceItems", {
        repo,
        manifest_url
      });
    }
    if (this.updateDepositId === "bookmarks") {
      this.imjoyReady.then(imjoy => {
        this.startFromBookmarks(imjoy).catch(e => {
          alert(`Failed to load from deposit URL: ${e}`);
        });
      });
    } else if (this.updateDepositId) {
      this.startFromDepositURL().catch(e => {
        alert(`Failed to load from deposit URL: ${e}`);
      });
    }
  },
  computed: {
    sameNameDeposits() {
      return (
        this.similarDeposits &&
        this.similarDeposits.filter(item => item.name === this.rdf.name)
      );
    },
    formatedModelYaml() {
      return this.rdfYaml && "```yaml\n" + this.rdfYaml + "\n```\n";
    },
    prereserveUrl() {
      if (this.prereserveDOI) {
        return `${this.$store.state.zenodoBaseURL}/deposit/${this.prereserveDOI.recid}`;
      } else {
        return null;
      }
    },
    userId() {
      return this.client && this.client.getUserId();
    },
    ...mapState({
      imjoyReady: state => state.imjoyReady,
      imjoy: state => state.imjoy,
      siteConfig: state => state.siteConfig,
      loadedUrl: state => state.loadedUrl,
      resourceItems: state => state.resourceItems,
      client: state => state.zenodoClient,
      zenodoBaseURL: state => state.zenodoBaseURL,
      bookmarks: state => state.bookmarks
    })
  },
  data() {
    return {
      // rdfFiles: null,
      dropFiles: null,
      uploadProgress: 0,
      uploadStatus: "",
      uploaded: false,
      jsonFields: null,
      rdfYaml: null,
      rdf: null,
      stepIndex: 0,
      publishedUrl: null,
      publishedDOI: null,
      requestedJoinCommunity: true,
      rdfType: "model",
      editedFiles: null,
      prereserveDOI: null,
      URI4Load: null,
      similarDeposits: null,
      depositId: null,
      uploadMode: null
    };
  },
  methods: {
    async gettingStarted() {
      await this.imjoy.api.showDialog({
        src:
          "https://slides.imjoy.io/?slides=https://raw.githubusercontent.com/imodpasteur/ShareLoc.XYZ/main/docs/how-to-upload-slides.md",
        passive: true,
        name: "Getting Started",
        fullscreen: true
      });
    },
    async startFromBookmarks(imjoy) {
      this.rdf = (await this.generateYamlFile(this.bookmarks[0])).rdf;
      let samples = [];
      for (let item of this.bookmarks) {
        if (item?.attachments?.samples) {
          samples = samples.concat(item.attachments.samples);
        }
        // add all the tags
        item.tags.forEach(tag => {
          if (!this.rdf.tags.includes(tag)) this.rdf.tags.push(tag);
        });
        if (item.download_url) {
          const filename = item.download_url
            .split("?")[0]
            .split("/")
            .pop();
          samples.push({
            name: item.name,
            files: [
              {
                type: "generator",
                async generate() {
                  const file = await fetchFile(
                    item.download_url,
                    filename,
                    msg => {
                      imjoy.api.showMessage(msg);
                    }
                  );
                  file.converted = [file];
                  file.sampleName = item.name;
                  return file;
                },
                sampleName: item.name,
                name: filename,
                url: item.download_url
              }
            ],
            views: item.covers.map(cover => {
              return {
                image: cover,
                image_name: cover
                  .split("?")[0]
                  .split("/")
                  .pop()
              };
            })
          });
        }
      }
      this.rdf.attachments = this.rdf.attachments || {};
      this.rdf.attachments.samples = samples;
      this.stepIndex = 1;
    },
    async startFromDepositURL() {
      const loadingComponent = this.$buefy.loading.open({
        container: this.$el
      });
      try {
        if (!this.client.credential) await this.login();
        this.URI4Load = `${this.zenodoBaseURL}/record/${this.updateDepositId}`;
        await this.loadRdfFromURL(this.URI4Load);
      } catch (e) {
        alert("Failed to load resource: " + this.updateDepositId);
      } finally {
        loadingComponent.close();
      }
    },
    startUpload() {
      this.rdf = {};
      // this.rdfFiles = [];
      this.stepIndex = 1;
    },
    submitRDF(rdf) {
      this.$nextTick(async () => {
        this.rdfYaml = (await this.generateYamlFile(rdf)).text;
        this.editedFiles = rdf.config._files;
        delete rdf.config._files;
        this.rdf = rdf;

        this.similarDeposits = await this.client.getResourceItems({
          community: this.siteConfig.zenodo_config.community,
          sort: "bestmatch",
          query: rdf.name
        });
        console.log("Similar deposits:", this.similarDeposits);
        // if there is any similar items, we can try to login first
        if (this.similarDeposits.length > 0)
          await this.client.getCredential(true);

        this.stepIndex = 2;
        this.$forceUpdate();
      });
    },
    async loadRdfFromURL(url) {
      try {
        const doiURLRegex = doiRegex.resolvePath();
        if (doiURLRegex.test(url)) {
          url = await resolveDOI(url.match(doiURLRegex)[4]);
        } else if (doiRegex().test(url)) {
          url = await resolveDOI(url);
        }
        const zenodoRegex = /zenodo.org\/(record|deposit)\/([0-9]+)/;
        const m = zenodoRegex.exec(url);
        if (m) {
          this.depositId = parseInt(m[2]);
          let depositionInfo;
          if (url.includes("/record/"))
            depositionInfo = await this.client.getDeposit(this.depositId);
          else {
            await this.client.getCredential(true);
            depositionInfo = await this.client.retrieve(this.depositId);
          }
          console.log("orcid matched: " + this.depositId, depositionInfo);
          this.rdf = await getFullRdfFromDeposit(depositionInfo);
          console.log("Full RDF:", this.rdf);
          // this.files = depositionInfo.files.map(item => {
          //   return {
          //     type: "remote",
          //     name: item.filename || item.key, // depending on what api we use, it may be in two different format
          //     size: item.filesize || item.size,
          //     url: item.links.self,
          //     checksum: item.checksum
          //   };
          // });
          // load files
          // this.rdfFiles = depositionInfo.files.map(item => {
          //   return {
          //     type: "remote",
          //     name: item.filename || item.key, // depending on what api we use, it may be in two different format
          //     size: item.filesize || item.size,
          //     url: item.links.self,
          //     checksum: item.checksum
          //   };
          // });
          if (this.rdf.documentation) {
            const baseUrl = depositionInfo.links.bucket + "/";
            const docsUrl = this.rdf.documentation.startsWith("http")
              ? this.rdf.documentation
              : new URL(this.rdf.documentation, baseUrl).href;
            const response = await fetch(docsUrl);
            this.rdf.config._docstring = await response.text();
          }
          this.stepIndex = 1;
        } else {
          alert(`Failed to parse RDF URL: ${url}`);
        }
      } catch (e) {
        console.error(e);
        alert(`Failed to fetch RDF from ${url}, error: ${e}`);
      }
    },

    async publishDeposition() {
      const loadingComponent = this.$buefy.loading.open({
        container: this.$el
      });
      try {
        const result = await this.client.publish(this.depositId);
        console.log("Published", result);
        this.publishedDOI = result.doi;
        this.publishedUrl = `${this.$store.state.zenodoBaseURL}/record/${this.depositId}`;
      } catch (e) {
        console.error(e);
        alert(`Failed to publish: ${e}`);
      } finally {
        loadingComponent.close();
      }
    },
    async login() {
      try {
        await this.client.getCredential(true);
        this.$forceUpdate();
      } catch (e) {
        alert(`Failed to login: ${e}`);
      }
    },
    async generateYamlFile(rdf) {
      const rdfCopy = Object.assign({}, rdf);
      // TODO: is there any field in the config we want to preserve?
      delete rdfCopy.config;
      let rdfYaml = yaml.dump(rdfCopy);
      rdfYaml = yaml.load(rdfYaml);
      if (rdfYaml?.attachments?.samples)
        rdfYaml.attachments.samples.forEach(sample => {
          sample.views.forEach(screenshot => {
            delete screenshot.image;
          });
        });
      rdfYaml = yaml.dump(rdfYaml);
      const blob = new Blob([rdfYaml], {
        type: "application/yaml"
      });
      const file = new File([blob], "rdf.yaml", { type: "application/yaml" });
      file.text = rdfYaml;
      file.rdf = rdfCopy;
      return file;
    },
    async createOrUpdateDeposit(depositId, skipUpload) {
      try {
        await this.client.getCredential(true);
        this.$forceUpdate();
      } catch (e) {
        alert(`Failed to login: ${e}`);
        return;
      }
      const loadingComponent = this.$buefy.loading.open({
        container: this.$el
      });
      this.similarDeposits = null;
      try {
        this.uploadProgress = 1;
        let depositionInfo;
        this.uploadMode = "new";
        if (depositId) {
          try {
            depositionInfo = await this.client.retrieve(depositId);

            // enter edit mode if submitted
            // if deposit.state == 'inprogress', the metadata can be updated
            // if deposit.submitted == false files can be updated as well.
            if (!skipUpload) {
              // The response body of this action is NOT the new version deposit, but the original resource.
              // The new version deposition can be accessed through the "latest_draft" under "links" in the response body.
              const deposit = await this.client.createNewVersion(
                depositionInfo
              );
              // e.g. https://sandbox.zenodo.org/api/deposit/depositions/868929
              const tmp = deposit.links.latest_draft.split("/");
              depositId = parseInt(tmp[tmp.length - 1]);
              depositionInfo = await this.client.retrieve(depositId);
            }
            if (
              depositionInfo.state !== "inprogress" &&
              depositionInfo.state !== "unsubmitted"
            )
              await this.client.edit(depositId);
            this.uploadMode = "new";
          } catch (e) {
            console.error(e);
            if (
              !confirm(
                `Failed to retrieve existing deposit (id: ${depositId}), would you like to create a new deposit instead?`
              )
            ) {
              return;
            }
          }
        } else depositionInfo = await this.client.createDeposition();

        this.depositId = depositionInfo.id;
        const baseUrl = `${this.client.baseURL}/record/${this.depositId}/files/`; //"file:///"; //depositionInfo.links.bucket + "/";

        let docstring = null;
        if (
          this.rdf.documentation &&
          !this.rdf.documentation.startsWith("http") &&
          this.rdf.documentation.endsWith(".md")
        ) {
          const file = this.editedFiles.filter(
            fn => fn === this.rdf.documentation.replace("./", "")
          )[0];
          if (file) {
            docstring = await file.async("string"); // get markdown
            docstring = DOMPurify.sanitize(marked(docstring));
          }
        }
        this.rdf.config = this.rdf.config || {};
        this.rdf.config._deposit = depositionInfo;

        const metadata = rdfToMetadata(this.rdf, baseUrl, docstring);
        // this will send a email request to the admin of bioimgae-io team
        if (this.requestedJoinCommunity) {
          metadata.communities.push({
            identifier: this.siteConfig.zenodo_config.community
          });
        }
        metadata.prereserve_doi = true; // we will generate the doi and store it in the model yaml file
        depositionInfo = await this.client.updateMetadata(
          depositionInfo,
          metadata
        );

        // transform the RDF here
        this.prereserveDOI = depositionInfo.metadata.prereserve_doi;
        this.rdf.id = depositionInfo.conceptdoi; //doi and recid
        this.rdf.config._doi = depositionInfo.metadata.prereserve_doi.doi;
        this.rdf.config._conceptdoi = depositionInfo.conceptdoi;
        if (skipUpload) {
          this.stepIndex = 3;
          return depositionInfo;
        }
        console.log("Edited files", this.editedFiles);
        const uploadFiles = this.editedFiles.filter(
          file => file.type !== "remote"
        );
        // sort the files so we will upload the covers in the end
        // this allows zenodo to display it as preview
        if (this.rdf.covers && this.rdf.covers.length > 0) {
          const covers = this.rdf.covers;
          uploadFiles.sort((a, b) => {
            if (
              covers.includes("./" + a.name) &&
              !covers.includes("./" + b.name)
            )
              return 1;
            else if (
              !covers.includes("./" + a.name) &&
              covers.includes("./" + b.name)
            )
              return -1;
            else return 0;
          });
        }
        for (let i = 0; i < uploadFiles.length; i++) {
          let file = uploadFiles[i];
          if (file.type === "generator") {
            // assuming we already have the generated name fixed
            file = await file.generate();
            if (file.sampleName) {
              // fix the converted file name and size in the attachments
              const sample = this.rdf.attachments.samples.filter(
                sample => sample.name === file.sampleName
              )[0];
              const generatedNames = file.converted.map(f => f.name);
              // remove the original files, e.g. the csv files
              sample.files = sample.files.filter(
                f => !generatedNames.includes(f.name)
              );
              // replace with converted files
              sample.files.push({
                name: file.name,
                size: file.size,
                checksum: file.checksum
              });
            }
          }

          const newName = file.sampleName
            ? file.sampleName + "/" + file.name
            : file.name;
          await this.client.uploadFile(depositionInfo, file, newName, size => {
            this.uploadProgress = Math.round((size / file.size) * 100);
            this.uploadStatus = `Uploading ${i + 1}/${uploadFiles.length}(${
              this.uploadProgress
            }%): ${file.name.slice(0, 40)}... `;
            this.$forceUpdate();
          });
        }

        // upload RDF file
        const rdfFile = await this.generateYamlFile(this.rdf);
        await this.client.uploadFile(depositionInfo, rdfFile);
        this.rdfYaml = rdfFile.text;

        this.uploadProgress = 0;
        this.uploadStatus = `Successfully uploaded ${uploadFiles.length} files.`;
        this.uploaded = true;
        this.stepIndex = 3;
      } catch (e) {
        console.error(e);
        alert(`Oops, failed to upload file: ${e}`);
      } finally {
        this.uploadProgress = 0;
        loadingComponent.close();
      }
    }
  }
};
</script>
<style scoped>
.upload {
  padding: 10px;
  width: 100%;
  overflow: auto;
  height: calc(100% - 48px);
  display: block;
}
.width-limited {
  max-width: 1080px;
  margin-left: auto !important;
  margin-right: auto !important;
  float: none !important;
}
</style>
