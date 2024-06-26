<template>
	<n8n-tabs
		:options="options"
		:model-value="modelValue"
		@update:model-value="onTabSelect"
		@tooltip-click="onTooltipClick"
	/>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { mapStores } from 'pinia';
import {
	BUILTIN_NODES_DOCS_URL,
	COMMUNITY_NODES_INSTALLATION_DOCS_URL,
	NPM_PACKAGE_DOCS_BASE_URL,
} from '@/constants';
import type { INodeUi, ITab } from '@/Interface';
import { useNDVStore } from '@/stores/ndv.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import type { INodeTypeDescription } from 'n8n-workflow';
import { NodeConnectionType } from 'n8n-workflow';

import { isCommunityPackageName } from '@/utils/nodeTypesUtils';
import { useExternalHooks } from '@/composables/useExternalHooks';

export default defineComponent({
	name: 'NodeSettingsTabs',
	props: {
		modelValue: {
			type: String,
			default: '',
		},
		nodeType: {},
		pushRef: {
			type: String,
		},
	},
	setup() {
		const externalHooks = useExternalHooks();
		return {
			externalHooks,
		};
	},
	computed: {
		...mapStores(useNDVStore, useWorkflowsStore),
		activeNode(): INodeUi | null {
			return this.ndvStore.activeNode;
		},
		documentationUrl(): string {
			const nodeType = this.nodeType as INodeTypeDescription | null;

			if (!nodeType) {
				return '';
			}

			if (nodeType.documentationUrl && nodeType.documentationUrl.startsWith('http')) {
				return nodeType.documentationUrl;
			}

			const utmTags =
				'?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link' +
				'&utm_campaign=' +
				nodeType.name;

			// Built-in node documentation available via its codex entry
			const primaryDocUrl = nodeType.codex?.resources?.primaryDocumentation?.[0]?.url;
			if (primaryDocUrl) {
				return primaryDocUrl + utmTags;
			}

			if (this.isCommunityNode) {
				return `${NPM_PACKAGE_DOCS_BASE_URL}${nodeType.name.split('.')[0]}`;
			}

			// Fallback to the root of the node documentation
			return BUILTIN_NODES_DOCS_URL + utmTags;
		},
		isCommunityNode(): boolean {
			const nodeType = this.nodeType as INodeTypeDescription | null;
			if (nodeType) {
				return isCommunityPackageName(nodeType.name);
			}
			return false;
		},
		packageName(): string {
			const nodeType = this.nodeType as INodeTypeDescription;
			return nodeType.name.split('.')[0];
		},
		options(): ITab[] {
			const options: ITab[] = [
				{
					label: this.$locale.baseText('nodeSettings.parameters'),
					value: 'params',
				},
			];
			if (this.documentationUrl) {
				options.push({
					label: this.$locale.baseText('nodeSettings.docs'),
					value: 'docs',
					href: this.documentationUrl,
				});
			}
			if (this.isCommunityNode) {
				options.push({
					icon: 'cube',
					value: 'communityNode',
					align: 'right',
					tooltip: this.$locale.baseText('generic.communityNode.tooltip', {
						interpolate: {
							docUrl: COMMUNITY_NODES_INSTALLATION_DOCS_URL,
							packageName: this.packageName,
						},
					}),
				});
			}
			// If both tabs have align right, both will have excessive left margin
			const pushCogRight = this.isCommunityNode ? false : true;
			options.push({
				icon: 'cog',
				value: 'settings',
				align: pushCogRight ? 'right' : undefined,
			});

			return options;
		},
	},
	methods: {
		onTabSelect(tab: string) {
			if (tab === 'docs' && this.nodeType) {
				void this.externalHooks.run('dataDisplay.onDocumentationUrlClick', {
					nodeType: this.nodeType as INodeTypeDescription,
					documentationUrl: this.documentationUrl,
				});
				this.$telemetry.track('User clicked ndv link', {
					node_type: this.activeNode.type,
					workflow_id: this.workflowsStore.workflowId,
					push_ref: this.pushRef,
					pane: NodeConnectionType.Main,
					type: 'docs',
				});
			}

			if (tab === 'settings' && this.nodeType) {
				this.$telemetry.track('User viewed node settings', {
					node_type: (this.nodeType as INodeTypeDescription).name,
					workflow_id: this.workflowsStore.workflowId,
				});
			}

			if (tab === 'settings' || tab === 'params') {
				this.$emit('update:modelValue', tab);
			}
		},
		onTooltipClick(tab: string, event: MouseEvent) {
			if (tab === 'communityNode' && (event.target as Element).localName === 'a') {
				this.$telemetry.track('user clicked cnr docs link', { source: 'node details view' });
			}
		},
	},
});
</script>

<style lang="scss">
#communityNode > div {
	cursor: auto;

	&:hover {
		color: unset;
	}
}
</style>
