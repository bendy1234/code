<template>
  <ConfirmModalWrapper
    ref="modal_confirm"
    title="Are you sure you want to delete this instance?"
    description="If you proceed, all data for your instance will be removed. You will not be able to recover it."
    :has-to-type="false"
    proceed-label="Delete"
    @proceed="removeProfile"
  />
  <ModalWrapper ref="modalConfirmUnlock" header="Are you sure you want to unlock this instance?">
    <div class="modal-delete">
      <div
        class="markdown-body"
        v-html="
          'If you proceed, you will not be able to re-lock it without using the `Reinstall modpack` button.'
        "
      />
      <div class="input-group push-right">
        <button class="btn" @click="$refs.modalConfirmUnlock.hide()">
          <XIcon />
          Cancel
        </button>
        <button class="btn btn-danger" @click="unlockProfile">
          <LockIcon />
          Unlock
        </button>
      </div>
    </div>
  </ModalWrapper>

  <ModalWrapper ref="modalConfirmUnpair" header="Are you sure you want to unpair this instance?">
    <div class="modal-delete">
      <div
        class="markdown-body"
        v-html="
          'If you proceed, you will not be able to re-pair it without creating an entirely new instance.'
        "
      />
      <div class="input-group push-right">
        <button class="btn" @click="$refs.modalConfirmUnpair.hide()">
          <XIcon />
          Cancel
        </button>
        <button class="btn btn-danger" @click="unpairProfile">
          <XIcon />
          Unpair
        </button>
      </div>
    </div>
  </ModalWrapper>

  <ModalWrapper ref="changeVersionsModal" header="Change instance versions">
    <div class="change-versions-modal universal-body">
      <div class="input-row">
        <p class="input-label">Loader</p>
        <Chips v-model="loader" :items="loaders" :never-empty="false" />
      </div>
      <div class="input-row">
        <p class="input-label">Game Version</p>
        <div class="versions">
          <DropdownSelect
            v-model="gameVersion"
            :options="selectableGameVersions"
            name="Game Version Dropdown"
            render-up
          />
          <Checkbox v-model="showSnapshots" class="filter-checkbox" label="Include snapshots" />
        </div>
      </div>
      <div v-if="loader !== 'vanilla'" class="input-row">
        <p class="input-label">Loader Version</p>
        <DropdownSelect
          :model-value="selectableLoaderVersions[loaderVersionIndex]"
          :options="selectableLoaderVersions"
          :display-name="(option) => option?.id"
          name="Version selector"
          render-up
          @change="(value) => (loaderVersionIndex = value.index)"
        />
      </div>
      <div class="push-right input-group">
        <button class="btn" @click="$refs.changeVersionsModal.hide()">
          <XIcon />
          Cancel
        </button>
        <button
          class="btn btn-primary"
          :disabled="!isValid || !isChanged || editing"
          @click="saveGvLoaderEdits()"
        >
          <SaveIcon />
          {{ editing ? 'Saving...' : 'Save changes' }}
        </button>
      </div>
    </div>
  </ModalWrapper>
  <section class="card">
    <div class="label">
      <h3>
        <span class="label__title size-card-header">Instance</span>
      </h3>
    </div>
    <label for="instance-icon">
      <span class="label__title">Icon</span>
    </label>
    <div class="input-group">
      <Avatar :src="icon ? convertFileSrc(icon) : icon" size="md" class="project__icon" />
      <div class="input-stack">
        <button id="instance-icon" class="btn" @click="setIcon">
          <UploadIcon />
          Select icon
        </button>
        <button :disabled="!icon" class="btn" @click="resetIcon">
          <TrashIcon />
          Remove icon
        </button>
      </div>
    </div>

    <label for="project-name">
      <span class="label__title">Name</span>
    </label>
    <input
      id="profile-name"
      v-model="title"
      autocomplete="off"
      maxlength="80"
      type="text"
      :disabled="instance.linked_data"
    />

    <div class="adjacent-input">
      <label for="edit-versions">
        <span class="label__title">Edit mod loader/game versions</span>
        <span class="label__description">
          Allows you to change the mod loader, loader version, or game version of the instance.
        </span>
      </label>
      <button
        id="edit-versions"
        class="btn"
        :disabled="offline"
        @click="$refs.changeVersionsModal.show()"
      >
        <EditIcon />
        Edit versions
      </button>
    </div>

    <div class="adjacent-input">
      <label>
        <span class="label__title">Categories</span>
        <span class="label__description">
          Set the categories of this instance, for display in the library page. This is purely
          cosmetic.
        </span>
      </label>
      <multiselect
        v-model="groups"
        :options="availableGroups"
        :multiple="true"
        :searchable="true"
        :show-no-results="false"
        :close-on-select="false"
        :clear-search-on-select="false"
        :show-labels="false"
        :taggable="true"
        tag-placeholder="Add new category"
        placeholder="Select categories..."
        @tag="
          (newTag) => {
            groups.push(newTag.trim().substring(0, 32))
            availableGroups.push(newTag.trim().substring(0, 32))
          }
        "
      />
    </div>
  </section>
  <Card>
    <div class="label">
      <h3>
        <span class="label__title size-card-header">Java</span>
      </h3>
    </div>
    <div class="settings-group">
      <h3>Installation</h3>
      <Checkbox v-model="overrideJavaInstall" label="Override global java installations" />
      <JavaSelector v-model="javaInstall" :disabled="!overrideJavaInstall" />
    </div>
    <hr class="card-divider" />
    <div class="settings-group">
      <h3>Java arguments</h3>
      <Checkbox v-model="overrideJavaArgs" label="Override global java arguments" />
      <input
        id="java-args"
        v-model="javaArgs"
        autocomplete="off"
        :disabled="!overrideJavaArgs"
        type="text"
        class="installation-input"
        placeholder="Enter java arguments..."
      />
    </div>
    <div class="settings-group">
      <h3>Environment variables</h3>
      <Checkbox v-model="overrideEnvVars" label="Override global environment variables" />
      <input
        v-model="envVars"
        autocomplete="off"
        :disabled="!overrideEnvVars"
        type="text"
        class="installation-input"
        placeholder="Enter environment variables..."
      />
    </div>
    <hr class="card-divider" />
    <div class="settings-group">
      <h3>Java memory</h3>
      <Checkbox v-model="overrideMemorySettings" label="Override global memory settings" />
      <Slider
        v-model="memory.maximum"
        :disabled="!overrideMemorySettings"
        :min="512"
        :max="maxMemory"
        :step="64"
        unit="mb"
      />
    </div>
  </Card>
  <Card>
    <div class="label">
      <h3>
        <span class="label__title size-card-header">Window</span>
      </h3>
    </div>
    <div class="adjacent-input">
      <Checkbox v-model="overrideWindowSettings" label="Override global window settings" />
    </div>
    <div class="adjacent-input">
      <label for="fullscreen">
        <span class="label__title">Fullscreen</span>
        <span class="label__description">
          Make the game start in full screen when launched (using options.txt).
        </span>
      </label>
      <Toggle
        id="fullscreen"
        :model-value="fullscreenSetting"
        :checked="fullscreenSetting"
        :disabled="!overrideWindowSettings"
        @update:model-value="
          (e) => {
            fullscreenSetting = e
          }
        "
      />
    </div>
    <div class="adjacent-input">
      <label for="width">
        <span class="label__title">Width</span>
        <span class="label__description"> The width of the game window when launched. </span>
      </label>
      <input
        id="width"
        v-model="resolution[0]"
        autocomplete="off"
        :disabled="!overrideWindowSettings || fullscreenSetting"
        type="number"
        placeholder="Enter width..."
      />
    </div>
    <div class="adjacent-input">
      <label for="height">
        <span class="label__title">Height</span>
        <span class="label__description"> The height of the game window when launched. </span>
      </label>
      <input
        id="height"
        v-model="resolution[1]"
        autocomplete="off"
        :disabled="!overrideWindowSettings || fullscreenSetting"
        type="number"
        class="input"
        placeholder="Enter height..."
      />
    </div>
  </Card>
  <Card>
    <div class="label">
      <h3>
        <span class="label__title size-card-header">Hooks</span>
      </h3>
    </div>
    <div class="adjacent-input">
      <Checkbox v-model="overrideHooks" label="Override global hooks" />
    </div>
    <div class="adjacent-input">
      <label for="pre-launch">
        <span class="label__title">Pre launch</span>
        <span class="label__description"> Ran before the instance is launched. </span>
      </label>
      <input
        id="pre-launch"
        v-model="hooks.pre_launch"
        autocomplete="off"
        :disabled="!overrideHooks"
        type="text"
        placeholder="Enter pre-launch command..."
      />
    </div>
    <div class="adjacent-input">
      <label for="wrapper">
        <span class="label__title">Wrapper</span>
        <span class="label__description"> Wrapper command for launching Minecraft. </span>
      </label>
      <input
        id="wrapper"
        v-model="hooks.wrapper"
        autocomplete="off"
        :disabled="!overrideHooks"
        type="text"
        placeholder="Enter wrapper command..."
      />
    </div>
    <div class="adjacent-input">
      <label for="post-exit">
        <span class="label__title">Post exit</span>
        <span class="label__description"> Ran after the game closes. </span>
      </label>
      <input
        id="post-exit"
        v-model="hooks.post_exit"
        autocomplete="off"
        :disabled="!overrideHooks"
        type="text"
        placeholder="Enter post-exit command..."
      />
    </div>
  </Card>
  <Card v-if="instance.linked_data">
    <div class="label">
      <h3>
        <span class="label__title size-card-header">Modpack</span>
      </h3>
    </div>
    <div class="adjacent-input">
      <label for="general-modpack-info">
        <span class="label__description"> <strong>Modpack: </strong> {{ instance.name }} </span>
        <span class="label__description">
          <strong>Version: </strong>
          {{
            installedVersionData?.name != null
              ? installedVersionData.name.charAt(0).toUpperCase() +
                installedVersionData.name.slice(1)
              : getLocalVersion(props.instance.path)
          }}
        </span>
      </label>
    </div>
    <div v-if="!isPackLocked" class="adjacent-input">
      <Card class="unlocked-instance">
        This is an unlocked instance. There may be unexpected behaviour unintended by the modpack
        creator.
      </Card>
    </div>
    <div v-else class="adjacent-input">
      <label for="unlock-profile">
        <span class="label__title">Unlock instance</span>
        <span class="label__description">
          Allows modifications to the instance, which allows you to add projects to the modpack. The
          pack will remain linked, and you can still change versions. Only mods listed in the
          modpack will be modified on version changes.
        </span>
      </label>
      <Button id="unlock-profile" @click="$refs.modalConfirmUnlock.show()">
        <LockIcon /> Unlock
      </Button>
    </div>

    <div class="adjacent-input">
      <label for="unpair-profile">
        <span class="label__title">Unpair instance</span>
        <span class="label__description">
          Removes the link to an external Modrinth modpack on the instance. This allows you to edit
          modpacks you download through the browse page but you will not be able to update the
          instance from a new version of a modpack if you do this.
        </span>
      </label>
      <Button id="unpair-profile" @click="$refs.modalConfirmUnpair.show()">
        <XIcon /> Unpair
      </Button>
    </div>

    <div v-if="instance.linked_data.project_id" class="adjacent-input">
      <label for="change-modpack-version">
        <span class="label__title">Change modpack version</span>
        <span class="label__description">
          Changes to another version of the modpack, allowing upgrading or downgrading. This will
          replace all files marked as relevant to the modpack.
        </span>
      </label>

      <Button
        id="change-modpack-version"
        :disabled="inProgress || installing"
        @click="modpackVersionModal.show()"
      >
        <SwapIcon />
        Change modpack version
      </Button>
    </div>
    <div class="adjacent-input">
      <label for="repair-modpack">
        <span class="label__title">Reinstall modpack</span>
        <span class="label__description">
          Removes all projects and reinstalls Modrinth modpack. Use this to fix unexpected behaviour
          if your instance is diverging from the Modrinth modpack. This also re-locks the instance.
        </span>
      </label>
      <Button id="repair-modpack" color="highlight" :disabled="offline" @click="repairModpack">
        <DownloadIcon /> Reinstall
      </Button>
    </div>
  </Card>
  <Card>
    <div class="label">
      <h3>
        <span class="label__title size-card-header">Instance management</span>
      </h3>
    </div>
    <div v-if="instance.install_stage == 'installed'" class="adjacent-input">
      <label for="duplicate-profile">
        <span class="label__title">Duplicate instance</span>
        <span class="label__description">
          Creates another copy of the instance, including saves, configs, mods, and everything.
        </span>
      </label>
      <Button
        id="repair-profile"
        :disabled:="installing || inProgress || offline"
        @click="duplicateProfile"
      >
        <ClipboardCopyIcon /> Duplicate
      </Button>
    </div>
    <div class="adjacent-input">
      <label for="repair-profile">
        <span class="label__title">Repair instance</span>
        <span class="label__description">
          Reinstalls Minecraft dependencies and checks for corruption. Use this if your game is not
          launching due to launcher-related errors.
        </span>
      </label>
      <Button
        id="repair-profile"
        color="highlight"
        :disabled="installing || inProgress || repairing || offline"
        @click="repairProfile(true)"
      >
        <HammerIcon /> Repair
      </Button>
    </div>
    <div class="adjacent-input">
      <label for="delete-profile">
        <span class="label__title">Delete instance</span>
        <span class="label__description">
          Fully removes a instance from the disk. Be careful, as once you delete a instance there is
          no way to recover it.
        </span>
      </label>
      <Button
        id="delete-profile"
        color="danger"
        :disabled="removing"
        @click="$refs.modal_confirm.show()"
      >
        <TrashIcon /> Delete
      </Button>
    </div>
  </Card>
  <ModpackVersionModal
    v-if="instance.linked_data"
    ref="modpackVersionModal"
    :instance="instance"
    :versions="props.versions"
  />
