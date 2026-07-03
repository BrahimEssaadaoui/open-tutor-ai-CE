<!-- Dashboard Étudiant — Performance & Productivité -->
<script lang="ts">
	import { getContext, onMount, onDestroy } from 'svelte';
	import type { Writable } from 'svelte/store';
	import type { i18n as i18nType } from 'i18next';
	import { goto } from '$app/navigation';
	import { browser } from '$app/environment';
	import { chatId as storeChatId, isDemo, demoData, user } from '$lib/stores';
	import { dashboardTab } from '$lib/stores/focusTimer';
	import { getSupportRequests, type SupportResponse, updateSupportChatId } from '$lib/apis/supports';
	import { page } from '$app/stores';
	import { fade, scale } from 'svelte/transition';
	import { toast } from 'svelte-sonner';

	import PerformanceTab from '../dashboard/PerformanceTab.svelte';
	import ProductivityTab from '../dashboard/ProductivityTab.svelte';

	const i18n = getContext<Writable<i18nType>>('i18n');

	// ── Greeting ───────────────────────────────────────────────────────────────
	$: firstName = $user?.name?.split(' ')[0] ?? '';

	function formatDateFR(): string {
		const now = new Date();
		const days = ['Dimanche', 'Lundi', 'Mardi', 'Mercredi', 'Jeudi', 'Vendredi', 'Samedi'];
		const months = [
			'janvier', 'février', 'mars', 'avril', 'mai', 'juin',
			'juillet', 'août', 'septembre', 'octobre', 'novembre', 'décembre'
		];
		return `${days[now.getDay()]} ${now.getDate()} ${months[now.getMonth()]} ${now.getFullYear()}`;
	}

	// ── Support request management (existing logic preserved) ─────────────────
	let userSupports: SupportResponse[] = [];
	let isLoading = true;

	$: displaySupports = $isDemo
		? $demoData.supports.map((s) => ({
				id: s.id,
				title: s.title,
				description: s.description,
				status: s.progress < 30 ? 'not-started' : s.progress < 100 ? 'in-progress' : 'completed',
				category: s.category,
				difficulty: s.difficulty,
				progress: s.progress
			}))
		: userSupports;

	let pendingSupportId = '';
	let chatIdSubscription: Function;
	let urlCheckInterval: ReturnType<typeof setInterval>;
	let currentPath = '';
	let chatIdFromURL = '';

	onMount(async () => {
		if (browser) {
			storeChatId.set('');
			sessionStorage.removeItem('selectedModels');

			if (localStorage.getItem('pendingSupportData')) {
				localStorage.removeItem('pendingSupportData');
			}
			const keysToRemove: string[] = [];
			for (let i = 0; i < localStorage.length; i++) {
				const key = localStorage.key(i);
				if (key && key.startsWith('chat-input-')) keysToRemove.push(key);
			}
			keysToRemove.forEach((k) => localStorage.removeItem(k));

			if ($isDemo) {
				isLoading = false;
			} else {
				const token = localStorage.getItem('token');
				if (token) {
					try {
						const supports = await getSupportRequests(token);
						if (supports && Array.isArray(supports)) userSupports = supports;
					} catch {
						userSupports = [];
					} finally {
						isLoading = false;
					}
				} else {
					isLoading = false;
				}
			}

			if (!window.openTutorEvents) window.openTutorEvents = new EventTarget();
			window.openTutorEvents.addEventListener('chatCreated', ((event: CustomEvent) => {
				const newChatId = event.detail?.chatId;
				if (newChatId && pendingSupportId) updateSupportWithChatId(pendingSupportId, newChatId);
			}) as EventListener);

			chatIdSubscription = storeChatId.subscribe((newChatId) => {
				if (newChatId && newChatId !== 'local' && pendingSupportId)
					updateSupportWithChatId(pendingSupportId, newChatId);
			});

			urlCheckInterval = setInterval(() => {
				try {
					const pendingSupportData = localStorage.getItem('pendingSupportData');
					if (!pendingSupportData) { clearInterval(urlCheckInterval); return; }
					const supportData = JSON.parse(pendingSupportData);
					if (Date.now() - (supportData.timestamp || 0) >= 30 * 60 * 1000) {
						localStorage.removeItem('pendingSupportData');
						clearInterval(urlCheckInterval);
						return;
					}
					const currentURL = window.location.pathname;
					if (currentURL.startsWith('/student/c/')) {
						const newChatId = currentURL.split('/student/c/')[1].split('/')[0];
						if (newChatId && supportData.id) updateSupportWithChatId(supportData.id, newChatId);
					}
				} catch {
					localStorage.removeItem('pendingSupportData');
					clearInterval(urlCheckInterval);
				}
			}, 1000);
		}
	});

	onDestroy(() => {
		if (browser) {
			if (chatIdSubscription) chatIdSubscription();
			if (urlCheckInterval) clearInterval(urlCheckInterval);
		}
	});

	$: if ($page && $page.url && browser) {
		currentPath = $page.url.pathname || '';
		if (currentPath.startsWith('/student/c/')) {
			chatIdFromURL = currentPath.replace('/student/c/', '').split('/')[0];
			if (chatIdFromURL && localStorage.getItem('pendingSupportData')) {
				try {
					const supportData = JSON.parse(localStorage.getItem('pendingSupportData') || '{}');
					const supportId = supportData.id;
					const age = Date.now() - (supportData.timestamp || 0);
					if (supportId && age < 30 * 60 * 1000) updateSupportWithChatId(supportId, chatIdFromURL);
					else if (age >= 30 * 60 * 1000) localStorage.removeItem('pendingSupportData');
				} catch {
					localStorage.removeItem('pendingSupportData');
				}
			}
		}
	}

	async function updateSupportWithChatId(supportId: string, chatId: string) {
		if (!supportId || !chatId || !browser || chatId === 'local' || chatId === 'undefined') return;
		let pendingSupportData: string | null;
		try {
			pendingSupportData = localStorage.getItem('pendingSupportData');
			if (!pendingSupportData) return;
			const supportData = JSON.parse(pendingSupportData);
			if (supportData.id !== supportId) return;
			if (Date.now() - (supportData.timestamp || 0) >= 30 * 60 * 1000) {
				localStorage.removeItem('pendingSupportData');
				return;
			}
		} catch {
			localStorage.removeItem('pendingSupportData');
			return;
		}
		try {
			const token = localStorage.getItem('token');
			if (!token) return;
			await updateSupportChatId(token, supportId, chatId);
			localStorage.removeItem('pendingSupportData');
			pendingSupportId = '';
		} catch {
			try {
				const supportData = JSON.parse(pendingSupportData || '{}');
				const attemptCount = (supportData.attempts || 0) + 1;
				if (attemptCount >= 3) localStorage.removeItem('pendingSupportData');
				else {
					supportData.attempts = attemptCount;
					localStorage.setItem('pendingSupportData', JSON.stringify(supportData));
				}
			} catch {
				localStorage.removeItem('pendingSupportData');
			}
		}
	}

	// ── Support popup ──────────────────────────────────────────────────────────
	let showSupportPopup = false;
	let dontShowAgain = false;

	if (browser) {
		dontShowAgain = localStorage.getItem('hideSupportPopup') === 'true';
	}

	$: if (browser) {
		if (dontShowAgain) localStorage.setItem('hideSupportPopup', 'true');
		else localStorage.removeItem('hideSupportPopup');
	}

	function toggleSupportPopup() {
		if (dontShowAgain || (browser && localStorage.getItem('hideSupportPopup') === 'true')) {
			goto('/student/support/create');
			return;
		}
		showSupportPopup = !showSupportPopup;
	}

	function handleCreateSupport() {
		goto('/student/support/create');
		showSupportPopup = false;
	}
