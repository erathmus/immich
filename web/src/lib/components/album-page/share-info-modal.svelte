<script lang="ts">
  import {
    getMyUser,
    removeUserFromAlbum,
    type AlbumResponseDto,
    type UserResponseDto,
    updateAlbumUser,
    AlbumUserRole,
  } from '@immich/sdk';
  import { mdiDotsVertical } from '@mdi/js';
  import { createEventDispatcher, onMount } from 'svelte';
  import { handleError } from '../../utils/handle-error';
  import ConfirmDialog from '../shared-components/dialog/confirm-dialog.svelte';
  import MenuOption from '../shared-components/context-menu/menu-option.svelte';
  import { NotificationType, notificationController } from '../shared-components/notification/notification';
  import UserAvatar from '../shared-components/user-avatar.svelte';
  import FullScreenModal from '$lib/components/shared-components/full-screen-modal.svelte';
  import { t } from 'svelte-i18n';
  import ButtonContextMenu from '$lib/components/shared-components/context-menu/button-context-menu.svelte';

  export let album: AlbumResponseDto;
  export let onClose: () => void;

  const dispatch = createEventDispatcher<{
    remove: string;
    refreshAlbum: void;
  }>();

  let currentUser: UserResponseDto;
  let selectedRemoveUser: UserResponseDto | null = null;

  $: isOwned = currentUser?.id == album.ownerId;

  onMount(async () => {
    try {
      currentUser = await getMyUser();
    } catch (error) {
      handleError(error, $t('errors.unable_to_refresh_user'));
    }
  });

  const handleMenuRemove = (user: UserResponseDto) => {
    selectedRemoveUser = user;
  };

  const handleRemoveUser = async () => {
    if (!selectedRemoveUser) {
      return;
    }

    const userId = selectedRemoveUser.id === currentUser?.id ? 'me' : selectedRemoveUser.id;

    try {
      await removeUserFromAlbum({ id: album.id, userId });
      dispatch('remove', userId);
      const message = userId === 'me' ? `Left ${album.albumName}` : `Removed ${selectedRemoveUser.name}`;
      notificationController.show({ type: NotificationType.Info, message });
    } catch (error) {
      handleError(error, $t('errors.unable_to_remove_album_users'));
    } finally {
      selectedRemoveUser = null;
    }
  };

  const handleSetReadonly = async (user: UserResponseDto, role: AlbumUserRole) => {
    try {
      await updateAlbumUser({ id: album.id, userId: user.id, updateAlbumUserDto: { role } });
      const message = `Set ${user.name} as ${role}`;
      dispatch('refreshAlbum');
      notificationController.show({ type: NotificationType.Info, message });
    } catch (error) {
      handleError(error, $t('errors.unable_to_change_album_user_role'));
    } finally {
      selectedRemoveUser = null;
    }
  };
</script>

{#if !selectedRemoveUser}
  <FullScreenModal title={$t('options')} {onClose}>
    <section class="immich-scrollbar max-h-[400px] overflow-y-auto pb-4">
      <div class="flex w-full place-items-center justify-between gap-4 p-5">
        <div class="flex place-items-center gap-4">
          <UserAvatar user={album.owner} size="md" />
          <p class="text-sm font-medium">{album.owner.name}</p>
        </div>

        <div id="icon-{album.owner.id}" class="flex place-items-center">
          <p class="text-sm">{$t('owner')}</p>
        </div>
      </div>
      {#each album.albumUsers as { user, role }}
        <div
          class="flex w-full place-items-center justify-between gap-4 p-5 rounded-xl transition-colors hover:bg-gray-50 dark:hover:bg-gray-700"
        >
          <div class="flex place-items-center gap-4">
            <UserAvatar {user} size="md" />
            <p class="text-sm font-medium">{user.name}</p>
          </div>

          <div id="icon-{user.id}" class="flex place-items-center gap-2 text-sm">
            <div>
              {#if role === AlbumUserRole.Viewer}
                Viewer
              {:else}
                Editor
              {/if}
            </div>
            {#if isOwned}
              <ButtonContextMenu icon={mdiDotsVertical} size="20" title={$t('options')}>
                {#if role === AlbumUserRole.Viewer}
                  <MenuOption onClick={() => handleSetReadonly(user, AlbumUserRole.Editor)} text={$t('allow_edits')} />
                {:else}
                  <MenuOption
                    onClick={() => handleSetReadonly(user, AlbumUserRole.Viewer)}
                    text={$t('disallow_edits')}
                  />
                {/if}
                <MenuOption onClick={() => handleMenuRemove(user)} text={$t('remove')} />
              </ButtonContextMenu>
            {:else if user.id == currentUser?.id}
              <button
                type="button"
                on:click={() => (selectedRemoveUser = user)}
                class="text-sm font-medium text-immich-primary transition-colors hover:text-immich-primary/75 dark:text-immich-dark-primary"
                >{$t('leave')}</button
              >
            {/if}
          </div>
        </div>
      {/each}
    </section>
  </FullScreenModal>
{/if}

{#if selectedRemoveUser && selectedRemoveUser?.id === currentUser?.id}
  <ConfirmDialog
    title="Leave album?"
    prompt="Are you sure you want to leave {album.albumName}?"
    confirmText={$t('leave')}
    onConfirm={handleRemoveUser}
    onCancel={() => (selectedRemoveUser = null)}
  />
{/if}

{#if selectedRemoveUser && selectedRemoveUser?.id !== currentUser?.id}
  <ConfirmDialog
    title="Remove user?"
    prompt="Are you sure you want to remove {selectedRemoveUser.name}?"
    confirmText={$t('remove')}
    onConfirm={handleRemoveUser}
    onCancel={() => (selectedRemoveUser = null)}
  />
{/if}