</template>

<script setup>
import {
  TrashIcon,
  UploadIcon,
  EditIcon,
  XIcon,
  SaveIcon,
  LockIcon,
  HammerIcon,
  DownloadIcon,
  ClipboardCopyIcon,
} from '@modrinth/assets'
import { Button, Toggle, Card, Slider, Checkbox, Avatar, Chips, DropdownSelect } from '@modrinth/ui'
import { SwapIcon } from '@/assets/icons'

import { Multiselect } from 'vue-multiselect'
import { useRouter } from 'vue-router'
import {
  duplicate,
  edit,
  edit_icon,
  get_optimal_jre_key,
  install,
  list,
  remove,
  update_repair_modrinth,
} from '@/helpers/profile.js'
import { computed, readonly, ref, shallowRef, watch } from 'vue'
import { get_max_memory } from '@/helpers/jre.js'
import { get } from '@/helpers/settings.js'
import JavaSelector from '@/components/ui/JavaSelector.vue'
import { convertFileSrc } from '@tauri-apps/api/core'
import { open } from '@tauri-apps/plugin-dialog'
import { get_loader_versions } from '@/helpers/metadata.js'
import { get_game_versions, get_loaders } from '@/helpers/tags.js'
import { handleError } from '@/store/notifications.js'
import { useBreadcrumbs } from '@/store/breadcrumbs'
import ModpackVersionModal from '@/components/ui/ModpackVersionModal.vue'
import { trackEvent } from '@/helpers/analytics'
import ModalWrapper from '@/components/ui/modal/ModalWrapper.vue'
import ConfirmModalWrapper from '@/components/ui/modal/ConfirmModalWrapper.vue'

