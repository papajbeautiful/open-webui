<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { flyAndScale } from '$lib/utils/transitions';
	import { getContext, onMount, tick } from 'svelte';

	import { config, user, tools as _tools, mobile } from '$lib/stores';
	import { createPicker } from '$lib/utils/google-drive-picker';

	import { getTools } from '$lib/apis/tools';

	import Dropdown from '$lib/components/common/Dropdown.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import DocumentArrowUpSolid from '$lib/components/icons/DocumentArrowUpSolid.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import GlobeAltSolid from '$lib/components/icons/GlobeAltSolid.svelte';
	import WrenchSolid from '$lib/components/icons/WrenchSolid.svelte';
	import CameraSolid from '$lib/components/icons/CameraSolid.svelte';
	import PhotoSolid from '$lib/components/icons/PhotoSolid.svelte';
	import CommandLineSolid from '$lib/components/icons/CommandLineSolid.svelte';

	const i18n = getContext('i18n');

	export let selectedToolIds: string[] = [];

	export let selectedModels: string[] = [];
	export let fileUploadCapableModels: string[] = [];

	export let screenCaptureHandler: Function;
	export let uploadFilesHandler: Function;
	export let inputFilesHandler: Function;

	export let uploadGoogleDriveHandler: Function;
	export let uploadOneDriveHandler: Function;

	export let onClose: Function;

	let tools = {};
	let show = false;
	let showAllTools = false;

	$: if (show) {
		init();
	}

	let fileUploadEnabled = true;
	$: fileUploadEnabled =
		fileUploadCapableModels.length === selectedModels.length &&
		($user?.role === 'admin' || $user?.permissions?.chat?.file_upload);

	const init = async () => {
		if ($_tools === null) {
			await _tools.set(await getTools(localStorage.token));
		}

		tools = $_tools.reduce((a, tool, i, arr) => {
			a[tool.id] = {
				name: tool.name,
				description: tool.meta.description,
				enabled: selectedToolIds.includes(tool.id)
			};
			return a;
		}, {});
	};

	const detectMobile = () => {
		const userAgent = navigator.userAgent || navigator.vendor || window.opera;
		return /android|iphone|ipad|ipod|windows phone/i.test(userAgent);
	};

	function handleFileChange(event) {
		const inputFiles = Array.from(event.target?.files);
		if (inputFiles && inputFiles.length > 0) {
			console.log(inputFiles);
			inputFilesHandler(inputFiles);
		}
	}
</script>

<!-- Hidden file input used to open the camera on mobile -->
<input
	id="camera-input"
	type="file"
	accept="image/*"
	capture="environment"
	on:change={handleFileChange}
	style="display: none;"
/>

<Dropdown
	bind:show
	on:change={(e) => {
		if (e.detail === false) {
			onClose();
		}
	}}
