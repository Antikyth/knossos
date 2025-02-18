<template>
  <div v-if="version">
    <div v-if="showKnownErrors" class="known-errors card">
      <ul>
        <li v-if="version.version_number === ''">
          Your version must have a version number.
        </li>
        <li v-if="version.game_versions.length === 0">
          Your version must have the supported Minecraft versions selected.
        </li>
        <li v-if="newFiles.length === 0 && version.files.length === 0">
          Your version must have a file uploaded.
        </li>
        <li
          v-if="
            version.loaders.length === 0 &&
            project.project_type !== 'resourcepack'
          "
        >
          Your version must have the supported mod loaders selected.
        </li>
      </ul>
    </div>
    <div class="content card">
      <ModalConfirm
        ref="modal_confirm"
        title="Are you sure you want to delete this version?"
        description="This will remove this version forever (like really forever)."
        :has-to-type="false"
        proceed-label="Delete"
        @proceed="deleteVersion()"
      />
      <ModalReport
        ref="modal_version_report"
        :item-id="version.id"
        item-type="version"
      />
      <div class="columns">
        <nuxt-link
          v-if="mode === 'version'"
          class="iconified-button back-button"
          :to="`${
            $nuxt.context.from &&
            ($nuxt.context.from.name === 'type-id-changelog' ||
              $nuxt.context.from.name === 'type-id-versions')
              ? $nuxt.context.from.fullPath
              : `/${project.project_type}/${
                  project.slug ? project.slug : project.id
                }/versions`
          }`"
        >
          <BackIcon aria-hidden="true" />
          Back to list
        </nuxt-link>
      </div>
      <div>
        <div v-if="mode === 'version'" class="version-header">
          <h2>{{ version.name }}</h2>

          <div v-if="version.featured" class="featured">
            <StarIcon aria-hidden="true" />
            Featured
          </div>
          <div
            v-else-if="featuredVersions.find((x) => x.id === version.id)"
            class="featured"
          >
            <StarIcon aria-hidden="true" />
            Auto-featured
          </div>
        </div>
        <div
          v-if="mode === 'edit' || mode === 'create'"
          class="version-data-inputs"
        >
          <input
            v-model="version.name"
            class="full-width-input"
            type="text"
            placeholder="Enter an optional version name..."
            maxlength="64"
          />
          <Checkbox v-model="version.featured" label="Featured" />
          <hr class="card-divider" />
        </div>
        <div v-if="mode === 'edit'" class="header-buttons button-group columns">
          <h3 class="column-grow-1">Edit version</h3>
          <nuxt-link
            v-if="$auth.user"
            :to="`/${project.project_type}/${
              project.slug ? project.slug : project.id
            }/version/${encodeURI(version.displayUrlEnding)}`"
            class="iconified-button"
          >
            <CrossIcon aria-hidden="true" />
            Cancel
          </nuxt-link>
          <button
            class="iconified-button brand-button"
            @click="saveEditedVersion"
          >
            <SaveIcon aria-hidden="true" />
            Save
          </button>
        </div>
        <div
          v-else-if="mode === 'create'"
          class="header-buttons button-group columns"
        >
          <h3 class="column-grow-1">Create version</h3>
          <nuxt-link
            v-if="$auth.user"
            :to="`/${project.project_type}/${
              project.slug ? project.slug : project.id
            }/versions`"
            class="iconified-button"
          >
            <CrossIcon aria-hidden="true" />
            Cancel
          </nuxt-link>
          <button class="iconified-button brand-button" @click="createVersion">
            <CheckIcon aria-hidden="true" />
            Create
          </button>
        </div>
        <div v-else class="button-group">
          <a
            v-if="primaryFile"
            v-tooltip="
              primaryFile.filename + ' (' + $formatBytes(primaryFile.size) + ')'
            "
            :href="primaryFile.url"
            class="bold-button iconified-button brand-button"
            :title="`Download ${primaryFile.filename}`"
          >
            <DownloadIcon aria-hidden="true" />
            Download
          </a>
          <button
            v-if="$auth.user"
            class="action iconified-button"
            @click="$refs.modal_version_report.show()"
          >
            <ReportIcon aria-hidden="true" />
            Report
          </button>
          <a v-else class="action iconified-button" :href="authUrl">
            <ReportIcon aria-hidden="true" />
            Report
          </a>
          <nuxt-link
            v-if="currentMember"
            class="action iconified-button"
            :to="`/${project.project_type}/${
              project.slug ? project.slug : project.id
            }/version/${encodeURI(version.displayUrlEnding)}/edit`"
            @click.prevent="mode = 'edit'"
          >
            <EditIcon aria-hidden="true" />
            Edit
          </nuxt-link>
          <button
            v-if="currentMember"
            class="action iconified-button danger-button"
            @click="$refs.modal_confirm.show()"
          >
            <TrashIcon aria-hidden="true" />
            Delete
          </button>
        </div>
        <section v-if="mode === 'edit' || mode === 'create'">
          <h3>Changelog</h3>
          <Chips
            v-model="changelogViewMode"
            class="separator"
            :items="['source', 'preview']"
          />
          <div
            v-if="changelogViewMode === 'source'"
            class="resizable-textarea-wrapper"
          >
            <textarea id="body" v-model="version.changelog" spellcheck="true" />
          </div>
          <div
            v-if="changelogViewMode === 'preview'"
            v-highlightjs
            class="markdown-body"
            v-html="
              version.changelog
                ? $xss($md.render(version.changelog))
                : 'No changelog specified.'
            "
          ></div>
        </section>
        <section v-else>
          <h3>Changelog</h3>
          <div
            v-highlightjs
            class="markdown-body"
            v-html="
              version.changelog
                ? $xss($md.render(version.changelog))
                : 'No changelog specified.'
            "
          ></div>
          <hr class="card-divider" />
        </section>
        <section>
          <h3>Metadata</h3>
          <div :class="'data-wrapper ' + mode">
            <div class="data">
              <p class="title">Release channel</p>
              <Multiselect
                v-if="mode === 'edit' || mode === 'create'"
                v-model="version.version_type"
                class="input"
                placeholder="Select one"
                :options="['release', 'beta', 'alpha']"
                :custom-label="
                  (value) => value.charAt(0).toUpperCase() + value.slice(1)
                "
                :searchable="false"
                :close-on-select="true"
                :show-labels="false"
                :allow-empty="false"
              />
              <VersionBadge
                v-else-if="version.version_type === 'release'"
                class="value"
                type="release"
                color="green"
              />
              <VersionBadge
                v-else-if="version.version_type === 'beta'"
                class="value"
                type="beta"
                color="orange"
              />
              <VersionBadge
                v-else-if="version.version_type === 'alpha'"
                class="value"
                type="alpha"
                color="red"
              />
            </div>
            <div v-if="project.project_type !== 'resourcepack'" class="data">
              <p class="title">
                Loaders<span
                  v-if="mode === 'edit' || mode === 'create'"
                  class="required"
                  >*</span
                >
              </p>
              <multiselect
                v-if="mode === 'edit' || mode === 'create'"
                v-model="version.loaders"
                :options="
                  $tag.loaders
                    .filter((x) =>
                      x.supported_project_types.includes(
                        project.actualProjectType.toLowerCase()
                      )
                    )
                    .map((it) => it.name)
                "
                :custom-label="(value) => $formatCategory(value)"
                :loading="$tag.loaders.length === 0"
                :multiple="true"
                :searchable="false"
                :show-no-results="false"
                :close-on-select="false"
                :clear-on-select="false"
                :show-labels="false"
                :limit="6"
                :hide-selected="true"
                placeholder="Choose loaders..."
              />
              <p v-else class="value">
                {{ version.loaders.map((x) => $formatCategory(x)).join(', ') }}
              </p>
            </div>
            <div v-if="mode === 'version'" class="data">
              <p class="title">Downloads</p>
              <p class="value">{{ $formatNumber(version.downloads) }}</p>
            </div>
            <div class="data">
              <p class="title">
                Version number<span
                  v-if="mode === 'edit' || mode === 'create'"
                  class="required"
                  >*</span
                >
              </p>
              <input
                v-if="mode === 'edit' || mode === 'create'"
                v-model="version.version_number"
                type="text"
                placeholder="Enter the version number..."
                maxlength="32"
              />
              <p v-else class="value">{{ version.version_number }}</p>
            </div>
            <div class="data">
              <p class="title">
                Minecraft versions<span
                  v-if="mode === 'edit' || mode === 'create'"
                  class="required"
                  >*</span
                >
              </p>
              <div v-if="mode === 'edit' || mode === 'create'">
                <multiselect
                  v-model="version.game_versions"
                  :options="
                    showSnapshots
                      ? $tag.gameVersions.map((x) => x.version)
                      : $tag.gameVersions
                          .filter((it) => it.version_type === 'release')
                          .map((x) => x.version)
                  "
                  :loading="$tag.gameVersions.length === 0"
                  :multiple="true"
                  :searchable="true"
                  :show-no-results="false"
                  :close-on-select="false"
                  :clear-on-select="false"
                  :show-labels="false"
                  :limit="6"
                  :hide-selected="true"
                  placeholder="Choose versions..."
                />
                <Checkbox
                  v-model="showSnapshots"
                  label="Include snapshots"
                  description="Include snapshots"
                  style="margin-top: 0.5rem"
                  :border="false"
                />
              </div>
              <p v-else class="value">
                {{ $formatVersion(version.game_versions) }}
              </p>
            </div>
            <div v-if="mode === 'version'" class="data">
              <p class="title">Published</p>
              <p class="value">
                {{ $dayjs(version.date_published).format('MMM D, YYYY') }}
                <span
                  v-if="members.find((x) => x.user.id === version.author_id)"
                >
                  by
                  <nuxt-link
                    class="text-link"
                    :to="
                      '/user/' +
                      members.find((x) => x.user.id === version.author_id).user
                        .username
                    "
                    >{{
                      members.find((x) => x.user.id === version.author_id).user
                        .username
                    }}</nuxt-link
                  >
                </span>
              </p>
            </div>
            <div v-if="mode === 'version'" class="data">
              <p class="title">Version ID</p>
              <p class="value"><CopyCode :text="version.id" /></p>
            </div>
          </div>
          <hr class="card-divider" />
        </section>
        <section
          v-if="
            (mode === 'version' &&
              version.dependencies.filter((x) => !x.file_name).length > 0) ||
            ((mode === 'edit' || mode === 'create') &&
              project.project_type.toLowerCase() !== 'modpack')
          "
        >
          <h3>Dependencies</h3>
          <div class="dependencies">
            <div
              v-for="(dependency, index) in version.dependencies.filter(
                (x) => !x.file_name
              )"
              :key="index"
              class="dependency"
            >
              <Avatar
                :src="dependency.project ? dependency.project.icon_url : null"
                alt="dependency-icon"
                size="sm"
              />
              <div class="info">
                <nuxt-link
                  :to="
                    dependency.project
                      ? dependency.version
                        ? `/${dependency.project.project_type}/${
                            dependency.project.slug
                              ? dependency.project.slug
                              : dependency.project.id
                          }/version/${encodeURI(
                            dependency.version.version_number
                          )}`
                        : `/${dependency.project.project_type}/${
                            dependency.project.slug
                              ? dependency.project.slug
                              : dependency.project.id
                          }/`
                      : ''
                  "
                >
                  <h4 class="title">
                    {{
                      dependency.project
                        ? dependency.project.title
                        : 'Unknown Project'
                    }}
                  </h4>
                  <p v-if="dependency.version" class="version-number">
                    Version {{ dependency.version.version_number }} is
                    {{ dependency.dependency_type }}
                  </p>
                  <p v-else>
                    {{ dependency.dependency_type }}
                  </p>
                </nuxt-link>
                <div class="bottom">
                  <button
                    v-if="mode === 'edit' || mode === 'create'"
                    class="iconified-button"
                    @click="version.dependencies.splice(index, 1)"
                  >
                    <TrashIcon /> Remove
                  </button>
                </div>
              </div>
            </div>
          </div>
          <div
            v-if="mode === 'edit' || mode === 'create'"
            class="edit-dependency"
          >
            <h4>Add dependency</h4>
            <Chips
              v-model="dependencyAddMode"
              class="separator"
              :items="['project', 'version']"
            />
            <div class="edit-info">
              <input
                v-model="newDependencyId"
                type="text"
                oninput="this.value = this.value.replace(' ', '')"
                :placeholder="`Enter the ${dependencyAddMode} ID${
                  dependencyAddMode === 'project' ? '/slug' : ''
                }`"
                @keyup.enter="addDependency"
              />
              <Multiselect
                v-model="newDependencyType"
                class="input"
                placeholder="Select one"
                :options="['required', 'optional', 'incompatible', 'embedded']"
                :custom-label="
                  (value) => value.charAt(0).toUpperCase() + value.slice(1)
                "
                :searchable="false"
                :close-on-select="true"
                :show-labels="false"
                :allow-empty="false"
              />
              <button class="iconified-button" @click="addDependency">
                <PlusIcon />
                Add
              </button>
            </div>
          </div>
          <hr class="card-divider" />
        </section>
        <section
          v-if="
            version.dependencies.filter((x) => x.file_name).length > 0 &&
            mode === 'version'
          "
        >
          <div>
            <h3>External Dependencies</h3>
            <InfoIcon
              v-tooltip="
                'Mods not part of the Modrinth platform but depended on by this project'
              "
            />
          </div>
          <div class="external-dependency">
            <div
              v-for="(dependency, index) in version.dependencies.filter(
                (x) => x.file_name
              )"
              :key="index"
              class="external-dependency"
            >
              <p v-if="dependency.file_name">
                {{ decodeURIComponent(dependency.file_name) }}
              </p>
            </div>
          </div>
          <hr class="card-divider" />
        </section>
        <section
          v-if="
            version.files.length > 0 || mode === 'edit' || mode === 'create'
          "
        >
          <h3>
            Files<span
              v-if="mode === 'edit' || mode === 'create'"
              class="required"
              >*</span
            >
          </h3>
          <div
            v-for="(file, index) in version.files"
            :key="file.hashes.sha1"
            class="file"
          >
            <p class="filename">
              {{ file.filename }}
            </p>
            <div
              v-if="primaryFile.hashes.sha1 === file.hashes.sha1"
              class="featured"
            >
              <StarIcon aria-hidden="true" />
              Primary
            </div>
            <a
              :href="file.url"
              class="action iconified-button"
              :title="`Download ${file.filename}`"
              tabindex="0"
            >
              <DownloadIcon aria-hidden="true" />
              Download
            </a>
            <p v-if="mode === 'version'">({{ $formatBytes(file.size) }})</p>
            <button
              v-if="mode === 'edit'"
              class="action iconified-button"
              @click="
                deleteFiles.push(file.hashes.sha1)
                version.files.splice(index, 1)
              "
            >
              <TrashIcon aria-hidden="true" />
              Remove
            </button>
            <button
              v-if="
                mode === 'edit' && primaryFile.hashes.sha1 !== file.hashes.sha1
              "
              class="action iconified-button"
              @click="primaryFile = file"
            >
              <StarIcon aria-hidden="true" />
              Make primary
            </button>
          </div>
          <div v-if="mode === 'edit' || mode === 'create'">
            <div
              v-for="(file, index) in newFiles"
              :key="index + 'new'"
              class="file"
            >
              <p class="filename">{{ file.name }}</p>
              <button
                class="action iconified-button"
                @click="newFiles.splice(index, 1)"
              >
                <TrashIcon aria-hidden="true" />
                Remove
              </button>
            </div>
          </div>
          <FileInput
            v-if="mode === 'edit' || mode === 'create'"
            multiple
            should-always-reset
            class="choose-files"
            :accept="
              project.actualProjectType.toLowerCase() === 'modpack'
                ? '.mrpack,application/x-modrinth-modpack+zip'
                : project.project_type.toLowerCase() === 'mod'
                ? '.jar,actualProjectType/java-archive'
                : '*'
            "
            prompt="Choose files or drag them here"
            :max-size="524288000"
            @change="(x) => x.forEach((y) => newFiles.push(y))"
          />
          <span v-if="mode === 'edit' || mode === 'create'">
            You may upload multiple files, but this should only be used for
            cases like sources or Javadocs.
            <p
              v-if="project.project_type.toLowerCase() === 'modpack'"
              aria-label="Warning"
            >
              Modpack support is currently in alpha, and you may encounter
              issues. Our documentation includes instructions on
              <a
                href="https://docs.modrinth.com/docs/modpacks/creating_modpacks/"
                target="_blank"
                class="text-link"
                >creating modpacks</a
              >. Join us on
              <a
                href="https://discord.gg/EUHuJHt"
                target="_blank"
                class="text-link"
                >Discord</a
              >
              for support.
            </p>
          </span>
        </section>
      </div>
      <NuxtChild v-show="false" :mode.sync="mode" />
    </div>
  </div>
  <div v-else></div>
</template>
<script>
import Multiselect from 'vue-multiselect'
import FileInput from '~/components/ui/FileInput'

import InfoIcon from '~/assets/images/utils/info.svg?inline'
import TrashIcon from '~/assets/images/utils/trash.svg?inline'
import SaveIcon from '~/assets/images/utils/save.svg?inline'
import PlusIcon from '~/assets/images/utils/plus.svg?inline'
import CrossIcon from '~/assets/images/utils/x.svg?inline'
import EditIcon from '~/assets/images/utils/edit.svg?inline'
import DownloadIcon from '~/assets/images/utils/download.svg?inline'
import ReportIcon from '~/assets/images/utils/report.svg?inline'
import BackIcon from '~/assets/images/utils/left-arrow.svg?inline'
import StarIcon from '~/assets/images/utils/star.svg?inline'
import CheckIcon from '~/assets/images/utils/check.svg?inline'
import VersionBadge from '~/components/ui/Badge'
import Checkbox from '~/components/ui/Checkbox'
import Chips from '~/components/ui/Chips'
import ModalConfirm from '~/components/ui/ModalConfirm'
import ModalReport from '~/components/ui/ModalReport'
import CopyCode from '~/components/ui/CopyCode'
import Avatar from '~/components/ui/Avatar'

export default {
  components: {
    Avatar,
    CopyCode,
    ModalConfirm,
    ModalReport,
    FileInput,
    Chips,
    Checkbox,
    VersionBadge,
    DownloadIcon,
    TrashIcon,
    EditIcon,
    ReportIcon,
    BackIcon,
    StarIcon,
    CheckIcon,
    Multiselect,
    SaveIcon,
    PlusIcon,
    CrossIcon,
    InfoIcon,
  },
  props: {
    project: {
      type: Object,
      default() {
        return {}
      },
    },
    versions: {
      type: Array,
      default() {
        return []
      },
    },
    featuredVersions: {
      type: Array,
      default() {
        return []
      },
    },
    members: {
      type: Array,
      default() {
        return [{}]
      },
    },
    currentMember: {
      type: Object,
      default() {
        return null
      },
    },
    dependencies: {
      type: Object,
      default() {
        return {}
      },
    },
  },
  data() {
    return {
      mode: 'version',
      primaryFile: {},
      version: {},

      changelogViewMode: 'source',

      dependencyAddMode: 'project',
      newDependencyId: '',
      newDependencyType: 'required',

      newFiles: [],
      deleteFiles: [],
      showSnapshots: false,

      showKnownErrors: false,
    }
  },
  async fetch() {
    await this.setVersion()
  },
  head() {
    if (!this.version || !this.version.game_versions) {
      return {}
    }
    const title = `${
      this.mode === 'create' ? 'Create Version' : this.version.name
    } - ${this.project.title}`
    const description = `Download ${this.project.title} ${
      this.version.version_number
    } on Modrinth. Supports ${this.$formatVersion(
      this.version.game_versions
    )} ${this.version.loaders
      .map((x) => x.charAt(0).toUpperCase() + x.slice(1))
      .join(' & ')}. Published on ${this.$dayjs(
      this.version.date_published
    ).format('MMM D, YYYY')}. ${this.version.downloads} downloads.`

    return {
      title,
      meta: [
        {
          hid: 'og:title',
          name: 'og:title',
          content: title,
        },
        {
          hid: 'apple-mobile-web-app-title',
          name: 'apple-mobile-web-app-title',
          content: title,
        },
        {
          hid: 'og:description',
          name: 'og:description',
          content: description,
        },
        {
          hid: 'description',
          name: 'description',
          content: description,
        },
      ],
    }
  },
  computed: {
    authUrl() {
      return `${process.env.authURLBase}auth/init?url=${process.env.domain}${this.$route.path}`
    },
  },
  watch: {
    '$route.path': {
      async handler() {
        await this.setVersion()
      },
    },
  },
  methods: {
    checkFields() {
      return !(
        this.version.version_number === '' ||
        this.version.game_versions.length === 0 ||
        (this.version.loaders.length === 0 &&
          this.project.project_type !== 'resourcepack') ||
        (this.newFiles.length === 0 && this.version.files.length === 0)
      )
    },
    reset() {
      this.changelogViewMode = 'source'
      this.dependencyAddMode = 'project'
      this.newDependencyId = ''
      this.newDependencyType = 'required'

      this.primaryFile = {}
      this.version = {}
      this.newFiles = []
      this.deleteFiles = []
    },
    async setVersion() {
      this.reset()
      const path = this.$route.name.split('-')
      this.mode = path[path.length - 1]

      if (this.mode === 'create') {
        this.version = {
          id: 'none',
          project_id: this.project.id,
          author_id: this.currentMember.user.id,
          name: '',
          version_number: '',
          changelog: '',
          date_published: Date.now(),
          downloads: 0,
          version_type: 'release',
          files: [],
          dependencies: [],
          game_versions: [],
          loaders: [],
          featured: false,
        }

        return
      }

      this.version = this.versions.find(
        (x) => x.id === this.$route.params.version
      )

      if (!this.version)
        this.version = this.versions.find(
          (x) => x.displayUrlEnding === this.$route.params.version
        )

      if (!this.version) {
        this.$nuxt.context.error({
          statusCode: 404,
          message: 'The page could not be found',
        })
        return
      }

      this.version = JSON.parse(JSON.stringify(this.version))
      this.primaryFile =
        this.version.files.find((file) => file.primary) ?? this.version.files[0]

      if (!this.version.changelog && this.version.changelog_url) {
        this.version.changelog = (
          await this.$axios.get(this.version.changelog_url)
        ).data
      }

      for (const dependency of this.version.dependencies) {
        dependency.version = this.dependencies.versions.find(
          (x) => x.id === dependency.version_id
        )

        if (dependency.version) {
          dependency.project = this.dependencies.projects.find(
            (x) => x.id === dependency.version.project_id
          )
        }

        if (!dependency.project) {
          dependency.project = this.dependencies.projects.find(
            (x) => x.id === dependency.project_id
          )
        }
      }
    },
    async addDependency() {
      try {
        if (this.dependencyAddMode === 'project') {
          const project = (
            await this.$axios.get(`project/${this.newDependencyId}`)
          ).data

          this.version.dependencies.push({
            project,
            project_id: project.id,
            dependency_type: this.newDependencyType,
          })
          this.$emit('update:dependencies', {
            projects: this.dependencies.projects.concat([project]),
            versions: this.dependencies.versions,
          })
        } else if (this.dependencyAddMode === 'version') {
          const version = (
            await this.$axios.get(`version/${this.newDependencyId}`)
          ).data
          const project = (
            await this.$axios.get(`project/${version.project_id}`)
          ).data

          this.version.dependencies.push({
            version,
            project,
            version_id: version.id,
            project_id: project.id,
            dependency_type: this.newDependencyType,
          })
          this.$emit('update:dependencies', {
            projects: this.dependencies.projects.concat([project]),
            versions: this.dependencies.versions.concat([version]),
          })
        }
      } catch {
        this.$notify({
          group: 'main',
          title: 'Invalid Dependency',
          text: 'The specified dependency does not exist',
          type: 'error',
        })
      }

      this.newDependencyId = ''
    },
    async saveEditedVersion() {
      this.$nuxt.$loading.start()

      if (!this.checkFields()) {
        this.showKnownErrors = true
        this.$nuxt.$loading.finish()
        return
      }
      this.showKnownErrors = false

      try {
        if (this.newFiles.length > 0) {
          const formData = new FormData()
          formData.append('data', JSON.stringify({}))
          for (let i = 0; i < this.newFiles.length; i++) {
            formData.append(
              this.newFiles[i].name.concat('-' + i),
              new Blob([this.newFiles[i]]),
              this.newFiles[i].name
            )
          }

          await this.$axios({
            url: `version/${this.version.id}/file`,
            method: 'POST',
            data: formData,
            headers: {
              'Content-Type': 'multipart/form-data',
              Authorization: this.$auth.token,
            },
          })
        }

        for (const hash of this.deleteFiles) {
          await this.$axios.delete(
            `version_file/${hash}`,
            this.$defaultHeaders()
          )
        }

        this.version.primary_file = ['sha1', this.primaryFile.hashes.sha1]
        const copyVersion = JSON.parse(JSON.stringify(this.version))
        delete copyVersion.downloads
        copyVersion.name = copyVersion.name || copyVersion.version_number
        await this.$axios.patch(
          `version/${this.version.id}`,
          copyVersion,
          this.$defaultHeaders()
        )

        const [version, featuredVersions] = (
          await Promise.all([
            this.$axios.get(
              `version/${this.version.id}`,
              this.$defaultHeaders()
            ),
            this.$axios.get(
              `project/${this.version.project_id}/version?featured=true`,
              this.$defaultHeaders()
            ),
          ])
        ).map((it) => it.data)

        const editedVersions = this.versions
        const index = this.versions.findIndex((x) => x.id === this.version.id)
        editedVersions.splice(index, 1, version)

        const newEditedVersions = this.$computeVersions(editedVersions)

        this.$emit('update:versions', newEditedVersions)
        this.$emit('update:featuredVersions', featuredVersions)

        this.newFiles = []
        this.deleteFiles = []

        await this.$router.replace(
          `/${this.project.project_type}/${
            this.project.slug ? this.project.slug : this.project.id
          }/version/${encodeURI(newEditedVersions[index].displayUrlEnding)}`
        )
      } catch (err) {
        this.$notify({
          group: 'main',
          title: 'An error occurred',
          text: err.response.data.description,
          type: 'error',
        })
        window.scrollTo({ top: 0, behavior: 'smooth' })
      }
      this.$nuxt.$loading.finish()
    },
    async createVersion() {
      this.$nuxt.$loading.start()

      if (!this.checkFields()) {
        this.showKnownErrors = true
        this.$nuxt.$loading.finish()
        return
      }
      this.showKnownErrors = false

      const formData = new FormData()

      const fileParts = this.newFiles.map((f, idx) => `${f.name}-${idx}`)

      if (this.project.project_type === 'resourcepack') {
        this.version.loaders = ['minecraft']
      }

      const newVersion = {
        project_id: this.version.project_id,
        file_parts: fileParts,
        version_number: this.version.version_number,
        version_title: this.version.name || this.version.version_number,
        version_body: this.version.changelog,
        dependencies: this.version.dependencies,
        game_versions: this.version.game_versions,
        loaders: this.version.loaders,
        release_channel: this.version.version_type,
        featured: this.version.featured,
      }

      formData.append('data', JSON.stringify(newVersion))

      for (let i = 0; i < this.newFiles.length; i++) {
        formData.append(
          fileParts[i],
          new Blob([this.newFiles[i]]),
          this.newFiles[i].name
        )
      }

      try {
        const data = (
          await this.$axios({
            url: 'version',
            method: 'POST',
            data: formData,
            headers: {
              'Content-Type': 'multipart/form-data',
              Authorization: this.$auth.token,
            },
          })
        ).data

        const newProject = JSON.parse(JSON.stringify(this.project))
        newProject.versions = newProject.versions.concat([data.id])
        const newVersions = this.$computeVersions(this.versions.concat([data]))

        await this.$emit('update:project', newProject)
        await this.$emit('update:versions', newVersions)

        await this.$router.push(
          `/${this.project.project_type}/${
            this.project.slug ? this.project.slug : this.project.project_id
          }/version/${encodeURI(
            newVersions[newVersions.length - 1].displayUrlEnding
          )}`
        )
      } catch (err) {
        this.$notify({
          group: 'main',
          title: 'An error occurred',
          text: err.response.data.description,
          type: 'error',
        })
        window.scrollTo({ top: 0, behavior: 'smooth' })
      }
      this.$nuxt.$loading.finish()
    },
    async deleteVersion() {
      this.$nuxt.$loading.start()

      await this.$axios.delete(
        `version/${this.version.id}`,
        this.$defaultHeaders()
      )

      await this.$router.replace(
        `/${this.project.project_type}/${this.project.id}`
      )
      this.$nuxt.$loading.finish()
    },
  },
}
</script>

<style lang="scss" scoped>
.content {
  max-width: calc(100% - (2 * var(--spacing-card-lg)));
}

.required {
  color: var(--color-special-red);
}

.version-header {
  display: flex;
  align-items: center;
  margin: 1rem 0;

  h2 {
    margin: 0 0.75rem 0 0;
    font-size: var(--font-size-xl);
  }
}

section {
  margin: 1rem 0;

  .separator {
    margin: var(--spacing-card-sm) 0;
  }

  .choose-files {
    margin-bottom: var(--spacing-card-sm);
  }
}

.header-buttons {
  justify-content: right;
}

.buttons {
  display: flex;
  flex-wrap: wrap;
  row-gap: 0.5rem;

  @media screen and (min-width: 1024px) {
    margin-left: auto;
  }
}

.version-data-inputs {
  margin: 0.5rem 0;
}

.data-wrapper {
  display: flex;
  flex-wrap: wrap;
  flex-direction: column;

  @media screen and (min-width: 800px) {
    flex-direction: row;
  }

  .data {
    flex-basis: calc(33.333333% - 0.5rem);

    margin-right: 0.5rem;
    margin-bottom: 0.5rem;

    .title {
      font-weight: bold;
      margin-bottom: 0.25rem;
    }

    p {
      margin: 0;
    }
  }

  &.edit,
  &.create {
    .data {
      flex-basis: calc(50% - 0.5rem);

      input {
        margin: 0;
        width: 100%;
      }
    }
  }
}

.dependencies {
  display: flex;
  flex-wrap: wrap;
  column-gap: 2rem;
  row-gap: 1rem;

  .dependency {
    align-items: center;
    display: flex;
    gap: 0.25rem;

    @media screen and (min-width: 800px) {
      flex-basis: 30%;
    }

    .info {
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      padding: 0.25rem;

      p {
        margin: 0;
      }

      .title {
        margin: 0 0.25rem 0 0;
      }
    }
  }
}

.external-dependency {
  p {
    margin: 0.25rem 0;
  }
}

.edit-dependency {
  h4 {
    margin: 0 0 0.25rem 0;
  }

  .edit-info {
    display: flex;
    flex-wrap: wrap;
    row-gap: 0.25rem;

    .multiselect {
      max-width: 10rem;
    }

    .iconified-button {
      min-height: 2.5rem;
    }

    input,
    .multiselect,
    .iconified-button {
      margin: 0 0.5rem 0 0;
    }
  }
}

.file {
  display: flex;
  align-items: center;
  margin-bottom: 0.25rem;
  flex-wrap: wrap;
  row-gap: 0.25rem;

  * {
    margin-left: 0.25rem;
  }
  .filename {
    margin: 0;
    font-weight: bold;
  }
}

.featured {
  display: flex;
  align-items: center;

  svg {
    height: 1rem;
    width: auto;
    margin-right: 0.25rem;
  }
}

.options {
  margin-bottom: var(--spacing-card-sm);
}

.full-width-input {
  width: 100%;
  margin-bottom: 0.5rem;
}
</style>