const breadcrumbs = useBreadcrumbs()

const router = useRouter()

const props = defineProps({
  instance: {
    type: Object,
    required: true,
  },
  offline: {
    type: Boolean,
    default: false,
  },
  versions: {
    type: Array,
    required: true,
  },
})

const title = ref(props.instance.name)
const icon = ref(props.instance.icon_path)
const groups = ref(props.instance.groups)

const modpackVersionModal = ref(null)

const instancesList = await list()
const availableGroups = ref([
  ...new Set(
    instancesList.reduce((acc, obj) => {
      return acc.concat(obj.groups)
    }, []),
  ),
])

async function resetIcon() {
  icon.value = null
  await edit_icon(props.instance.path, null).catch(handleError)
  trackEvent('InstanceRemoveIcon')
}

async function setIcon() {
  const value = await open({
    multiple: false,
    filters: [
      {
        name: 'Image',
        extensions: ['png', 'jpeg', 'svg', 'webp', 'gif', 'jpg'],
      },
    ],
  })

  if (!value) return

  icon.value = value.path ?? value
  await edit_icon(props.instance.path, icon.value).catch(handleError)

  trackEvent('InstanceSetIcon')
}

const globalSettings = await get().catch(handleError)

const modalConfirmUnlock = ref(null)
const modalConfirmUnpair = ref(null)

