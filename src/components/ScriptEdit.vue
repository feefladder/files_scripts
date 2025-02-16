<template>
	<NcModal v-if="showModal"
		size="full"
		:spread-navigation="true"
		@close="closeModal">
		<NcActions>
			<NcActionButton @click="saveScript">
				<template #icon>
					<Save :size="20" />
				</template>
			</NcActionButton>
		</NcActions>
		<div v-if="saving" style="display: inline-block;" class="icon-loading" />

		<div class="container-script-edit">
			<div class="script-details">
				<input v-model="scriptTitle"
					type="text"
					class="input-script-name"
					:placeholder="t('files_scripts', 'Script name')">
				<textarea v-model="scriptDescription"
					class="input-script-description"
					:placeholder="t('files_scripts', 'A short description of what this action will do …')"
					rows="6" />

				<NcCheckboxRadioSwitch type="switch" :checked="!!script.enabled" @update:checked="toggleEnabled">
					{{ t('files_scripts', 'Enable script') }}
				</NcCheckboxRadioSwitch>

				<NcCheckboxRadioSwitch type="switch" :checked="!!script.public" @update:checked="togglePublic">
					{{ t('files_scripts', 'Allow on public shares') }}
				</NcCheckboxRadioSwitch>

				<NcCheckboxRadioSwitch type="switch" :checked.sync="limitGroupsEnabled" @update:checked="toggleLimitGroupsEnabled">
					{{ t('files_scripts', 'Limit to groups') }}
				</NcCheckboxRadioSwitch>
				<NcSelect v-if="limitGroupsEnabled"
					v-model="limitGroups"
					:placeholder="t('files_scripts', 'Select groups allowed to use this action')"
					class="multi-input-groups"
					:options="groups"
					:multiple="true"
					:closeOnSelect="false"
					:tag-width="80"
				/>

				<NcCheckboxRadioSwitch type="switch" :checked.sync="limitMimesEnabled" @update:checked="toggleLimitMimes">
					{{ t('files_scripts', 'Limit to specific MIME type') }}
				</NcCheckboxRadioSwitch>
				<NcTextField v-if="limitMimesEnabled"
			 		:value.sync="mimetype"
				 	:label="t('files_scripts', 'MIME type (e.g. text/plain)')"
				 	trailing-button-icon="close"
				 	:show-trailing-button="!!mimetype"
					@trailing-button-click="mimetype = ''"
				/>

				<EditInputs :script-id="script.id" @changed="updateInputs" />
			</div>

			<div class="script-editor">
				<CodeMirror v-model="scriptProgram" style="height: 100%" :options="cmOption" />
			</div>
		</div>
	</NcModal>
</template>

<script lang="ts">
import Save from 'vue-material-design-icons/ContentSave.vue'
import { NcModal, NcActions, NcActionButton, NcCheckboxRadioSwitch, NcNoteCard, NcSelect, NcTextField } from '@nextcloud/vue'
import FreeSelect from './generic/FreeSelect.vue'
import EditInputs from './ScriptEdit/EditInputs.vue'

import 'codemirror/mode/lua/lua.js'
import 'codemirror/addon/edit/matchbrackets.js'
import 'codemirror/addon/hint/show-hint.js'
import { mapState } from 'vuex'
import { showError, showSuccess } from '@nextcloud/dialogs'
import { ScriptInput } from '../types/script'
import { api } from '../api/script'
import { translate as t } from '../l10n'
import axios from '@nextcloud/axios'
import { generateOcsUrl } from '@nextcloud/router'
const CodeMirror = require('vue-codemirror').codemirror

