<script lang="ts">
	import Input from '$lib/components/ui/input/input.svelte';
	import Button from '$lib/components/ui/button/button.svelte';
	import Separator from '$lib/components/ui/separator/separator.svelte';
	import SidebarButton from './sidebar-button.svelte';
	import * as Collapsible from '$lib/components/ui/collapsible';
	import * as Dialog from '$lib/components/ui/dialog';
	import * as Popover from '$lib/components/ui/popover';
	import { Label } from '$lib/components/ui/label';
	import { toast } from 'svelte-sonner';
	import type { Category } from '$lib/supabase/categoriesApi';
	import * as DropdownMenu from '$lib/components/ui/dropdown-menu';
	import { notes } from '$lib/stores/notes';
	import { tasks } from '$lib/stores/tasks';
	import { profile } from '$lib/stores/profile';
	import { selectedNote } from '$lib/stores/notes';
	import { getAvatarUrl } from '$lib/utils';
	import type { SupabaseClient } from '@supabase/supabase-js';

	import {
		Trash,
		Menu,
		X,
		Notebook,
		CircleCheckBig,
		ChevronRight,
		Tag,
		House,
		Archive,
		CirclePlus,
		Check,
		Search,
		ChevronsUpDown,
		User,
		LogOut,
		Image
	} from 'lucide-svelte';

	import { goto } from '$app/navigation';
	import { enhance } from '$app/forms';

	export let email: string;
	export let selectedTab: string;
	export let categories: Category[];
	export let supabase: SupabaseClient;

	export let addNote: (categoryid: string | null) => void;
	export let addTask: (content: string) => void;
	export let logout: () => void;

	let showCategories: boolean = false;
	let newCategoryName = '';
	let dialogOpen = false;
	let searchOpen = false;
	let profilePopupOpen = false;

	let searchQuery = '';
	let searchResults: Array<{ type: 'note' | 'task'; item: any }> = [];

	$: {
		if (searchQuery.length > 0) {
			const filteredNotes = $notes.filter(
				(note) =>
					note.fileName.toLowerCase().includes(searchQuery.toLowerCase()) ||
					note.content.toLowerCase().includes(searchQuery.toLowerCase())
			);
			const filteredTasks = $tasks.filter((task) =>
				task.task.toLowerCase().includes(searchQuery.toLowerCase())
			);
			searchResults = [
				...filteredNotes.map((note) => ({ type: 'note' as const, item: note })),
				...filteredTasks.map((task) => ({ type: 'task' as const, item: task }))
			];
		} else {
			searchResults = [];
		}
	}

	function handleResultClick(result: { type: 'note' | 'task'; item: any }) {
		if (result.type === 'note') {
			selectedNote.set(result.item);
			if (result.item.deleted) {
				goto('/private/trash');
			} else {
				goto('/private/notes');
			}
		} else {
			goto('/private/tasks');
		}
		resetSearch();
	}

	function handleNewCategory(result: { type: string; data?: any }) {
		console.log(result);
		if (result.type === 'failure') {
			console.error(result.data?.error);
			toast.error(result.data?.error || 'An error occurred');
		} else {
			console.log('success');
			if (result.data?.success) {
				dialogOpen = false;
				toast.success(`Category created!`);
				// The store should update automatically due to realtime subscription
				goto(`/private/notes?categoryid=${result.data.category.id}`);
			} else {
				let error = result.data?.error || 'An error occurred';
				toast.error(error);
			}
		}
	}

	function resetSearch() {
		searchQuery = '';
		searchResults = [];
		searchOpen = false;
	}
</script>

<div
	class="min-w-[300px] flex flex-col bg-sidebar rounded-md justify-between border-r h-full border"