const overrideJavaInstall = ref(!!props.instance.java_path)
const optimalJava = readonly(await get_optimal_jre_key(props.instance.path).catch(handleError))
const javaInstall = ref({ path: optimalJava.path ?? props.instance.java_path })

const overrideJavaArgs = ref(props.instance.extra_launch_args?.length !== undefined)
const javaArgs = ref(
  (props.instance.extra_launch_args ?? globalSettings.extra_launch_args).join(' '),
)

const overrideEnvVars = ref(props.instance.custom_env_vars?.length !== undefined)
const envVars = ref(
  (props.instance.custom_env_vars ?? globalSettings.custom_env_vars)
    .map((x) => x.join('='))
    .join(' '),
)

const overrideMemorySettings = ref(!!props.instance.memory)
const memory = ref(props.instance.memory ?? globalSettings.memory)
const maxMemory = Math.floor((await get_max_memory().catch(handleError)) / 1024)

const overrideWindowSettings = ref(
  !!props.instance.game_resolution || !!props.instance.force_fullscreen,
)
const resolution = ref(props.instance.game_resolution ?? globalSettings.game_resolution)
const overrideHooks = ref(
  props.instance.hooks.pre_launch || props.instance.hooks.wrapper || props.instance.hooks.post_exit,
)
const hooks = ref(props.instance.hooks ?? globalSettings.hooks)

