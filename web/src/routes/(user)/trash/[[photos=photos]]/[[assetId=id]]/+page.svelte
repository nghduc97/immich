<script lang="ts">
  import { goto } from '$app/navigation';
  import empty3Url from '$lib/assets/empty-3.svg';
  import LinkButton from '$lib/components/elements/buttons/link-button.svelte';
  import Icon from '$lib/components/elements/icon.svelte';
  import UserPageLayout from '$lib/components/layouts/user-page-layout.svelte';
  import DeleteAssets from '$lib/components/photos-page/actions/delete-assets.svelte';
  import RestoreAssets from '$lib/components/photos-page/actions/restore-assets.svelte';
  import SelectAllAssets from '$lib/components/photos-page/actions/select-all-assets.svelte';
  import AssetGrid from '$lib/components/photos-page/asset-grid.svelte';
  import AssetSelectControlBar from '$lib/components/photos-page/asset-select-control-bar.svelte';
  import ConfirmDialogue from '$lib/components/shared-components/confirm-dialogue.svelte';
  import EmptyPlaceholder from '$lib/components/shared-components/empty-placeholder.svelte';
  import {
    NotificationType,
    notificationController,
  } from '$lib/components/shared-components/notification/notification';
  import { AppRoute } from '$lib/constants';
  import { createAssetInteractionStore } from '$lib/stores/asset-interaction.store';
  import { AssetStore } from '$lib/stores/assets.store';
  import { featureFlags, serverConfig } from '$lib/stores/server-config.store';
  import { handleError } from '$lib/utils/handle-error';
  import { emptyTrash, restoreTrash } from '@immich/sdk';
  import { mdiDeleteOutline, mdiHistory } from '@mdi/js';
  import type { PageData } from './$types';
  import { handlePromiseError } from '$lib/utils';

  export let data: PageData;

  $featureFlags.trash || handlePromiseError(goto(AppRoute.PHOTOS));

  const assetStore = new AssetStore({ isTrashed: true });
  const assetInteractionStore = createAssetInteractionStore();
  const { isMultiSelectState, selectedAssets } = assetInteractionStore;
  let isShowEmptyConfirmation = false;

  const handleEmptyTrash = async () => {
    isShowEmptyConfirmation = false;
    try {
      await emptyTrash();

      const deletedAssetIds = assetStore.assets.map((a) => a.id);
      const numberOfAssets = deletedAssetIds.length;
      assetStore.removeAssets(deletedAssetIds);

      notificationController.show({
        message: `Permanently deleted ${numberOfAssets} ${numberOfAssets == 1 ? 'asset' : 'assets'}`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Error emptying trash');
    }
  };

  const handleRestoreTrash = async () => {
    try {
      await restoreTrash();

      const restoredAssetIds = assetStore.assets.map((a) => a.id);
      const numberOfAssets = restoredAssetIds.length;
      assetStore.removeAssets(restoredAssetIds);

      notificationController.show({
        message: `Restored ${numberOfAssets} ${numberOfAssets == 1 ? 'asset' : 'assets'}`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Error restoring trash');
    }
  };
</script>

{#if $isMultiSelectState}
  <AssetSelectControlBar assets={$selectedAssets} clearSelect={() => assetInteractionStore.clearMultiselect()}>
    <SelectAllAssets {assetStore} {assetInteractionStore} />
    <DeleteAssets force onAssetDelete={(assetIds) => assetStore.removeAssets(assetIds)} />
    <RestoreAssets onRestore={(assetIds) => assetStore.removeAssets(assetIds)} />
  </AssetSelectControlBar>
{/if}

{#if $featureFlags.loaded && $featureFlags.trash}
  <UserPageLayout hideNavbar={$isMultiSelectState} title={data.meta.title} scrollbar={false}>
    <div class="flex place-items-center gap-2" slot="buttons">
      <LinkButton on:click={handleRestoreTrash}>
        <div class="flex place-items-center gap-2 text-sm">
          <Icon path={mdiHistory} size="18" />
          Restore all
        </div>
      </LinkButton>
      <LinkButton on:click={() => (isShowEmptyConfirmation = true)}>
        <div class="flex place-items-center gap-2 text-sm">
          <Icon path={mdiDeleteOutline} size="18" />
          Empty trash
        </div>
      </LinkButton>
    </div>

    <AssetGrid {assetStore} {assetInteractionStore}>
      <p class="font-medium text-gray-500/60 dark:text-gray-300/60 p-4">
        Trashed items will be permanently deleted after {$serverConfig.trashDays} days.
      </p>
      <EmptyPlaceholder text="Trashed photos and videos will show up here." src={empty3Url} slot="empty" />
    </AssetGrid>
  </UserPageLayout>
{/if}

{#if isShowEmptyConfirmation}
  <ConfirmDialogue
    id="empty-trash-modal"
    title="Empty trash"
    confirmText="Empty"
    onConfirm={handleEmptyTrash}
    onClose={() => (isShowEmptyConfirmation = false)}
  >
    <svelte:fragment slot="prompt">
      <p>Are you sure you want to empty the trash? This will remove all the assets in trash permanently from Immich.</p>
      <p><b>You cannot undo this action!</b></p>
    </svelte:fragment>
  </ConfirmDialogue>
{/if}
