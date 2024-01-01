<template>
	<Modal
		:name="WORKFLOW_CREDENTIALS_REPLACE_MODAL_KEY"
		width="65%"
		maxHeight="80%"
		:title="$locale.baseText('credentialsReplace.title')"
		:eventBus="modalBus"
		:scrollable="true"
	>
		<template #content>
			<div
				v-loading="isLoading"
				class="credentials-replace"
				data-test-id="credentials-replace-dialog"
			>
				<el-row>
					<el-col :span="10" class="input-name">
						{{ $locale.baseText('credentialsReplace.currentCredential') + ':' }}
					</el-col>
					<el-col :span="14" class="ignore-key-press">
						<n8n-select
							v-model="currentCredential"
							placeholder="Current Credential"
							filterable
							:disabled="readOnlyEnv"
							:limit-popper-width="true"
							data-test-id="credentials-replace-source-credential"
						>
							<n8n-option
								v-for="item in usedCredentials"
								:key="item.id"
								:label="`${item.name} - ${item.type}`"
								:value="item.id"
							>
							</n8n-option>
						</n8n-select>
					</el-col>
					<el-col :span="14" class="ignore-key-press"> </el-col>
				</el-row>
				<el-row>
					<el-col :span="10" class="setting-name">
						{{ $locale.baseText('credentialsReplace.targetCredential') + ':' }}
					</el-col>
					<el-col :span="14" class="ignore-key-press">
						<n8n-select
							v-model="targetCredential"
							placeholder="Target Credential"
							filterable
							:disabled="readOnlyEnv"
							:limit-popper-width="true"
							data-test-id="credentials-replace-target-credential"
						>
							<n8n-option
								v-for="item in availableCredentials"
								:key="item.id"
								:label="item.name"
								:value="item.id"
							>
							</n8n-option>
						</n8n-select>
					</el-col>
					<el-col :span="14" class="ignore-key-press"> </el-col>
				</el-row>
			</div>
		</template>
		<template #footer>
			<div class="action-buttons" data-test-id="credentials-replace-save-button">
				<n8n-button
					:disabled="readOnlyEnv || !targetCredential"
					:label="$locale.baseText('credentialsReplace.save')"
					size="large"
					float="right"
					@click="replaceCredentials"
				/>
			</div>
		</template>
	</Modal>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { mapStores } from 'pinia';

import { genericHelpers } from '@/mixins/genericHelpers';
import { useToast } from '@/composables/useToast';
import type { ICredentialsResponse, INodeUi, IWorkflowDb } from '@/Interface';
import Modal from '@/components/Modal.vue';
import { WORKFLOW_CREDENTIALS_REPLACE_MODAL_KEY } from '@/constants';
import { useUIStore } from '@/stores/ui.store';
import { useSettingsStore } from '@/stores/settings.store';
import { useUsersStore } from '@/stores/users.store';
import { useRootStore } from '@/stores/n8nRoot.store';
import { useWorkflowsEEStore } from '@/stores/workflows.ee.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { useCredentialsStore } from '@/stores/credentials.store';
import { createEventBus } from 'n8n-design-system/utils';
import { useExternalHooks } from '@/composables/useExternalHooks';
import { INodeCredentials, INode } from 'n8n-workflow';

export default defineComponent({
	name: 'CredentialsReplace',
	mixins: [genericHelpers],
	components: {
		Modal,
	},
	setup() {
		const externalHooks = useExternalHooks();

		return {
			externalHooks,
			...useToast(),
		};
	},
	data() {
		return {
			currentCredential: null,
			targetCredential: null,
			workflowSettings: {},
			isLoading: false,
			modalBus: createEventBus(),
			WORKFLOW_CREDENTIALS_REPLACE_MODAL_KEY,
		};
	},
	computed: {
		...mapStores(
			useRootStore,
			useUsersStore,
			useSettingsStore,
			useWorkflowsStore,
			useCredentialsStore,
			useWorkflowsEEStore,
			useUIStore,
		),
		currentCredentialType() {
			return this.credentials.filter((x) => x.id === this.currentCredential)[0]
				? this.credentials.filter((x) => x.id === this.currentCredential)[0].type
				: '';
			// return this.credentials.filter((x) => x.id === this.currentCredential)[0].type;
		},
		targetCredentialData() {
			return this.credentials.filter((x) => x.id === this.targetCredential)[0]
				? this.credentials.filter((x) => x.id === this.targetCredential)[0]
				: {};
			// return this.credentials.filter((x) => x.id === this.currentCredential)[0].type;
		},
		credentials(): ICredentialsResponse[] {
			return this.credentialsStore.allCredentials;
		},
		availableCredentials(): ICredentialsResponse[] {
			return this.credentials.filter(
				(x) => x.type === this.currentCredentialType && x.id !== this.currentCredential,
			);
		},
		usedCredentials() {
			return this.getUsedCredentials();
			// return this.workflowsStore.usedCredentials;
		},
		workflowName(): string {
			return this.workflowsStore.workflowName;
		},
		workflowId(): string {
			return this.workflowsStore.workflowId;
		},
		workflow(): IWorkflowDb {
			return this.workflowsStore.workflow;
		},
	},
	methods: {
		getUsedCredentials() {
			const usedCredentialsIds = new Set<string>();
			const usedCredentials = [];

			this.workflow.nodes.forEach((node: INodeUi) => {
				if (!node.credentials) {
					return;
				}

				Object.entries(node.credentials).forEach(([credentialType, credential]) => {
					if (!credential?.id || usedCredentialsIds.has(credential.id)) {
						return;
					}

					usedCredentialsIds.add(credential.id);

					usedCredentials.push({
						type: credentialType,
						name: credential.name,
						id: credential.id,
					});
				});
			});

			return usedCredentials;
		},
		replaceCredentials() {
			this.isLoading = true;

			let replacedCredentials = 0;

			this.workflow.nodes.forEach((node: INodeUi) => {
				const nodeCredentials: INodeCredentials | undefined = (node as unknown as INode)
					.credentials;
				if (this.currentCredentialType && nodeCredentials?.[this.currentCredentialType]) {
					(node.credentials as INodeCredentials)[this.currentCredentialType] =
						this.targetCredentialData;

					replacedCredentials += 1;
				}
			});

			this.uiStore.stateIsDirty = true;

			this.isLoading = false;

			this.showMessage({
				title: this.$locale.baseText('credentialsReplace.showMessage.title', {
					interpolate: { replacedCredentials },
				}),
				type: 'success',
			});

			this.modalBus.emit('close');
		},
	},
	watch: {
		currentCredential: function (val, oldVal) {
			console.log(val);
			this.targetCredential = null;
		},
	},
});
</script>

<style scoped lang="scss">
.credentials-replace {
	font-size: var(--font-size-s);
	.el-row {
		padding: 0.25em 0;
	}
}

.input-name {
	line-height: 32px;

	svg {
		display: inline-flex;
		opacity: 0;
		transition: opacity 0.3s ease;
	}

	&:hover {
		svg {
			opacity: 1;
		}
	}
}
</style>