const fullscreenSetting = ref(!!props.instance.force_fullscreen)

const unlinkModpack = ref(false)

const inProgress = ref(false)
const installing = computed(() => props.instance.install_stage !== 'installed')
const installedVersion = computed(() => props.instance?.linked_data?.version_id)
const installedVersionData = computed(() => {
  if (!installedVersion.value) return null
  return props.versions.find((version) => version.id === installedVersion.value)
})

watch(
  [
    title,
    groups,
    groups,
    overrideJavaInstall,
    javaInstall,
    overrideJavaArgs,
    javaArgs,
    overrideEnvVars,
    envVars,
    overrideMemorySettings,
    memory,
    overrideWindowSettings,
    resolution,
    fullscreenSetting,
    overrideHooks,
    hooks,
    unlinkModpack,
  ],
  async () => {
    await edit(props.instance.path, editProfileObject.value)
  },
  { deep: true },
)

const getLocalVersion = (path) => {
  const pathSlice = path.split(' ').slice(-1).toString()
  // If the path ends in (1), (2), etc. it's a duplicate instance and no version can be obtained.
  if (/^\(\d\)/.test(pathSlice)) {
    return 'Unknown'
  }
  return pathSlice
}

const editProfileObject = computed(() => {
  const editProfile = {
    name: title.value.trim().substring(0, 32) ?? 'Instance',
    groups: groups.value.map((x) => x.trim().substring(0, 32)).filter((x) => x.length > 0),
    loader_version: props.instance.loader_version,
    linked_data: props.instance.linked_data,
    java: {},
    hooks: {},
  }

  if (overrideJavaInstall.value) {
    if (javaInstall.value.path !== '') {
      editProfile.java_path = javaInstall.value.path.replace('java.exe', 'javaw.exe')
    }
  }

  if (overrideJavaArgs.value) {
    editProfile.extra_launch_args = javaArgs.value.trim().split(/\s+/).filter(Boolean)
  }

  if (overrideEnvVars.value) {
    editProfile.custom_env_vars = envVars.value
      .trim()
      .split(/\s+/)
      .filter(Boolean)
      .map((x) => x.split('=').filter(Boolean))
  }

  if (overrideMemorySettings.value) {
    editProfile.memory = memory.value
  }

  if (overrideWindowSettings.value) {
    editProfile.force_fullscreen = fullscreenSetting.value

    if (!fullscreenSetting.value) {
      editProfile.game_resolution = resolution.value
    }
  }

  if (overrideHooks.value) {
    editProfile.hooks = hooks.value
  }

  if (unlinkModpack.value) {
    editProfile.linked_data = null
  }

  breadcrumbs.setName('Instance', editProfile.name)

  return editProfile
})

