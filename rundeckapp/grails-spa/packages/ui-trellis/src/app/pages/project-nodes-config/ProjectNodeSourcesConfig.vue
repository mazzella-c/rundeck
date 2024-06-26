<template>
  <project-plugin-config
    config-prefix="resources.source"
    service-name="ResourceModelSource"
    :help="help"
    :add-button-text="addButtonText"
    @saved="pluginsConfigWasSaved"
    @modified="pluginsConfigWasModified"
    @reset="pluginsConfigWasReset"
    :edit-button-text="$t('Edit Node Sources')"
    :edit-mode="editMode"
    :mode-toggle="modeToggle"
    :event-bus="eventBus"
  >
    <template #item-extra="{ plugin, mode }">
      <div>
        <div
          v-if="
            isWriteable(plugin.origIndex) &&
            (showWriteableLinkMode === 'any' || showWriteableLinkMode === mode)
          "
        >
          <a
            :href="editPermalink(plugin.origIndex)"
            class="btn btn-sm btn-default"
          >
            <i class="glyphicon glyphicon-pencil"></i>
            {{ $t("Edit Nodes") }}
          </a>
        </div>
        <div v-if="sourceErrors(plugin.origIndex)" class="row row-space">
          <div class="col-sm-12">
            <div class="well well-sm">
              <div class="text-info">
                {{ $t("The Node Source had an error") }}:
              </div>
              <span class="text-danger">{{
                sourceErrors(plugin.origIndex)
              }}</span>
            </div>
          </div>
        </div>
      </div>
    </template>
  </project-plugin-config>
</template>
<script lang="ts">
import { defineComponent, PropType } from "vue";
import { EventBus, getRundeckContext, RundeckContext } from "../../../library";

import ProjectPluginConfig from "./ProjectPluginConfig.vue";
import { getProjectNodeSources, NodeSource } from "./nodeSourcesUtil";

export default defineComponent({
  components: {
    ProjectPluginConfig,
  },
  props: {
    help: {
      type: String,
      required: false,
    },
    addButtonText: {
      type: String,
      required: false,
    },
    editMode: {
      type: Boolean,
      default: false,
    },
    modeToggle: {
      type: Boolean,
      default: true,
    },
    showWriteableLinkMode: {
      type: String,
      default: "show",
    },
    eventBus: {
      type: Object as PropType<typeof EventBus>,
      required: false,
    },
  },
  emits: ["saved", "reset", "modified"],

  data() {
    return {
      project: "",
      rundeckContext: {} as RundeckContext,
      sourcesData: [] as NodeSource[],
    };
  },
  watch: {
    sourcesData: function (val, oldVal) {
      this.checkUnauthorized();
    },
  },
  mounted() {
    this.rundeckContext = getRundeckContext();
    if (
      window._rundeck &&
      window._rundeck.rdBase &&
      window._rundeck.projectName
    ) {
      this.loadNodeSourcesData();
    }
  },
  methods: {
    isWriteable(index: number): boolean {
      return (
        this.sourcesData.length > index &&
        this.sourcesData[index].resources.writeable
      );
    },
    editPermalink(index: number): string | undefined {
      return this.sourcesData.length > index &&
        this.sourcesData[index].resources.editPermalink
        ? this.sourcesData[index].resources.editPermalink
        : "#";
    },
    sourceErrors(index: number): string | undefined {
      return this.sourcesData.length > index
        ? this.sourcesData[index].errors
        : undefined;
    },
    pluginsConfigWasSaved() {
      this.$emit("saved");
      this.$emit("reset");
      this.loadNodeSourcesData().then();
    },
    pluginsConfigWasModified() {
      this.$emit("modified");
    },
    pluginsConfigWasReset() {
      this.$emit("reset");
    },
    async loadNodeSourcesData() {
      try {
        this.sourcesData = await getProjectNodeSources();
        this.checkUnauthorized();
      } catch (e) {
        return console.warn("Error getting node sources list", e);
      }
    },
    checkUnauthorized() {
      const globalErrors = [];
      this.sourcesData.forEach((source: NodeSource) => {
        if (
          source.errors !== undefined &&
          (source.errors.indexOf("Unauthorized access") > 0 ||
            source.errors.indexOf("storage") > 0)
        ) {
          globalErrors.push(source.errors);
          this.eventBus.emit("nodes-unauthorized", this.sourcesData.length);
        } else {
          if (globalErrors.length == 0) {
            this.eventBus.emit("nodes-unauthorized", 0);
          }
        }
      });
    },
  },
});
</script>
