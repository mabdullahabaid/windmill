<script lang="ts">
	import { goto } from '$lib/navigation'
	import { base } from '$lib/base'

	import { Button, FileInput } from '$lib/components/common'
	import Drawer from '$lib/components/common/drawer/Drawer.svelte'
	import DrawerContent from '$lib/components/common/drawer/DrawerContent.svelte'
	import { LayoutDashboard, Loader2, Plus } from 'lucide-svelte'
	import { importStore } from '../apps/store'
	import { RawAppService } from '$lib/gen'
	import { workspaceStore } from '$lib/stores'
	import Path from '../Path.svelte'
	import Tooltip from '../Tooltip.svelte'
	import YAML from 'yaml'

	let drawer: Drawer | undefined = undefined
	let rawAppDrawer: Drawer | undefined = undefined
	let pendingRaw: string = ''
	let pendingCode: string = ''
	let summary: string = ''
	let path: string = ''
	let pathError: string = ''

	let importType: 'yaml' | 'json' = 'yaml'

	async function importRaw() {
		$importStore = importType === 'yaml' ? YAML.parse(pendingRaw) : JSON.parse(pendingRaw)
		await goto('/apps/add?nodraft=true')
		drawer?.closeDrawer?.()
	}

	async function importRawApp() {
		await RawAppService.createRawApp({
			workspace: $workspaceStore!,
			requestBody: {
				path,
				summary,
				value: pendingCode
			}
		})
		await goto(`/apps/get_raw/0/${path}`)
		rawAppDrawer?.closeDrawer?.()
	}
</script>

<!-- Buttons -->
<div class="flex flex-row gap-2">
	<Button
		size="sm"
		spacingSize="xl"
		startIcon={{ icon: Plus }}
		href="{base}/apps/add?nodraft=true"
		color="marine"
		dropdownItems={[
			{
				label: 'Import low-code app from YAML',
				onClick: () => {
					drawer?.toggleDrawer?.()
					importType = 'yaml'
				}
			},
			{
				label: 'Import low-code app from JSON',
				onClick: () => {
					drawer?.toggleDrawer?.()
					importType = 'json'
				}
			},
			{
				label: 'Import app in React/Vue/Svelte',
				onClick: () => rawAppDrawer?.toggleDrawer?.()
			}
		]}
	>
		<div class="flex flex-row items-center">
			App <LayoutDashboard class="ml-1.5" size={18} />
		</div>
	</Button>
</div>

<!-- Raw JSON -->
<Drawer bind:this={drawer} size="800px">
	<DrawerContent
		title={'Import low-code app from ' + (importType === 'yaml' ? 'YAML' : 'JSON')}
		on:close={() => drawer?.toggleDrawer?.()}
	>
		{#await import('$lib/components/SimpleEditor.svelte')}
			<Loader2 class="animate-spin" />
		{:then Module}
			<Module.default
				bind:code={pendingRaw}
				lang={importType}
				class="h-full"
				fixedOverflowWidgets={false}
			/>
		{/await}
		<svelte:fragment slot="actions">
			<Button size="sm" on:click={importRaw}>Import</Button>
		</svelte:fragment>
	</DrawerContent>
</Drawer>

<!-- Raw JSON -->
<Drawer bind:this={rawAppDrawer} size="800px">
	<DrawerContent
		title="Import app in React/Vue/Svelte"
		on:close={() => rawAppDrawer?.toggleDrawer?.()}
	>
		<Path bind:error={pathError} bind:path initialPath="" namePlaceholder={'app'} kind="resource" />
		<h2 class="border-b pb-1 mt-10 mb-4">Summary</h2>

		<input
			type="text"
			bind:value={summary}
			placeholder="Short summary to be displayed when listed"
		/>

		<h2 class="border-b pb-1 mt-10 mb-4"
			>IIFE JS code <Tooltip
				documentationLink="https://www.windmill.dev/docs/react_vue_svelte_apps/react"
				>Bundle that contains an IIFE code that will mount itself to a "root" element. Any framework
				or vanilla JS can be used to create an app and templates are provided for the major
				frameworks: React/Vue/Svelte. In those frontend apps, it is possible to inline scripts
				directly to be executed by windmill backend which makes it a convenient way of building apps
				with both frontend and backend all-in-one.</Tooltip
			></h2
		>
		<FileInput
			accept={'.js'}
			multiple={false}
			convertTo={'text'}
			iconSize={24}
			class="text-sm py-4"
			on:change={({ detail }) => {
				pendingCode = detail?.[0]
			}}
		/>

		<svelte:fragment slot="actions">
			<Button disabled={pathError != ''} size="sm" on:click={importRawApp}>Import</Button>
		</svelte:fragment>
	</DrawerContent>
</Drawer>