const repairing = ref(false)

async function duplicateProfile() {
  await duplicate(props.instance.path).catch(handleError)
  trackEvent('InstanceDuplicate', {
    loader: props.instance.loader,
    game_version: props.instance.game_version,
  })
}

async function repairProfile(force) {
  repairing.value = true
  await install(props.instance.path, force).catch(handleError)
  repairing.value = false

  trackEvent('InstanceRepair', {
    loader: props.instance.loader,
    game_version: props.instance.game_version,
  })
}

async function unpairProfile() {
  const editProfile = props.instance
  editProfile.linked_data = null
  await edit(props.instance.path, editProfile)
  installedVersion.value = null
  installedVersionData.value = null
  modalConfirmUnpair.value.hide()
}

async function unlockProfile() {
  const editProfile = props.instance
  editProfile.linked_data.locked = false
  await edit(props.instance.path, editProfile)
  modalConfirmUnlock.value.hide()
}

const isPackLocked = computed(() => {
  return props.instance.linked_data && props.instance.linked_data.locked
})

async function repairModpack() {
  inProgress.value = true
  await update_repair_modrinth(props.instance.path).catch(handleError)
  inProgress.value = false

  trackEvent('InstanceRepair', {
    loader: props.instance.loader,
    game_version: props.instance.game_version,
  })
}

const removing = ref(false)
async function removeProfile() {
  removing.value = true
  await remove(props.instance.path).catch(handleError)
  removing.value = false

  trackEvent('InstanceRemove', {
    loader: props.instance.loader,
    game_version: props.instance.game_version,
  })

  await router.push({ path: '/' })
}

const changeVersionsModal = ref(null)
const showSnapshots = ref(false)

const [
  fabric_versions,
  forge_versions,
  quilt_versions,
  neoforge_versions,
  all_game_versions,
  loaders,
] = await Promise.all([
  get_loader_versions('fabric').then(shallowRef).catch(handleError),
  get_loader_versions('forge').then(shallowRef).catch(handleError),
  get_loader_versions('quilt').then(shallowRef).catch(handleError),
  get_loader_versions('neo').then(shallowRef).catch(handleError),
  get_game_versions().then(shallowRef).catch(handleError),
  get_loaders()
    .then((value) =>
      value
        .filter((item) => item.supported_project_types.includes('modpack'))
        .map((item) => item.name.toLowerCase()),
    )
    .then(ref)
    .catch(handleError),
])
loaders.value.unshift('vanilla')