</script>

<!-- ── Page header ── -->
<div class="mb-6 flex flex-col sm:flex-row sm:items-start sm:justify-between gap-3">
	<div>
		<h1 class="text-2xl font-bold text-gray-800 dark:text-white flex items-center gap-2">
			Bonjour, {firstName} 👋
		</h1>
		<p class="text-sm text-gray-500 dark:text-gray-400 mt-0.5">{formatDateFR()}</p>
	</div>
	<button
		class="inline-flex items-center gap-2 px-4 py-2 text-sm font-semibold bg-gradient-to-r from-blue-600 to-indigo-600 hover:from-blue-700 hover:to-indigo-700 text-white rounded-full transition shadow-sm shadow-blue-200 dark:shadow-blue-900/30 self-start"
		on:click={toggleSupportPopup}
	>
		<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
			<path fill-rule="evenodd"
				d="M10 3a1 1 0 011 1v5h5a1 1 0 110 2h-5v5a1 1 0 11-2 0v-5H4a1 1 0 110-2h5V4a1 1 0 011-1z"
				clip-rule="evenodd" />
		</svg>
		{$i18n.t('Support')}
	</button>
</div>

<!-- ── Performance section header ── -->
<div class="mb-4">
	<h2 class="text-base font-semibold text-gray-700 dark:text-gray-200">Votre performance</h2>
	<p class="text-xs text-gray-400 dark:text-gray-500 mt-0.5">
		{new Date().toLocaleDateString('fr-FR', { month: 'long', year: 'numeric' })}
	</p>