export default {
	name: 'ScriptEdit',
	components: {
		axios,
		NcModal,
		NcActions,
		NcActionButton,
		CodeMirror,
		NcCheckboxRadioSwitch,
		NcNoteCard,
		NcSelect,
		Save,
		EditInputs,
		FreeSelect,
		NcTextField,
	},
	data() {
		const mainBackgroundColor = getComputedStyle(document.querySelector('body'))
				.getPropertyValue('--color-main-background-rgb')
				.split(',')
		const darkMode = Number(mainBackgroundColor[0]) < 128
			&& Number(mainBackgroundColor[1]) < 128
			&& Number(mainBackgroundColor[2]) < 128

		return {
			scriptInputs: [],
			groups: [],
			dirtyInputs: false,
			saving: false,
			limitGroupsEnabled: false,
			limitMimesEnabled: false,
			cmOption: {
				tabSize: 4,
				styleActiveLine: true,
				lineNumbers: true,
				fixedGutter: false,
				line: true,
				foldGutter: false,
				styleSelectedText: true,
				matchBrackets: true,
				showCursorWhenSelecting: true,
				mode: 'text/x-lua',
				theme: darkMode ? 'material-darker' : 'idea',
				extraKeys: { Ctrl: 'autocomplete' },
				hintOptions: {
					completeSingle: true,
				},
			},
		}
	},
	computed: {
		...mapState({ script: 'selectedScript' }),
		showModal(): boolean {
			return !!this.script
		},
		scriptTitle: {
			get() {
				return this.script.title
			},
			set(value) {
				this.$store.commit('updateCurrentScript', { title: value })
			},
		},
		scriptProgram: {
			get() {
				return this.script.program
			},
			set(value) {
				this.$store.commit('updateCurrentScript', { program: value })
			},
		},
		scriptDescription: {
			get() {
				return this.script.description
			},
			set(value) {
				this.$store.commit('updateCurrentScript', { description: value })
			},
		},
		limitGroups: {
			get() {
				return this.script ? this.script.limitGroups : []
			},
			set(value) {
				this.$store.commit('updateCurrentScript', { limitGroups: value })
			},
		},
		mimetype: {
			get() {
				return this.script ? this.script.mimetype : ""
			},
			set(value) {
				this.$store.commit('updateCurrentScript', { mimetype: value })
			},
		}
	},

	mounted() {
		this.loadGroups()
		document.addEventListener("keydown", this.doSaveKeyboardShortcut);
	},

	beforeDestroy() {
		document.removeEventListener("keydown", this.doSaveKeyboardShortcut);
	},

	watch: {
		showModal(newValue) {
			newValue && this.remounted()

			// Hack to fix codemirror rendering issue
			if (newValue === true) {
				setTimeout(() => {
					const evt = document.createEvent('UIEvents')
					evt.initUIEvent('resize', true, false, window, 0)
					window.dispatchEvent(evt)
				}, 500)
			}
		},
	},

	methods: {
		t,
		remounted() {
			this.limitGroupsEnabled = this.limitGroups.length > 0
			this.limitMimesEnabled = !!this.mimetype
		},
		doSaveKeyboardShortcut(e) {
			const ctrlS = (e.keyCode === 83 && e.ctrlKey)
			if (this.showModal && ctrlS) {
				e.preventDefault()
				this.saveScript()
			}
		},
		saveScript() {
			const self = this
			this.saving = true
			this.saveScriptAsync()
				.then(() => {
					self.dirtyInputs = false
					showSuccess(t('files_scripts', 'Saved'), { timeout: 2000 })
				})
				.catch((error) => {
					let message = t('files_scripts', 'An error occurred during saving')
					if (error.response && error.response.data.error) {
						message = error.response.data.error
					}
					showError(message)
				})
				.then(() => {
					this.saving = false
				})
		},
		async saveScriptAsync() {
			await this.$store.dispatch('saveScript')
			if (this.dirtyInputs) {
				await api.updateScriptInputs(this.script, this.scriptInputs)
			}
		},
		updateInputs(scriptInputs: ScriptInput[]) {
			this.scriptInputs = scriptInputs
			this.dirtyInputs = true
		},
		closeModal() {
			this.$store.commit('clearSelected')
			this.dirtyInputs = false
			this.scriptInputs = []
		},
		toggleEnabled() {
			this.$store.commit('selectedToggleValue', 'enabled')
		},
		togglePublic() {
			this.$store.commit('selectedToggleValue', 'public')
		},
		toggleLimitGroupsEnabled() {
			if (!this.limitGroupsEnabled) {
				this.limitGroups = []
			}
		},
		toggleLimitMimes() {
			if (!this.limitMimesEnabled) {
				this.mimetype = ""
			}
		},
		async loadGroups() {
			const response = await axios.get(generateOcsUrl('cloud/groups'))
			if (response && response.data && response.data.ocs) {
				this.groups = response.data.ocs.data.groups
			}
		}
	},
}
</script>

<style scoped lang="scss">
@import 'codemirror/theme/material-darker.css';
@import 'codemirror/theme/idea.css';

@import 'codemirror/lib/codemirror.css';
@import '../../css/codemirror.css';

.container-script-edit {
	display: flex;
}

.script-details {
	flex: 0 0 25%;
	height: 90vh;
	padding: 6px;
	overflow-y: auto;
	overflow-x: clip;
}

.section-description {
	opacity: .7;
	margin-bottom: 16px;
}

.script-editor {
	flex: 1 0;
	border: solid 1px slategray;
	height: 90vh;
}

.input-script-name {
	width: 100%;
	font-size: large;
}

.input-script-description {
	width: 100%;
	resize: vertical;
}

.multi-input-groups {
	width: 100%;
}
</style>