const loader = ref(props.instance.loader)
const gameVersion = ref(props.instance.game_version)
const selectableGameVersions = computed(() => {
  return all_game_versions.value
    .filter((item) => {
      let defaultVal = item.version_type === 'release' || showSnapshots.value
      if (loader.value === 'fabric') {
        defaultVal &= fabric_versions.value.gameVersions.some((x) => item.version === x.id)
      } else if (loader.value === 'forge') {
        defaultVal &= forge_versions.value.gameVersions.some((x) => item.version === x.id)
      } else if (loader.value === 'quilt') {
        defaultVal &= quilt_versions.value.gameVersions.some((x) => item.version === x.id)
      } else if (loader.value === 'neoforge') {
        defaultVal &= neoforge_versions.value.gameVersions.some((x) => item.version === x.id)
      }

      return defaultVal
    })
    .map((item) => item.version)
})

const selectableLoaderVersions = computed(() => {
  if (gameVersion.value) {
    if (loader.value === 'fabric') {
      return fabric_versions.value.gameVersions[0].loaders
    } else if (loader.value === 'forge') {
      return forge_versions.value.gameVersions.find((item) => item.id === gameVersion.value).loaders
    } else if (loader.value === 'quilt') {
      return quilt_versions.value.gameVersions[0].loaders
    } else if (loader.value === 'neoforge') {
      return neoforge_versions.value.gameVersions.find((item) => item.id === gameVersion.value)
        .loaders
    }
  }
  return []
})
const loaderVersionIndex = ref(
  selectableLoaderVersions.value.findIndex((x) => x.id === props.instance.loader_version),
)

const isValid = computed(() => {
  return (
    selectableGameVersions.value.includes(gameVersion.value) &&
    (loaderVersionIndex.value >= 0 || loader.value === 'vanilla')
  )
})

const isChanged = computed(() => {
  return (
    loader.value !== props.instance.loader ||
    gameVersion.value !== props.instance.game_version ||
    (loaderVersionIndex.value >= 0 &&
      selectableLoaderVersions.value[loaderVersionIndex.value].id !== props.instance.loader_version)
  )
})

watch(loader, () => (loaderVersionIndex.value = 0))

const editing = ref(false)
async function saveGvLoaderEdits() {
  editing.value = true

  const editProfile = editProfileObject.value
  editProfile.loader = loader.value
  editProfile.game_version = gameVersion.value

  if (loader.value !== 'vanilla') {
    editProfile.loader_version = selectableLoaderVersions.value[loaderVersionIndex.value].id
  } else {
    loaderVersionIndex.value = -1
  }
  await edit(props.instance.path, editProfile).catch(handleError)
  await repairProfile(false)

  editing.value = false
  changeVersionsModal.value.hide()
}
</script>

<style scoped lang="scss">
.change-versions-modal {
  display: flex;
  flex-direction: column;
  gap: 1rem;

  :deep(.animated-dropdown .options) {
    max-height: 13.375rem;
  }

  .input-label {
    font-size: 1rem;
    font-weight: bolder;
    color: var(--color-contrast);
    margin-bottom: 0.5rem;
  }

  .versions {
    display: flex;
    flex-direction: row;
    gap: 1rem;
  }
}

.settings-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin: 1rem 0;

  h3 {
    margin: 0;
  }
}

.installation-input {
  width: 100%;
}

:deep(button.checkbox) {
  border: none;
}

.unlocked-instance {
  background-color: var(--color-bg);
}

.modal-delete {
  padding: var(--gap-lg);
  display: flex;
  flex-direction: column;

  .markdown-body {
    margin-bottom: 1rem;
  }

  .confirmation-label {
    margin-bottom: 0.5rem;
  }

  .confirmation-text {
    padding-right: 0.25ch;
    margin: 0 0.25rem;
  }

  .confirmation-input {
    input {
      width: 20rem;
      max-width: 100%;
    }
  }

  .button-group {
    margin-left: auto;
    margin-top: 1.5rem;
  }
}
</style>