>
	<Tooltip content={$i18n.t('More')}>
		<slot />
	</Tooltip>

	<div slot="content">
		<DropdownMenu.Content
			class="w-full max-w-[200px] rounded-xl px-1 py-1 border border-gray-300/30 dark:border-gray-700/50 z-50 bg-white dark:bg-gray-850 dark:text-white shadow-sm"
			sideOffset={10}
			alignOffset={-8}
			side="top"
			align="start"
			transition={flyAndScale}
		>
			{#if Object.keys(tools).length > 0}
				<div class="{showAllTools ? '' : 'max-h-28'} overflow-y-auto scrollbar-thin">
					{#each Object.keys(tools) as toolId}
						<button
							class="flex w-full justify-between gap-2 items-center px-3 py-2 text-sm font-medium cursor-pointer rounded-xl"
							on:click={() => {
								tools[toolId].enabled = !tools[toolId].enabled;
							}}
						>
							<div class="flex-1 truncate">
								<Tooltip
									content={tools[toolId]?.description ?? ''}
									placement="top-start"
									className="flex flex-1 gap-2 items-center"
								>
									<div class="shrink-0">
										<WrenchSolid />
									</div>

									<div class=" truncate">{tools[toolId].name}</div>
								</Tooltip>
							</div>

							<div class=" shrink-0">
								<Switch
									state={tools[toolId].enabled}
									on:change={async (e) => {
										const state = e.detail;
										await tick();
										if (state) {
											selectedToolIds = [...selectedToolIds, toolId];
										} else {
											selectedToolIds = selectedToolIds.filter((id) => id !== toolId);
										}
									}}
								/>
							</div>
						</button>
					{/each}
				</div>
				{#if Object.keys(tools).length > 3}
					<button
						class="flex w-full justify-center items-center text-sm font-medium cursor-pointer rounded-lg hover:bg-gray-50 dark:hover:bg-gray-800"
						on:click={() => {
							showAllTools = !showAllTools;
						}}
						title={showAllTools ? $i18n.t('Show Less') : $i18n.t('Show All')}
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							fill="none"
							viewBox="0 0 24 24"
							stroke-width="2.5"
							stroke="currentColor"
							class="size-3 transition-transform duration-200 {showAllTools
								? 'rotate-180'
								: ''} text-gray-300 dark:text-gray-600"
						>
							<path stroke-linecap="round" stroke-linejoin="round" d="m19.5 8.25-7.5 7.5-7.5-7.5"
							></path>
						</svg>
					</button>
				{/if}
				<hr class="border-black/5 dark:border-white/5 my-1" />
			{/if}

			<Tooltip
				content={fileUploadCapableModels.length !== selectedModels.length
					? $i18n.t('Model(s) do not support file upload')
					: !fileUploadEnabled
						? $i18n.t('You do not have permission to upload files.')
						: ''}
				className="w-full"
			>
				<DropdownMenu.Item
					class="flex gap-2 items-center px-3 py-2 text-sm  font-medium cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800  rounded-xl {!fileUploadEnabled
						? 'opacity-50'
						: ''}"
					on:click={() => {
						if (fileUploadEnabled) {
							if (!detectMobile()) {
								screenCaptureHandler();
							} else {
								const cameraInputElement = document.getElementById('camera-input');

								if (cameraInputElement) {
									cameraInputElement.click();
								}
							}
						}
					}}
				>
					<CameraSolid />
					<div class=" line-clamp-1">{$i18n.t('Capture')}</div>
				</DropdownMenu.Item>
			</Tooltip>

			<Tooltip
				content={fileUploadCapableModels.length !== selectedModels.length
					? $i18n.t('Model(s) do not support file upload')
					: !fileUploadEnabled
						? $i18n.t('You do not have permission to upload files.')
						: ''}
				className="w-full"
			>
				<DropdownMenu.Item
					class="flex gap-2 items-center px-3 py-2 text-sm font-medium cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl {!fileUploadEnabled
						? 'opacity-50'
						: ''}"
					on:click={() => {
						if (fileUploadEnabled) {
							uploadFilesHandler();
						}
					}}
				>
					<DocumentArrowUpSolid />
					<div class="line-clamp-1">{$i18n.t('Upload Files')}</div>
				</DropdownMenu.Item>
			</Tooltip>

			{#if fileUploadEnabled}
				{#if $config?.features?.enable_google_drive_integration}
					<DropdownMenu.Item
						class="flex gap-2 items-center px-3 py-2 text-sm font-medium cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
						on:click={() => {
							uploadGoogleDriveHandler();
						}}
					>
						<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 87.3 78" class="w-5 h-5">
							<path
								d="m6.6 66.85 3.85 6.65c.8 1.4 1.95 2.5 3.3 3.3l13.75-23.8h-27.5c0 1.55.4 3.1 1.2 4.5z"
								fill="#0066da"
							/>
							<path
								d="m43.65 25-13.75-23.8c-1.35.8-2.5 1.9-3.3 3.3l-25.4 44a9.06 9.06 0 0 0 -1.2 4.5h27.5z"
								fill="#00ac47"
							/>
							<path
								d="m73.55 76.8c1.35-.8 2.5-1.9 3.3-3.3l1.6-2.75 7.65-13.25c.8-1.4 1.2-2.95 1.2-4.5h-27.502l5.852 11.5z"
								fill="#ea4335"
							/>
							<path
								d="m43.65 25 13.75-23.8c-1.35-.8-2.9-1.2-4.5-1.2h-18.5c-1.6 0-3.15.45-4.5 1.2z"
								fill="#00832d"
							/>
							<path
								d="m59.8 53h-32.3l-13.75 23.8c1.35.8 2.9 1.2 4.5 1.2h50.8c1.6 0 3.15-.45 4.5-1.2z"
								fill="#2684fc"
							/>
							<path
								d="m73.4 26.5-12.7-22c-.8-1.4-1.95-2.5-3.3-3.3l-13.75 23.8 16.15 28h27.45c0-1.55-.4-3.1-1.2-4.5z"
								fill="#ffba00"
							/>
						</svg>
						<div class="line-clamp-1">{$i18n.t('Google Drive')}</div>
					</DropdownMenu.Item>
				{/if}

				{#if $config?.features?.enable_onedrive_integration}
					<DropdownMenu.Item
						class="flex gap-2 items-center px-3 py-2 text-sm font-medium cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl w-full"
						on:click={() => {
							uploadOneDriveHandler('organizations');
						}}
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 32 32"
							class="w-5 h-5"
							fill="none"
						>
							<!-- Background circles -->
							<circle cx="22" cy="10" r="8" fill="#0F7B82"/>
							<circle cx="20" cy="18" r="7" fill="#16A085"/>
							<circle cx="12" cy="20" r="6" fill="#1ABC9C"/>
							<circle cx="18" cy="24" r="5" fill="#48C9B0"/>
							
							<!-- Main SharePoint square with S -->
							<rect x="6" y="8" width="14" height="14" rx="2" fill="#0F7B82"/>
							<text x="13" y="18" text-anchor="middle" fill="white" font-family="Arial, sans-serif" font-size="10" font-weight="bold">S</text>
						</svg>
						<div class="line-clamp-1">{$i18n.t('SharePoint')}</div>
					</DropdownMenu.Item>
				{/if}
			{/if}
		</DropdownMenu.Content>	
	</div>
</Dropdown>