</div>

<!-- ── Tab navigation ── -->
<div class="flex gap-1 mb-5 border border-gray-200 dark:border-gray-700 rounded-full p-0.5 w-fit bg-white dark:bg-gray-800 shadow-sm">
	<button
		class="px-5 py-1.5 rounded-full text-sm font-medium transition
			{$dashboardTab === 'performance'
				? 'bg-white dark:bg-gray-700 text-gray-800 dark:text-white shadow-sm border border-gray-200 dark:border-gray-600'
				: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200'}"
		on:click={() => dashboardTab.set('performance')}
	>
		Performance
	</button>
	<button
		class="px-5 py-1.5 rounded-full text-sm font-medium transition
			{$dashboardTab === 'productivity'
				? 'bg-white dark:bg-gray-700 text-gray-800 dark:text-white shadow-sm border border-gray-200 dark:border-gray-600'
				: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200'}"
		on:click={() => dashboardTab.set('productivity')}
	>
		Productivité
	</button>
</div>

<!-- ── Tab content ── -->
{#if $dashboardTab === 'performance'}
	<PerformanceTab />
{:else}
	<ProductivityTab />
{/if}

<!-- ── Support popup ── -->
{#if showSupportPopup}
	<div
		class="fixed inset-0 backdrop-blur-sm bg-white/30 dark:bg-black/30 flex items-center justify-center z-50"
		role="dialog"
		aria-modal="true"
		in:fade
	>
		<div
			class="bg-white dark:bg-gray-800 rounded-xl shadow-2xl p-4 w-11/12 sm:w-full max-w-sm mx-auto relative overflow-y-auto max-h-[90vh] ring-1 ring-gray-200 dark:ring-gray-700"
			transition:scale={{ duration: 200 }}
		>
			<button
				class="absolute top-2 right-2 text-gray-400 hover:text-gray-600 dark:hover:text-gray-300 focus:outline-none"
				on:click={() => (showSupportPopup = false)}
			>
				<span class="text-xl font-light">×</span>
			</button>

			<div class="flex justify-center mb-4">
				<img src="/favicon.png" alt="OT Logo" class="w-20 h-20" />
			</div>

			<h2 class="text-center text-lg font-bold text-gray-900 dark:text-white mb-4">
				{$i18n.t('Create Personalized Tutorials for any Subject or Topic')}
			</h2>
			<h3 class="text-center text-md font-medium mb-4 text-gray-900 dark:text-white">
				{$i18n.t('Create Your Learning Path')}
			</h3>

			<div class="space-y-3 mb-6 px-2">
				{#each [
					$i18n.t('Choose your topic and level'),
					$i18n.t('Set your learning objectives'),
					$i18n.t('Enjoy AI-powered personalized learning')
				] as step, idx}
					<div class="flex items-center gap-3">
						<div class="flex-shrink-0 bg-[#004AAD] text-white rounded-full w-6 h-6 flex items-center justify-center">
							<span class="font-bold text-sm">{idx + 1}</span>
						</div>
						<span class="text-sm text-gray-800 dark:text-gray-200">{step}</span>
					</div>
				{/each}
			</div>

			<div class="flex justify-center mb-4">
				<button
					class="bg-indigo-600 hover:bg-indigo-700 text-white py-2 px-8 rounded-full font-medium text-sm transition"
					on:click={handleCreateSupport}
				>
					{$i18n.t('Create My support')}
				</button>
			</div>

			<div class="flex items-center justify-center gap-2">
				<input
					type="checkbox"
					id="dontShowDash"
					bind:checked={dontShowAgain}
					class="h-3 w-3 text-indigo-600 bg-white dark:bg-gray-700 border-gray-300 dark:border-gray-600 rounded"
				/>
				<label for="dontShowDash" class="text-xs text-gray-500 dark:text-gray-400">
					{$i18n.t("Don't show me again")}
				</label>
			</div>
		</div>
	</div>
{/if}
