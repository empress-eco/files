<template>
  <nav
    ondragstart="return false;"
    ondrop="return false;"
    class="bg-white top-0 border-b w-full">
    <div
      class="mx-auto pl-4 py-2.5 pr-2 h-12 z-10 flex items-center justify-between">
      <Breadcrumbs />
      <div v-if="isLoggedIn">
        <Dropdown
          :options="
            $store.state.hasWriteAccess ? fileOptions : readOnlyfileOptions
          "
          placement="left"
          class="basis-5/12 lg:basis-auto">
          <Button v-if="$route.meta.documentPage && isLoggedIn" variant="ghost">
            <FeatherIcon class="h-4" name="more-horizontal" />
          </Button>
        </Dropdown>
        <div
          v-if="
            $route.meta.documentPage && connectedUsers.length > 1 && isLoggedIn
          "
          class="bg-gray-200 rounded flex justify-center items-center px-1 mx-2">
          <UsersBar />
        </div>
        <Button
          v-if="$route.name === 'Recents'"
          class="line-clamp-1 truncate w-full"
          :disabled="!currentViewEntites?.length"
          theme="red"
          :variant="'subtle'"
          @click="this.emitter.emit('showCTADelete')">
          <template #prefix>
            <FeatherIcon name="trash-2" class="w-4" />
          </template>
          Clear Recents
        </Button>
        <Button
          v-else-if="$route.name === 'Favourites'"
          class="line-clamp-1 truncate"
          :disabled="!currentViewEntites?.length"
          theme="red"
          :variant="'subtle'"
          @click="this.emitter.emit('showCTADelete')">
          <template #prefix>
            <FeatherIcon name="trash-2" class="w-4" />
          </template>
          Clear Favourites
        </Button>
        <Button
          class="line-clamp-1 truncate"
          v-else-if="$route.name === 'Trash'"
          @click="this.emitter.emit('showCTADelete')"
          :disabled="!currentViewEntites?.length"
          theme="red"
          :variant="'subtle'">
          <template #prefix>
            <FeatherIcon name="trash-2" class="w-4" />
          </template>
          Empty Trash
        </Button>
        <Button
          v-else-if="$route.name === 'Document'"
          :disabled="!$store.state.hasWriteAccess"
          @click="emitter.emit('saveDocument')"
          class="bg-gray-200 rounded flex justify-center items-center px-1 mx-2"
          variant="subtle">
          <template #prefix>
            <FeatherIcon name="save" class="w-4" />
          </template>
          Save
        </Button>
        <Dropdown
          v-else
          :options="newEntityOptions"
          placement="left"
          class="basis-5/12 lg:basis-auto">
          <Button variant="solid">
            <template #prefix>
              <FeatherIcon name="upload" class="w-4" />
            </template>
            New
            <template #suffix>
              <FeatherIcon name="chevron-down" class="w-4" />
            </template>
          </Button>
        </Dropdown>
        <div v-if="$store.state.hasWriteAccess"></div>
        <!--

          Pushed implementation
          
        <Button
          class="stroke-1.5 ml-4 md:ml-5"
          appearance="minimal"
          icon="bell"></Button>
          
        -->
      </div>
      <div v-else class="ml-auto">
        <Button @click="$router.push({ name: 'Login' })" variant="solid">
          Sign In
        </Button>
      </div>
    </div>
  </nav>
  <NewFolderDialog
    v-model="showNewFolderDialog"
    :parent="$route.params.entityName"
    @success="
      () => {
        emitter.emit('fetchFolderContents');
        showNewFolderDialog = false;
      }
    " />
</template>
<script>
import UsersBar from "./UsersBar.vue";
import { Dropdown, FeatherIcon, Button } from "frappe-ui";
import NewFolderDialog from "@/components/NewFolderDialog.vue";
import Breadcrumbs from "@/components/Breadcrumbs.vue";
import { formatDate } from "@/utils/format";
import { FileDown } from "lucide-vue-next";
import { FileUp, FolderUp, FolderPlus, FileText } from "lucide-vue-next";
import { h } from "vue";

