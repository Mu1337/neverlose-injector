<script lang="ts">
  import { onMount } from 'svelte';
  import { fade, scale } from 'svelte/transition';

  type Branch = 'Release' | 'Nightly';
  type Game = 'cs2-csgo_legacy' | 'csgo' | 'cs2';
  type View = 'boot' | 'launcher' | 'details' | 'closingDetails' | 'launching';
  type InstalledGames = {
    cs2_legacy_branch: boolean;
    csgo_standalone: boolean;
    cs2_standalone: boolean;
  };
  type ConfigEntry = {
    entry_id: number;
    name: string;
  };
  type LauncherVersion = {
    tag: string;
    name: string;
    changelog: string;
    updated_at: string;
    url: string;
    assets: LauncherAsset[];
  };
  type LauncherAsset = {
    name: string;
    url: string;
    size: number;
  };

  const WEBSITE_URL = 'https://example.com/';
  const DISCORD_URL = 'https://discord.gg/example';
  const API_DOCS_URL = 'https://docs.example.com/';

  let view = $state<View>('boot');
  let game = $state<Game>('csgo');
  let installedStatus = $state<InstalledGames>({
    cs2_legacy_branch: true,
    csgo_standalone: true,
    cs2_standalone: true,
  });
  let branch = $state<Branch>('Release');
  let branchOpen = $state(false);
  let versionOpen = $state(false);
  let configOpen = $state(false);
  let profileOpen = $state(false);
  let profileSaving = $state(false);
  let profileError = $state('');
  let username = $state('User');
  let profileNameInput = $state('');
  let avatarDataUrl = $state('/avatar.png');
  let avatarDataUrlBeforeEdit = $state('');
  let pendingAvatarBytes = $state<number[] | null>(null);
  let avatarInput = $state<HTMLInputElement | null>(null);
  let configs = $state<ConfigEntry[]>([
    { entry_id: 1, name: 'Default Config' },
    { entry_id: 2, name: 'Custom Config' }
  ]);
  let selectedConfigId = $state<number | null>(1);
  let gitMetadata = $state<LauncherGitMetadata>({
    releases: [],
    nightlies: []
  });
  let selectedVersion = $state('');
  let launchPending = $state(false);
  let launchError = $state('');
  let progress = $state(0);
  let themeVariables = $state('');
  let launchTimer: number | undefined;
  let closeTimer: number | undefined;

  const selectedVersionList = $derived(branch === 'Release' ? gitMetadata.releases : gitMetadata.nightlies);
  const selectedVersionData = $derived(
    selectedVersionList.find((version) => version.tag === selectedVersion) ?? selectedVersionList[0]
  );
  const selectedVersionLabel = $derived(selectedVersion || 'No builds');
  const selectedBranchLabel = $derived(`${branch} ${selectedVersionLabel}`);
  const selectedConfigName = $derived(
    configs.find((config) => config.entry_id === selectedConfigId)?.name ?? 'Loading...'
  );
  const versions = $derived(selectedVersionList);
  const updatedAtLabel = $derived(formatGitDate(selectedVersionData?.updated_at));
  const changelogEntries = $derived(parseChangelog(selectedVersionData?.changelog));
  const selectedReleaseUrl = $derived(selectedVersionData?.url ?? '');

  type LauncherSettings = {
    username: string;
    avatar_data_url: string | null;
    selected_config_id: number | null;
    configs: ConfigEntry[];
  };

  type LauncherGitMetadata = {
    releases: LauncherVersion[];
    nightlies: LauncherVersion[];
  };

  onMount(() => {
    const blockContextMenu = (event: MouseEvent) => {
      event.preventDefault();
    };

    const closeFloatingMenus = (event: MouseEvent) => {
      const target = event.target as HTMLElement;

      if (
        target.closest(
          '.branch-trigger, .branch-menu, .metadata-trigger, .metadata-menu, .changelog a, .profile-trigger, .profile-popout'
        )
      ) {
        return;
      }

      closeMenus();
    };

    window.addEventListener('contextmenu', blockContextMenu);
    window.addEventListener('mousedown', closeFloatingMenus, true);
    void loadGitMetadata();

    const bootTimer = window.setTimeout(() => {
      showLauncher();
    }, 1000);

    profileNameInput = username;

    return () => {
      window.clearTimeout(bootTimer);
      window.removeEventListener('contextmenu', blockContextMenu);
      window.removeEventListener('mousedown', closeFloatingMenus, true);
    };
  });

  const appids: Record<Game, number> = {
    csgo: 4465480,
    'cs2-csgo_legacy': 730,
    cs2: 730,
  };

  function showLauncher() {
    view = 'launcher';
  }

  async function loadGitMetadata() {
    try {
      const resp = await fetch('/changelogs.json');
      const data = await resp.json();
      const entry = data[game] ?? { tag: 'v1.0.0', updated_at: new Date().toISOString(), url: '', changelog: 'Demo version' };
      gitMetadata = {
        releases: [{ tag: entry.tag, name: entry.tag, changelog: entry.changelog, updated_at: entry.updated_at, url: entry.url, assets: [] }],
        nightlies: []
      };
      selectedVersion = entry.tag;
    } catch {
      gitMetadata = { releases: [{ tag: 'v1.0.0', name: 'v1.0.0', changelog: 'Demo UI Version', updated_at: '', url: '', assets: [] }], nightlies: [] };
      selectedVersion = 'v1.0.0';
    }
  }

  function openDetails() {
    view = 'details';
    branchOpen = false;
    versionOpen = false;
    configOpen = false;
    launchPending = false;
  }

  function closeDetails() {
    if (launchTimer) {
      window.clearTimeout(launchTimer);
      launchTimer = undefined;
    }

    branchOpen = false;
    versionOpen = false;
    configOpen = false;
    launchPending = false;

    if (view === 'details') {
      view = 'closingDetails';
      closeTimer = window.setTimeout(() => {
        view = 'launcher';
        closeTimer = undefined;
      }, 240);
      return;
    }

    view = 'launcher';
  }

  function toggleBranch(event: MouseEvent) {
    event.stopPropagation();
    versionOpen = false;
    configOpen = false;
    branchOpen = !branchOpen;
  }

  function selectBranch(next: Branch) {
    branch = next;
    selectedVersion = (next === 'Release' ? gitMetadata.releases : gitMetadata.nightlies)[0]?.tag ?? '';
    branchOpen = false;
  }

  function toggleVersion(event: MouseEvent) {
    event.stopPropagation();
    configOpen = false;
    branchOpen = false;
    versionOpen = !versionOpen;
  }

  function selectVersion(version: string) {
    selectedVersion = version;
    versionOpen = false;
  }

  function toggleConfig(event: MouseEvent) {
    event.stopPropagation();
    branchOpen = false;
    versionOpen = false;
    configOpen = !configOpen;
  }

  function selectConfig(config: ConfigEntry) {
    selectedConfigId = config.entry_id;
    configOpen = false;
  }

  function openExternal(url: string) {
    if (!url) {
      return;
    }
    window.open(url, '_blank', 'noreferrer');
  }

  function openProfile(event: MouseEvent) {
    event.stopPropagation();
    profileNameInput = username;
    avatarDataUrlBeforeEdit = avatarDataUrl;
    pendingAvatarBytes = null;
    profileError = '';
    branchOpen = false;
    versionOpen = false;
    configOpen = false;
    profileOpen = !profileOpen;
  }

  function chooseAvatar() {
    avatarInput?.click();
  }

  async function handleAvatarChange(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) {
      return;
    }

    if (!file.type.startsWith('image/')) {
      profileError = 'Choose an image file.';
      input.value = '';
      return;
    }

    try {
      const rounded = await roundedAvatarPng(file);
      pendingAvatarBytes = Array.from(rounded.bytes);
      avatarDataUrl = rounded.dataUrl;
      profileError = '';
    } catch (error) {
      profileError = String(error);
    } finally {
      input.value = '';
    }
  }

  function cancelProfileEdit() {
    profileNameInput = username;
    avatarDataUrl = avatarDataUrlBeforeEdit;
    pendingAvatarBytes = null;
    profileError = '';
  }

  async function roundedAvatarPng(file: File) {
    const source = await fileToDataUrl(file);
    const image = await loadImage(source);
    const size = 256;
    const canvas = document.createElement('canvas');
    canvas.width = size;
    canvas.height = size;
    const context = canvas.getContext('2d');
    if (!context) {
      throw new Error('Could not prepare profile image.');
    }

    const edge = Math.min(image.naturalWidth, image.naturalHeight);
    const sx = (image.naturalWidth - edge) / 2;
    const sy = (image.naturalHeight - edge) / 2;
    context.clearRect(0, 0, size, size);
    context.save();
    context.beginPath();
    context.arc(size / 2, size / 2, size / 2, 0, Math.PI * 2);
    context.clip();
    context.drawImage(image, sx, sy, edge, edge, 0, 0, size, size);
    context.restore();

    const dataUrl = canvas.toDataURL('image/png');
    const response = await fetch(dataUrl);
    const bytes = new Uint8Array(await response.arrayBuffer());
    return { bytes, dataUrl };
  }

  function loadImage(src: string) {
    return new Promise<HTMLImageElement>((resolve, reject) => {
      const image = new Image();
      image.onload = () => resolve(image);
      image.onerror = () => reject(new Error('Could not load profile image.'));
      image.src = src;
    });
  }

  function fileToDataUrl(file: File) {
    return new Promise<string>((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(String(reader.result ?? ''));
      reader.onerror = () => reject(reader.error);
      reader.readAsDataURL(file);
    });
  }

  async function saveProfile() {
    const nextUsername = profileNameInput.trim();
    if (!nextUsername) {
      profileError = 'Enter a profile name.';
      return;
    }

    profileSaving = true;
    profileError = '';
    
    username = nextUsername;
    profileNameInput = username;
    avatarDataUrlBeforeEdit = avatarDataUrl;
    pendingAvatarBytes = null;
    profileSaving = false;
  }

  async function saveProfileName() {
    if (profileSaving || profileNameInput.trim() === username) {
      return;
    }
    await saveProfile();
  }

  function handleProfileNameKeydown(event: KeyboardEvent) {
    if (event.key === 'Enter') {
      event.preventDefault();
      void saveProfileName();
      return;
    }

    if (event.key === 'Escape') {
      event.preventDefault();
      profileNameInput = username;
      profileError = '';
    }
  }

  function launch(appid: number) {
    branchOpen = false;
    launchPending = true;
    launchError = '';
    progress = 0;

    if (closeTimer) {
      window.clearTimeout(closeTimer);
      closeTimer = undefined;
    }

    launchTimer = window.setTimeout(() => {
      if (!launchPending) {
        return;
      }

      view = 'launching';
      launchTimer = undefined;
      void startLaunchProgress();
    }, 230);
  }

  async function startLaunchProgress() {
    const startedAt = performance.now();
    let finished = false;
    const tick = () => {
      const elapsed = performance.now() - startedAt;
      progress = finished ? 100 : Math.min(92, 18 + (elapsed / 3200) * 74);

      if (!finished) {
        requestAnimationFrame(tick);
      }
    };

    requestAnimationFrame(tick);

    // Simulate demo launch
    await new Promise(resolve => setTimeout(resolve, 2000));
    finished = true;
    progress = 100;
    
    window.setTimeout(() => {
      launchPending = false;
      view = 'launcher';
    }, 1000);
  }

  function closeBranchFromBackdrop() {
    closeMenus();
  }

  function closeMenus() {
    branchOpen = false;
    versionOpen = false;
    configOpen = false;
    if (profileOpen && pendingAvatarBytes) {
      avatarDataUrl = avatarDataUrlBeforeEdit;
      pendingAvatarBytes = null;
    }
    profileOpen = false;
  }

  function formatGitDate(value: string | undefined) {
    if (!value) {
      return 'Unknown';
    }

    const date = new Date(value);
    if (Number.isNaN(date.getTime())) {
      return value;
    }

    const day = date.getDate().toString().padStart(2, '0');
    const month = (date.getMonth() + 1).toString().padStart(2, '0');
    const year = date.getFullYear();
    const hour = date.getHours().toString().padStart(2, '0');
    const minute = date.getMinutes().toString().padStart(2, '0');

    return `${day}.${month}.${year} ${hour}:${minute}`;
  }

  function parseChangelog(value: string | undefined) {
    const trimmed = value?.trim();

    if (!trimmed) {
      return ['No changelog provided.'];
    }

    return trimmed
      .split(/\r?\n/)
      .map((line) => line.trim())
      .filter(Boolean)
      .map((line) => line.replace(/^[-*]\s*/, '- '));
  }

  function renderMarkdownLine(value: string) {
    return escapeHtml(value).replace(
      /\[([^\]]+)\]\((https?:\/\/[^)\s]+)\)/g,
      '<a href="$2" target="_blank" rel="noreferrer">$1</a>'
    );
  }

  function escapeHtml(value: string) {
    return value
      .replaceAll('&', '&amp;')
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;')
      .replaceAll("'", '&#39;');
  }
