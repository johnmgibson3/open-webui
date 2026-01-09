<script lang="ts">
	import { getAdminConfig, updateAdminConfig } from '$lib/apis/auths';
	import Switch from '$lib/components/common/Switch.svelte';
	import { onMount, getContext } from 'svelte';
	import { toast } from 'svelte-sonner';

	const i18n = getContext('i18n');

	export let saveHandler: Function;

	let loading = false;
	let adminConfig: any = null;

	const updateHandler = async () => {
		if (!adminConfig) {
			toast.error($i18n.t('Configuration not loaded yet'));
			return;
		}

		loading = true;
		const res = await updateAdminConfig(localStorage.token, adminConfig).catch((error) => {
			toast.error(`${error}`);
			loading = false;
			return null;
		});

		if (res) {
			saveHandler();
		} else {
			toast.error($i18n.t('Failed to update settings'));
		}

		loading = false;
	};

	onMount(async () => {
		adminConfig = await getAdminConfig(localStorage.token);

		// Initialize sharing fields if they don't exist
		if (adminConfig) {
			if (adminConfig.ENABLE_INDIVIDUAL_USER_SHARING === undefined) {
				adminConfig.ENABLE_INDIVIDUAL_USER_SHARING = true;
			}
			if (!adminConfig.USER_SHARING_SCOPE) {
				adminConfig.USER_SHARING_SCOPE = 'restricted';
			}
			if (!adminConfig.USER_SHARING_MODE) {
				adminConfig.USER_SHARING_MODE = 'email';
			}
		}
	});
</script>

<form
	class="flex flex-col h-full justify-between space-y-3 text-sm"
	on:submit|preventDefault={async () => {
		await updateHandler();
	}}
>
	<div class=" space-y-3 pr-1.5 overflow-y-scroll max-h-[22rem] scrollbar-hidden">
		<div>
			<div class=" mb-1 text-sm font-medium">{$i18n.t('User Sharing')}</div>

			<div class="flex w-full justify-between">
				<div class=" self-center text-xs font-medium">
					{$i18n.t('Enable Individual User Sharing')}
				</div>

				{#if adminConfig}
					<Switch bind:state={adminConfig.ENABLE_INDIVIDUAL_USER_SHARING} />
				{/if}
			</div>

			<div class=" text-xs text-gray-400">
				{$i18n.t(
					'Allow users to share resources with individual users, not just groups. When disabled, only group-based sharing is available.'
				)}
			</div>
		</div>

		{#if adminConfig && adminConfig.ENABLE_INDIVIDUAL_USER_SHARING}
			<hr class=" dark:border-gray-850" />

			<div>
				<div class=" mb-2 text-sm font-medium">{$i18n.t('Sharing Scope')}</div>

				<div class="flex w-full gap-2">
					<div class="flex-1">
						<label for="sharing-scope-restricted">
							<input
								id="sharing-scope-restricted"
								type="radio"
								bind:group={adminConfig.USER_SHARING_SCOPE}
								value="restricted"
								class="mr-2"
							/>
							<span class="text-xs font-medium">{$i18n.t('Restricted')}</span>
						</label>
						<div class="text-xs text-gray-400 ml-6">
							{$i18n.t('Users can only share with members of groups they belong to')}
						</div>
					</div>
				</div>

				<div class="flex w-full gap-2 mt-2">
					<div class="flex-1">
						<label for="sharing-scope-global">
							<input
								id="sharing-scope-global"
								type="radio"
								bind:group={adminConfig.USER_SHARING_SCOPE}
								value="global"
								class="mr-2"
							/>
							<span class="text-xs font-medium">{$i18n.t('Global')}</span>
						</label>
						<div class="text-xs text-gray-400 ml-6">
							{$i18n.t('Users can share with any user in the system')}
						</div>
					</div>
				</div>
			</div>

			<hr class=" dark:border-gray-850" />

			<div>
				<div class=" mb-2 text-sm font-medium">{$i18n.t('User Selection Mode')}</div>

				<div class="flex w-full gap-2">
					<div class="flex-1">
						<label for="sharing-mode-email">
							<input
								id="sharing-mode-email"
								type="radio"
								bind:group={adminConfig.USER_SHARING_MODE}
								value="email"
								class="mr-2"
							/>
							<span class="text-xs font-medium">{$i18n.t('Email Only')}</span>
						</label>
						<div class="text-xs text-gray-400 ml-6">
							{$i18n.t('Users must enter exact email addresses')}
						</div>
					</div>
				</div>

				<div class="flex w-full gap-2 mt-2">
					<div class="flex-1">
						<label for="sharing-mode-search">
							<input
								id="sharing-mode-search"
								type="radio"
								bind:group={adminConfig.USER_SHARING_MODE}
								value="search"
								class="mr-2"
							/>
							<span class="text-xs font-medium">{$i18n.t('Search Only')}</span>
						</label>
						<div class="text-xs text-gray-400 ml-6">
							{$i18n.t('Users can search for others by name')}
						</div>
					</div>
				</div>

				<div class="flex w-full gap-2 mt-2">
					<div class="flex-1">
						<label for="sharing-mode-both">
							<input
								id="sharing-mode-both"
								type="radio"
								bind:group={adminConfig.USER_SHARING_MODE}
								value="both"
								class="mr-2"
							/>
							<span class="text-xs font-medium">{$i18n.t('Both')}</span>
						</label>
						<div class="text-xs text-gray-400 ml-6">
							{$i18n.t('Users can use either email or search')}
						</div>
					</div>
				</div>
			</div>
		{/if}
	</div>

	<div class="flex justify-end pt-3 text-sm font-medium">
		<button
			class="px-4 py-2 bg-emerald-700 hover:bg-emerald-800 text-gray-50 transition rounded-lg"
			type="submit"
			disabled={loading || !adminConfig}
		>
			{loading ? $i18n.t('Saving...') : $i18n.t('Save')}
		</button>
	</div>
</form>