export default {
  name: "Navbar",
  components: {
    NewFolderDialog,
    Dropdown,
    FeatherIcon,
    Button,
    Breadcrumbs,
    UsersBar,
    FileUp,
    FileDown,
    FolderUp,
    FolderPlus,
    FileText,
  },
  props: {
    breadcrumbs: {
      type: Array,
      default: null,
    },
    mobileSidebarIsOpen: {
      type: Boolean,
      required: true,
    },
  },
  emits: ["toggleMobileSidebar", "fetchRecents"],
  data() {
    return {
      previewEntity: null,
      showPreview: false,
      showNewFolderDialog: false,
      newEntityOptions: [
        {
          group: "Upload",
          items: [
            {
              label: "Upload file",
              icon: h(FileUp, {
                id: "foo",
                class: "text-gray-900 stroke-[1.5]",
              }),
              onClick: () => this.emitter.emit("uploadFile"),
            },
            {
              label: "Upload Folder",
              icon: h(FolderUp, {
                id: "foo",
                class: "text-gray-900 stroke-[1.5]",
              }),
              onClick: () => this.emitter.emit("uploadFolder"),
              isEnabled: () => this.selectedEntities.length === 0,
            },
          ],
        },
        {
          group: "New",
          items: [
            {
              label: "New folder",
              icon: h(FolderPlus, {
                id: "foo",
                class: "text-gray-900 stroke-[1.5]",
              }),
              onClick: () => (this.showNewFolderDialog = true),
            },
            {
              label: "New Document",
              icon: h(FileText, {
                id: "foo",
                class: "text-gray-900 stroke-[1.5]",
              }),
              onClick: async () => {
                await this.$resources.createDocument.submit({
                  title: "Untitled Document",
                  content: null,
                  parent: this.$store.state.currentFolderID,
                });
                this.$router.push({
                  name: "Document",
                  params: { entityName: this.previewEntity.name },
                });
              },

              isEnabled: () => this.selectedEntities.length === 0,
            },
          ],
        },
      ],
      readOnlyfileOptions: [
        {
          group: "Export",
          items: [
            {
              icon: FileDown,
              label: "Export to PDF",
              onClick: () => this.emitter.emit("exportDocToPDF"),
            },
          ],
        },
      ],
      fileOptions: [
        {
          group: "Export",
          items: [
            {
              icon: FileDown,
              label: "Export to PDF",
              onClick: () => this.emitter.emit("exportDocToPDF"),
            },
          ],
        },
        {
          group: "Import",
          items: [
            {
              icon: FileDown,
              label: "Import from DOCX",
              onClick: () => this.emitter.emit("importDocFromWord"),
            },
          ],
        },
      ],
    };
  },
  computed: {
    fullName() {
      return this.$store.state.user.fullName;
    },
    imageURL() {
      return this.$store.state.user.imageURL;
    },
    isLoggedIn() {
      return this.$store.getters.isLoggedIn;
    },
    connectedUsers() {
      return this.$store.state.connectedUsers;
    },
    currentViewEntites() {
      return this.$store.state.currentViewEntites;
    },
  },
  /*   methods: {
    openEntity(entity) {
      if (entity.is_group) {
        this.$router.push({
          name: "Folder",
          params: { entityName: entity.name },
        });
      } else if (entity.document) {
        this.$router.push({
          name: "Document",
          params: { entityName: entity.name },
        });
      } else {
        this.previewEntity = entity;
        this.showPreview = true;
      }
    },
    hidePreview() {
      this.showPreview = false;
      this.previewEntity = null;
    },
  }, */
  resources: {
    createDocument() {
      return {
        url: "drive.api.files.create_document_entity",
        onSuccess(data) {
          data.modified = formatDate(data.modified);
          data.creation = formatDate(data.creation);
          this.$store.commit("setEntityInfo", [data]);
          this.previewEntity = data;
          data.owner = "Me";
        },
        onError(data) {
          console.log(data);
        },
        auto: false,
      };
    },
  },
};
</script>