</script>

{#snippet IconChevron()}
  <svg viewBox="0 0 24 24" aria-hidden="true">
    <path d="M7 10l5 5 5-5" />
  </svg>
{/snippet}

{#snippet IconPlay()}
  <svg viewBox="0 0 24 24" aria-hidden="true">
    <path d="M8 5v14l11-7z" />
  </svg>
{/snippet}

{#snippet IconClose()}
  <svg viewBox="0 0 24 24" aria-hidden="true">
    <path d="M7 7l10 10M17 7L7 17" />
  </svg>
{/snippet}

{#snippet IconCog()}
  <svg viewBox="0 0 24 24" aria-hidden="true">
    <path d="M12 8.25a3.75 3.75 0 1 0 0 7.5 3.75 3.75 0 0 0 0-7.5Z" />
    <path d="M19.43 12.98c.04-.32.07-.65.07-.98s-.02-.66-.07-.98l2.03-1.58-1.92-3.32-2.39.96a7.48 7.48 0 0 0-1.7-.98L15.1 3.5h-3.84l-.36 2.6c-.6.24-1.17.56-1.7.98l-2.39-.96-1.92 3.32 2.03 1.58c-.04.32-.07.65-.07.98s.02.66.07.98l-2.03 1.58 1.92 3.32 2.39-.96a7.48 7.48 0 0 0 1.7.98l.36 2.6h3.84l.36-2.6c.6-.24 1.17-.56 1.7-.98l2.39.96 1.92-3.32-2.03-1.58z" />
  </svg>
{/snippet}

<svelte:head>
  <title>Clean UI Launcher</title>
</svelte:head>

{#if branchOpen || versionOpen || configOpen || profileOpen}
  <button class="click-away" aria-label="Close menu" onclick={closeMenus}></button>
{/if}

<main
  class:boot={view === 'boot'}
  class:expanded={view !== 'boot'}
  class="shell"
  style={themeVariables}
>
  {#if view === 'boot'}
    <section class="boot-view" aria-label="Loading">
      <div class="boot-spinner">
        <svg viewBox="0 0 48 48" aria-hidden="true">
          <circle cx="24" cy="24" r="17"></circle>
        </svg>
      </div>
    </section>
  {:else}
    <section class="launcher-view" aria-label="Launcher">
      <div class="logo">NL</div>

      <nav class="side-nav" aria-label="Navigation">
        <button onclick={() => openExternal(WEBSITE_URL)}>Website</button>
        <button onclick={() => openExternal(DISCORD_URL)}>Support</button>
        <button>Market</button>
      </nav>

      <button class:active={profileOpen} class="profile profile-trigger" data-no-drag onclick={openProfile}>
        <span class="avatar">
          {#if avatarDataUrl}
            <img src={avatarDataUrl} alt="" draggable="false" />
          {/if}
        </span>
        <span>{username}</span>
      </button>

      {#if profileOpen}
        <button
          class="background-dim visible profile-dim"
          aria-label="Close profile settings"
          onclick={closeMenus}
          transition:fade={{ duration: 210 }}
        ></button>
        <section
          class="profile-popout"
          data-no-drag
          aria-label="Profile settings"
          transition:scale={{ duration: 220, start: 0.985, opacity: 0 }}
        >
          <button aria-label="Close profile settings" class="detail-close profile-close" onclick={closeMenus}>
            {@render IconClose()}
          </button>

          <div class="profile-content">
            <button class="avatar profile-popout-avatar" aria-label="Change profile image" onclick={chooseAvatar}>
              {#if avatarDataUrl}
                <img src={avatarDataUrl} alt="" draggable="false" />
              {/if}
              <span>Change</span>
            </button>
            <input
              bind:this={avatarInput}
              class="avatar-file"
              type="file"
              accept="image/png,image/jpeg,image/gif,image/webp"
              onchange={handleAvatarChange}
            />

            <div class="profile-name-card">
              <input
                bind:value={profileNameInput}
                maxlength="32"
                spellcheck="false"
                aria-label="Profile name"
                onblur={saveProfileName}
                onkeydown={handleProfileNameKeydown}
              />
            </div>

            {#if profileError}
              <p class="profile-error">{profileError}</p>
            {/if}

            {#if pendingAvatarBytes}
              <div class="profile-actions">
                <button onclick={cancelProfileEdit} disabled={profileSaving}>Cancel</button>
                <button class="save-profile" onclick={saveProfile} disabled={profileSaving}>
                  {profileSaving ? 'Saving...' : 'Save'}
                </button>
              </div>
            {/if}

            <div class="profile-wip">
              <span>Demo Version</span>
            </div>
          </div>
        </section>
      {/if}

      <section class="subscriptions" aria-label="Subscriptions">
        <h1>Subscription</h1>
        <p>Available subscriptions</p>

        <button
          class="subscription-card"
          class:active={installedStatus.cs2_standalone}
          class:disabled={!installedStatus.cs2_standalone}
          disabled={!installedStatus.cs2_standalone}
          onclick={() => (game = 'cs2') && openDetails()}
        >
          <span>
            <strong>Counter-Strike 2</strong>
            {#if installedStatus.cs2_standalone}
              <em>Demo - Available</em>
            {:else}
              <em class="not-installed-label">⚠️ Not Installed</em>
            {/if}
          </span>
          <img class="game-icon" src="/cs2.png" alt="" draggable="false" />
        </button>

        <button
          class="subscription-card"
          class:active={installedStatus.cs2_legacy_branch}
          class:disabled={!installedStatus.cs2_legacy_branch}
          disabled={!installedStatus.cs2_legacy_branch}
          onclick={() => (game = 'cs2-csgo_legacy') && openDetails()}
        >
          <span>
            <strong>Counter-Strike: Global Offensive</strong>
            {#if installedStatus.cs2_legacy_branch}
              <em>Demo - Available</em>
            {:else}
              <em class="not-installed-label">⚠️ Not Installed</em>
            {/if}
          </span>
          <img class="game-icon" src="/csgo.png" alt="" draggable="false" />
        </button>

        <button
          class="subscription-card"
          class:active={installedStatus.csgo_standalone}
          class:disabled={!installedStatus.csgo_standalone}
          disabled={!installedStatus.csgo_standalone}
          onclick={() => (game = 'csgo') && openDetails()}
        >
          <span>
            <strong>Apex Legends</strong>
            {#if installedStatus.csgo_standalone}
              <em>Demo - Available</em>
            {:else}
              <em class="not-installed-label">⚠️ Not Installed</em>
            {/if}
          </span>
          <img class="game-icon" src="/apex.png" alt="" draggable="false" />
        </button>

      </section>

      {#if view === 'details' || view === 'closingDetails' || view === 'launching'}
        <div class:visible={view !== 'closingDetails'} class="background-dim"></div>

        <section
          class:closing={view === 'closingDetails'}
          class:fading={launchPending || view === 'closingDetails' || view === 'launching'}
          class:launching={view === 'launching'}
          class="details"
          aria-label="Subscription details"
        >
          <div class="detail-content">
            <header>
              <img class="game-icon large" src={game === 'cs2-csgo_legacy' ? '/csgo.png' : game === 'csgo' ? '/apex.png' : '/cs2.png'} alt="" draggable="false" />
              {#if game === 'cs2-csgo_legacy'}
                <h2>Counter-Strike: Global Offensive</h2>
              {:else if game === 'csgo'}
                <h2>Apex Legends</h2>
              {:else}
                <h2>Counter-Strike 2</h2>
              {/if}
              <button aria-label="Close details" class="detail-close" onclick={closeDetails}>{@render IconClose()}</button>
            </header>

            <div class="detail-body">
              {#if launchError}
                <div class="launch-error" transition:fade={{ duration: 150 }}>
                  <button aria-label="Dismiss error" class="launch-error-close" onclick={() => launchError = ''}>
                    {@render IconClose()}
                  </button>
                  <div class="launch-error-title">Launch Failed</div>
                  <div class="launch-error-message">{launchError}</div>
                </div>
              {:else}
                <div class="metadata">
                  <div class="metadata-row with-menu">
                    <span class="label">Branch:</span>
                    <button
                      class:active={versionOpen}
                      class="metadata-trigger version-trigger"
                      onclick={toggleVersion}
                    >
                      <span class="trigger-icon branch-icon-small"></span>
                      <span>{selectedBranchLabel}</span>
                    </button>
                    {#if versionOpen}
                      <div class="metadata-menu version-menu">
                        {#each versions as version}
                          <button class:selected={version.tag === selectedVersion} onclick={() => selectVersion(version.tag)}>
                            {version.tag}
                          </button>
                        {:else}
                          <button disabled>No builds</button>
                        {/each}
                      </div>
                    {/if}
                  </div>
                  <p><span class="label">Updated:</span> <span class="value">{updatedAtLabel}</span></p>
                  <div class="metadata-row with-menu">
                    <span class="label">Config:</span>
                    <button
                      class:active={configOpen}
                      class="metadata-trigger config-trigger"
                      onclick={toggleConfig}
                    >
                      <span class="trigger-icon cog-icon-small">{@render IconCog()}</span>
                      <span>{selectedConfigName}</span>
                    </button>
                    {#if configOpen}
                      <div class="metadata-menu config-menu">
                        {#each configs as config}
                          <button class:selected={config.entry_id === selectedConfigId} onclick={() => selectConfig(config)}>
                            {config.name}
                          </button>
                        {/each}
                      </div>
                    {/if}
                  </div>
                  <p><span class="label">Last Launch:</span> <span class="value">Just Now</span></p>
                </div>

                <div class="changelog">
                  <p class="date">
                    Changelogs
                    {#if selectedReleaseUrl}
                      <a href={selectedReleaseUrl} target="_blank" rel="noreferrer">{selectedVersionLabel}</a>
                    {:else}
                      <span>{selectedVersionLabel}</span>
                    {/if}
                  </p>
                  <p>
                    {#each changelogEntries as entry}
                      {@html renderMarkdownLine(entry)}<br />
                    {/each}
                  </p>
                </div>
              {/if}
            </div>

            <footer>
              <div class="footer-links">
                <button onclick={() => openExternal(DISCORD_URL)}>Community</button>
                <button onclick={() => openExternal(API_DOCS_URL)}>API Documentation</button>
              </div>

              <div class="actions">
                <button
                  disabled={launchPending}
                  class:active={branchOpen}
                  class="branch-trigger"
                  aria-label="Select branch"
                  onclick={toggleBranch}
                >
                  {@render IconChevron()}
                </button>
                <button disabled={branchOpen || launchPending} class="load" onclick={() => launch(appids[game])}>
                  <span class="play-icon">{@render IconPlay()}</span>
                  Load (Demo)
                </button>
              </div>
            </footer>
          </div>
        </section>
      {/if}
    </section>
  {/if}
</main>