>
	<div class="flex flex-col h-full overflow-hidden">
		<!-- Header -->
		<div class="flex flex-col items-start w-full gap-2 px-4 pt-4">
			<DropdownMenu.Root bind:open={profilePopupOpen}>
				<DropdownMenu.Trigger class="w-full">
					<Button
						variant="ghost"
						class="items-center justify-start w-full pl-0 mr-0 hover:cursor-pointer h-fit"
					>
						<div class="flex flex-row items-center justify-between w-full gap-2 h-fit">
							<div class="flex flex-row items-center gap-2">
								<!-- Profile Avatar -->
								{#if $profile?.avatar_url}
									{#await getAvatarUrl(supabase, $profile?.avatar_url)}
										<div class="flex items-center justify-center bg-gray-200 rounded-full w-9 h-9">
											<p class="text-sm">{email.slice(0, 2).toUpperCase()}</p>
										</div>
									{:then url}
										<img
											src={url}
											alt="Profile"
											class="object-cover border rounded-full w-9 h-9"
											on:error={(e) => {
												if (e.target instanceof HTMLImageElement) {
													e.target.src = 'https://robohash.org/robot.png';
												}
											}}
										/>
									{/await}
								{:else}
									<div class="flex items-center justify-center bg-gray-200 rounded-full w-9 h-9">
										<p class="text-sm">{email.slice(0, 2).toUpperCase()}</p>
									</div>
								{/if}

								<div class="flex flex-col items-start">
									<p class="text-sm">{$profile?.full_name}</p>
									<p class="text-xs text-muted-foreground">{email}</p>
								</div>
							</div>

							<ChevronsUpDown class="w-4 h-4" />
						</div>
					</Button>
				</DropdownMenu.Trigger>
				<DropdownMenu.Content class="w-[300px]" side="bottom" align="start" sideOffset={5}>
					<DropdownMenu.Group>
						<DropdownMenu.Label>
							<Button
								size="sm"
								variant="ghost"
								class="justify-start w-full"
								on:click={() => {
									profilePopupOpen = false;
									goto('/private/settings/profile');
								}}
							>
								<User class="w-4 h-4 mr-2" />
								Profile
							</Button>
						</DropdownMenu.Label>
						<DropdownMenu.Separator />

						<DropdownMenu.Item>
							<Button size="sm" variant="ghost" class="justify-start w-full" on:click={logout}>
								<LogOut class="w-4 h-4 mr-2" />
								Log Out
							</Button>
						</DropdownMenu.Item>
					</DropdownMenu.Group>
				</DropdownMenu.Content>
			</DropdownMenu.Root>

			<!-- Search Input -->
			<Popover.Root bind:open={searchOpen}>
				<Popover.Trigger class="w-full">
					<Button variant="outline" class="justify-start w-full hover:cursor-text">
						<Search class="w-4 h-4 mr-2" /> Search</Button
					></Popover.Trigger
				>
				<Popover.Content class="w-[400px]" side="bottom" align="start" sideOffset={5}>
					<div class="flex flex-col gap-2">
						<div class="flex flex-row items-center gap-2">
							<Search class="w-4 h-4 mr-2" />
							<input
								type="text"
								placeholder="Search notes and tasks"
								class="w-full bg-transparent border-0 focus:ring-0 focus:outline-none placeholder-muted-foreground text-foreground"
								bind:value={searchQuery}
							/>
						</div>
						{#if searchQuery.length > 0}
							{#if searchResults.length > 0}
								<div class="max-h-[300px] overflow-y-auto">
									{#each searchResults as result}
										<button
											class="flex flex-row items-center justify-start w-full gap-2 px-4 py-2 text-left rounded hover:bg-gray-100"
											on:click={() => handleResultClick(result)}
										>
											<p class="text-sm">
												{result.type === 'note' ? result.item.fileName : result.item.task}
											</p>
											<p class="text-xs text-muted-foreground">
												{result.type === 'note' ? 'Note' : 'Task'}
											</p>
											<p class="text-xs text-muted-foreground">
												{result.type === 'note' && result.item.deleted ? '(Deleted)' : ''}
											</p>
										</button>
									{/each}
								</div>
							{:else}
								<p class="text-sm text-gray-500">No results found</p>
							{/if}
						{:else}
							<div class="max-h-[300px] overflow-y-auto pt-6">
								<p class="mb-2 font-mono text-xs font-normal text-muted-foreground">GO TO...</p>
								{#each $notes
									.filter((note) => !note.deleted)
									.sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
									.slice(0, 3) as note}
									<button
										class="flex flex-row items-center justify-start w-full gap-2 px-4 py-2 text-left rounded hover:bg-gray-100"
										on:click={() => handleResultClick({ type: 'note', item: note })}
									>
										<Notebook class="w-4 h-4 mr-2" />
										<p class="text-sm">{note.fileName}</p>
									</button>
								{/each}
							</div>
						{/if}
					</div>
				</Popover.Content>
			</Popover.Root>

			<!-- Add Note or Task List Buttons -->

			<div class="flex flex-row justify-between w-full gap-2">
				<Button on:click={() => addNote(null)} class="w-full">
					<Notebook class="w-4 h-4 mr-2" />
					Add Note</Button
				>

				<Button on:click={() => addTask('')} class="w-full">
					<CircleCheckBig class="w-4 h-4 mr-2" />
					Add Task
				</Button>
			</div>
		</div>

		<!-- List of options -->
		<div class="flex flex-col flex-grow w-full gap-2 px-4 pb-8 mt-8 overflow-y-scroll">
			<SidebarButton
				icon={House}
				text="Home"
				selected={selectedTab === 'home'}
				onClick={() => {
					goto('/private');
				}}
			/>

			<SidebarButton
				icon={Notebook}
				text="Notes"
				selected={selectedTab === 'notes'}
				onClick={() => {
					goto('/private/notes');
				}}
			/>

			<SidebarButton
				icon={CircleCheckBig}
				text="Tasks"
				selected={selectedTab === 'tasks'}
				onClick={() => {
					goto('/private/tasks');
				}}
			/>

			<div class="flex flex-row items-start justify-between gap-2">
				<Collapsible.Root bind:open={showCategories} class="w-full">
					<Collapsible.Trigger class="w-full">
						<Button size="sm" variant="ghost" class="flex justify-between w-full">
							<div class="flex flex-row items-center w-full gap-4">
								<ChevronRight
									class="w-4 h-4 transition-transform duration-300 {showCategories
										? 'rotate-90'
										: ''}"
								/>
								<p class="font-normal">Categories</p>
							</div>
						</Button>
					</Collapsible.Trigger>
					<Collapsible.Content>
						<div class="flex flex-col gap-2 pl-6">
							{#each categories as category}
								<SidebarButton
									icon={Tag}
									text={category.category}
									selected={selectedTab === category.id}
									onClick={() => {
										goto(`/private/notes?categoryid=${category.id}`);
									}}
								/>
							{/each}

							<SidebarButton
								icon={Archive}
								text="Uncategorized"
								selected={selectedTab === 'uncategorized'}
								onClick={() => {
									goto(`/private/notes?categoryid=uncategorized`);
								}}
							/>
						</div>
					</Collapsible.Content>
				</Collapsible.Root>

				<!-- Add Category -->
				<Dialog.Root bind:open={dialogOpen}>
					<Dialog.Trigger
						><Button
							class="z-50"
							variant="ghost"
							size="icon"
							on:click={(event) => {
								event.stopPropagation();
								dialogOpen = true;
								showCategories = true;
							}}
						>
							<CirclePlus class="w-4 h-4"></CirclePlus>
						</Button></Dialog.Trigger
					>
					<Dialog.Content>
						<Dialog.Header>
							<Dialog.Title>Create new category</Dialog.Title>
							<Dialog.Description>
								Categories are useful for grouping notes around a common topic. They are private to
								you.
							</Dialog.Description>

							<form
								method="POST"
								action="?/newcategory"
								class="flex flex-col w-full gap-2 pt-8"
								use:enhance={() => {
									return ({ result }) => {
										handleNewCategory(result);
									};
								}}
							>
								<Label for="category-name" class="text-sm font-medium">Category name</Label>
								<Input
									name="category-name"
									id="category-name"
									placeholder="Enter category name"
									bind:value={newCategoryName}
									required
								/>

								<div class="flex justify-end w-full pt-6">
									<Button type="submit"><Check class="w-4 h-4 mr-4"></Check> Save</Button>
								</div>
							</form>
						</Dialog.Header>
					</Dialog.Content>
				</Dialog.Root>
			</div>

			<SidebarButton
				icon={Image}
				text="Images"
				selected={selectedTab === 'images'}
				onClick={() => {
					goto('/private/images');
				}}
			/>

			<SidebarButton
				icon={Trash}
				text="Trash"
				selected={selectedTab === 'trash'}
				onClick={() => {
					goto('/private/trash');
				}}
			/>
		</div>
	</div>

	<!-- Logout -->
	<div class="flex flex-col items-center justify-center">
		<Separator class="py-0 my-0" />
		<div class="flex flex-row items-center justify-center w-full px-6 py-4">
			<Button class="w-full" variant="destructive" on:click={logout}>Logout</Button>
		</div>
	</div>
</div>
