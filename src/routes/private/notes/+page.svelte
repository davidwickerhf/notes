<script lang="ts">
	import { categories } from '$lib/stores/categories';
	import { onMount, onDestroy } from 'svelte';
	import { derived, writable } from 'svelte/store';
	import { Page } from '$lib/components/ui/pages';
	import { marked } from 'marked'; // Import the Markdown parser
	import * as notesApi from '$lib/supabase/notesApi';
	import * as categoriesApi from '$lib/supabase/categoriesApi';
	import { Button } from '$lib/components/ui/button';
	import { Input } from '$lib/components/ui/input';
	import * as AlertDialog from '$lib/components/ui/alert-dialog';
	import * as Select from '$lib/components/ui/select';
	import * as Dialog from '$lib/components/ui/dialog';
	import * as Collapsible from '$lib/components/ui/collapsible';
	import { Label } from '$lib/components/ui/label';
	import { toast } from 'svelte-sonner';
	import { Trash, Ellipsis, Check, Wand } from 'lucide-svelte';
	import { page } from '$app/stores';
	import * as DropdownMenu from '$lib/components/ui/dropdown-menu';
	import { Badge } from '$lib/components/ui/badge';
	import type { Session, SupabaseClient } from '@supabase/supabase-js';
	import { enhance } from '$app/forms';
	import { goto } from '$app/navigation';
	import { selectedNote } from '$lib/stores/notes';
	import { notes } from '$lib/stores/notes';
	import { tasks } from '$lib/stores/tasks';
	import { AiTask, Task } from '../../../lib/components/ui/components';
	import { MarkdownEditor } from '../../../lib/components/ui/components';

	export let data: {
		session: Session;
		supabase: SupabaseClient;
	};

	let isEditing: boolean = true;
	let noteContent = '';
	let parsedContent = '';
	let fileName = '';
	let textareaRef: any;
	let categoryDialogOpen: boolean = false;
	let categoryDeleteDialogOpen: boolean = false;
	let newCategoryName: string = '';
	let saveTimeout: ReturnType<typeof setTimeout> | null = null;

	// Track the previous note ID to detect actual note changes
	let previousNoteId: string | null = null;

	// Get the categoryid from the URL
	$: categoryId = $page.url.searchParams.get('categoryid');
	$: categoryName =
		categoryId === 'uncategorized'
			? 'Uncategorized'
			: categoryId
				? $categories.find((c) => c.id === categoryId)?.category
				: 'All Notes';
	$: newCategoryName = categoryName ?? '';

	// get noteId from the URL
	$: noteId = $page.url.searchParams.get('id');

	// If noteId is set, set selected note
	$: if (noteId && $page && $filteredNotes) {
		console.log('Finding selected note from parameters: ', noteId);
		if ($filteredNotes.length > 0) {
			selectedNote.set($filteredNotes.find((note) => note.id === noteId) ?? null);
		} else {
			selectedNote.set(null);
		}
	}

	// Filter notes based on the selected category
	$: filteredNotes = derived([notes, categories], ([$notes, $categories]) => {
		let filtered = $notes.filter((note) => !note.deleted);
		if (!categoryId) return filtered;
		if (categoryId === 'uncategorized') {
			return filtered.filter((note) => !note.categoryid);
		}
		return filtered.filter((note) => note.categoryid === categoryId);
	});

	// Update content when the selected note changes
	$: if ($selectedNote && $selectedNote.id !== previousNoteId) {
		console.log('Changed selected note: ', $selectedNote?.fileName);
		previousNoteId = $selectedNote.id;
		loadNoteContent($selectedNote);
	}

	$: extractedTasks =
		$selectedNote && $tasks.filter((task) => task.noteId === $selectedNote.id).length > 0
			? $tasks.filter((task) => task.noteId === $selectedNote.id)
			: [];

	// Category renaming
	function handleCategoryRename(result: { type: string; data?: any }) {
		console.log(result);
		if (result.type === 'failure') {
			console.error(result.data?.error);
			toast.error(result.data?.error || 'An error occurred');
		} else {
			console.log('success');
			if (result.data?.success) {
				categoryDialogOpen = false;
				toast.success(`Category renamed!`);
				// The store should update automatically due to realtime subscription
			} else {
				let error = result.data?.error || 'An error occurred';
				toast.error(error);
			}
		}
	}

	function saveNoteDebounced() {
		if (saveTimeout) clearTimeout(saveTimeout);
		saveTimeout = setTimeout(() => {
			saveNote(false); // Pass false to prevent reloading/resetting the note
		}, 2000);
	}

	// Add a new note
	async function addNote(categoryid: string | null) {
		const newNote: Omit<notesApi.Note, 'id' | 'created_at' | 'updated_at'> = {
			userId: data.session.user.id,
			fileName: 'New Note',
			content: '',
			categoryid: categoryid === 'uncategorized' ? null : categoryid,
			deleted: false
		};
		const createdNote = await notesApi.createNote(data.supabase, newNote);

		if (createdNote) {
			selectedNote.set(createdNote);
			toast.success('Note created');
		} else {
			toast.error('Error creating note');
		}
	}

	async function loadNoteContent(note: notesApi.Note | null, editing: boolean = false) {
		console.log('LOADING NOTE CONTENT: ', note?.fileName);
		if (note) {
			noteContent = note.content;
			fileName = note.fileName;
			parsedContent = marked(noteContent) as string;
			// TODO Missing the parsing of the content???
		} else {
			parsedContent = '';
			noteContent = '';
			fileName = '';
		}
		isEditing = editing;
	}

	async function deleteCategory(id: string) {
		if (!categoryId || categoryId === 'uncategorized') return;

		// Delete the category
		await categoriesApi.deleteCategoryPermanently(data.supabase, id);

		// Remove the categoryid from any notes that have it
		const notesToUpdate = $notes.filter((note) => note.categoryid === id);
		for (let note of notesToUpdate) {
			const updatedNote = await notesApi.updateNote(data.supabase, {
				...note,
				categoryid: null
			});
			if (updatedNote) {
				notes.update((currentNotes) =>
					currentNotes.map((n) => (n.id === updatedNote.id ? updatedNote : n))
				);
			}
		}

		// Update selected note to null
		selectedNote.set(null);

		setTimeout(() => {
			goto('/private/notes');
			toast.success('Category deleted');
		}, 0);
	}

	// Save the currently selected note and switch back to preview mode
	async function saveNote(reload: boolean = false) {
		if (!$selectedNote) {
			toast.error('No note selected');
			return;
		}

		if ($selectedNote.content === noteContent && $selectedNote.fileName === fileName) {
			console.log('No changes detected, skipping save');
			selectedNote.set($selectedNote);
			if (reload) loadNoteContent($selectedNote, false);
			return;
		}

		const updatedNote = await notesApi.updateNote(data.supabase, {
			...$selectedNote,
			content: noteContent,
			fileName: fileName
		});

		if (updatedNote) {
			selectedNote.set(updatedNote);
			// if (reload) loadNoteContent(updatedNote, false);
			if (reload) isEditing = false;
			await extractTasksFromNote(noteContent, updatedNote.id);
		} else {
			toast.error('Error saving note');
		}
	}

	// Delete a note
	async function deleteNote(noteId: string) {
		const deletedNote = await notesApi.moveToTrash(data.supabase, noteId);
		if (deletedNote) {
			// notes.update((currentNotes) => currentNotes.filter((note) => note.id !== noteId));
			selectedNote.set(null);
			toast.success('Note moved to trash');
		} else {
			toast.error('Error deleting note');
		}
	}

	function formatDate(date: string) {
		const dateObj = new Date(date);
		return dateObj.toLocaleDateString('en-US', { month: 'short', day: 'numeric', hour: 'numeric' });
	}

	// Function to handle category change
	async function handleCategoryChange(newCategoryId: string) {
		if ($selectedNote) {
			const updatedNote: notesApi.Note = {
				...$selectedNote,
				categoryid: newCategoryId !== 'uncategorized' ? newCategoryId : null
			};
			const result = await notesApi.updateNote(data.supabase, updatedNote);
			if (result) {
				selectedNote.set(result);
				notes.update((currentNotes) =>
					currentNotes.map((note) => (note.id === result.id ? result : note))
				);
				toast.success('Category updated');
			} else {
				toast.error('Failed to update category');
			}
		}
	}

	async function extractTasksFromNote(noteContent: string, noteId: string): Promise<string[]> {
		try {
			const res = await fetch('/api/extract-tasks', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					noteContent,
					noteId
				})
			});

			if (!res.ok) {
				const errorData = await res.json();
				console.error('Error extracting tasks:', errorData);
				if (errorData.error.type === 'insufficient_quota') {
					toast.error('AI quota exceeded. Please try again later or contact support.');
				} else {
					toast.error('Failed to extract tasks. Please try again.');
				}
				return [];
			}

			const data = await res.json();
			console.log('Extracted tasks:', data.tasks);
			return data.tasks;
		} catch (error) {
			console.error('Error extracting tasks:', error);
			toast.error('An unexpected error occurred. Please try again.');
			return [];
		}
	}
</script>

<Page title={categoryName}>
	<div slot="filter-bar" class="flex items-center justify-between w-full">
		<p class="text-sm text-primary/40">{$filteredNotes.length} notes</p>

		<!-- Action buttons -->
		<div class="flex items-center gap-2">
			{#if categoryId}
				<DropdownMenu.Root>
					<DropdownMenu.Trigger>
						<Button variant="ghost" size="icon">
							<Ellipsis class="w-4 h-4"></Ellipsis>
						</Button>
					</DropdownMenu.Trigger>
					<DropdownMenu.Content>
						<DropdownMenu.Group>
							<DropdownMenu.Label>Category {categoryName}</DropdownMenu.Label>
							<DropdownMenu.Separator />
							<DropdownMenu.Item
								on:click={() => {
									addNote(categoryId);
								}}>Add new note to {categoryName}</DropdownMenu.Item
							>
							{#if !(categoryId === 'uncategorized')}
								<DropdownMenu.Item
									on:click={() => {
										categoryDialogOpen = true;
									}}>Rename category</DropdownMenu.Item
								>
								<DropdownMenu.Item
									on:click={() => {
										categoryDeleteDialogOpen = true;
									}}>Delete category</DropdownMenu.Item
								>
							{/if}
						</DropdownMenu.Group>
					</DropdownMenu.Content>
				</DropdownMenu.Root>

				<!-- Add Category -->
				<Dialog.Root bind:open={categoryDialogOpen}>
					<Dialog.Content>
						<Dialog.Header>
							<Dialog.Title>Rename category</Dialog.Title>
							<Dialog.Description>
								Categories are useful for grouping notes around a common topic. They are private to
								you.
							</Dialog.Description>

							<form
								method="POST"
								action="?/renamecategory"
								class="flex flex-col w-full gap-2 pt-8"
								use:enhance={() => {
									return ({ result }) => {
										handleCategoryRename(result);
									};
								}}
							>
								<Label for="category-name" class="text-sm font-medium">Category name</Label>
								<input name="category-id" value={categoryId} hidden />
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

				<!-- Delete category -->

				<AlertDialog.Root bind:open={categoryDeleteDialogOpen}>
					<AlertDialog.Content>
						<AlertDialog.Header>
							<AlertDialog.Title>Are you absolutely sure?</AlertDialog.Title>
							<AlertDialog.Description>
								This action cannot be undone. This will permanently delete this category. All notes
								under this category will NOT be deleted, but they will be moved to "Uncategorized".
							</AlertDialog.Description>
						</AlertDialog.Header>
						<AlertDialog.Footer>
							<AlertDialog.Cancel>Cancel</AlertDialog.Cancel>
							<AlertDialog.Action on:click={() => deleteCategory(categoryId)}
								>Continue</AlertDialog.Action
							>
						</AlertDialog.Footer>
					</AlertDialog.Content>
				</AlertDialog.Root>
			{/if}
		</div>
	</div>
	<div slot="navigator" class="w-full">
		<!-- Notes list -->
		<div class="flex flex-col w-full h-full gap-2 overflow-y-scroll">
			{#each $filteredNotes as note (note.id)}
				<button
					class="flex flex-col w-full px-6 py-8 border rounded-md h-28 hover:bg-gray-100/80 {$selectedNote &&
					note.id === $selectedNote.id
						? 'bg-gray-100'
						: 'bg-white'}"
					on:click={() => {
						// Set selected
						console.log('Set selected: ', note.id);
						selectedNote.set(note);
					}}
				>
					<p class="text-sm font-normal">{note.fileName}</p>
					<div class="flex w-full gap-2">
						<p class="text-sm font-normal text-primary/40">{formatDate(note.updated_at)}</p>

						{#if !categoryId}
							{#if note.categoryid}
								{#await $categories.find((c) => c.id === note.categoryid)}
									<p class="text-xs font-light text-primary/40">Loading...</p>
								{:then category}
									<Badge class="rounded-sm" variant="secondary">
										{category?.category || 'Uncategorized'}
									</Badge>
								{/await}
							{/if}
						{/if}
					</div>
				</button>
			{/each}
		</div>
	</div>

	<!-- Note Content -->
	<div slot="content" class="flex flex-col items-start justify-start h-full px-4 pt-4">
		{#if $selectedNote}
			<div class="flex flex-col w-full h-full gap-8">
				<!-- Action bar -->
				<div class="flex flex-col gap-2">
					<div class="flex justify-between w-full">
						<!-- Category picker -->
						{#key $selectedNote.categoryid}
							<Select.Root
								onSelectedChange={(v) => {
									if (v && typeof v.value === 'string') {
										handleCategoryChange(v.value);
									}
								}}
								selected={$selectedNote?.categoryid
									? { value: $selectedNote.categoryid }
									: { value: 'uncategorized' }}
							>
								<Select.Trigger class="w-[180px]">
									{$categories.find((e) => e.id === $selectedNote?.categoryid)?.category ??
										'Choose a category'}
								</Select.Trigger>
								<Select.Content>
									{#each $categories as category}
										<Select.Item value={category.id}>{category.category}</Select.Item>
									{/each}

									<Select.Item value="uncategorized">Uncategorized</Select.Item>
								</Select.Content>
							</Select.Root>
						{/key}

						<div class="flex gap-2">
							<!-- Save Button (if changes present) -->
							{#if noteContent !== $selectedNote.content || fileName !== $selectedNote.fileName}
								<Button variant="outline" on:click={() => saveNote(true)}
									><Check class="w-4 h-4 mr-2"></Check>Save Changes</Button
								>
							{/if}

							<!-- Delete button -->
							<AlertDialog.Root>
								<AlertDialog.Trigger>
									<Button size="icon" variant="outline"><Trash class="w-4 h-4"></Trash></Button>
								</AlertDialog.Trigger>
								<AlertDialog.Content>
									<AlertDialog.Header>
										<AlertDialog.Title>Are you absolutely sure?</AlertDialog.Title>
										<AlertDialog.Description>
											This will move the note into the trash. You can always restore the note from
											the trash, or you can delete it permantely.
										</AlertDialog.Description>
									</AlertDialog.Header>
									<AlertDialog.Footer>
										<AlertDialog.Cancel>Cancel</AlertDialog.Cancel>
										<AlertDialog.Action
											on:click={() => {
												// Delete note
												if ($selectedNote) deleteNote($selectedNote.id);
											}}>Continue</AlertDialog.Action
										>
									</AlertDialog.Footer>
								</AlertDialog.Content>
							</AlertDialog.Root>
						</div>
					</div>
					<div class="flex items-center justify-between w-full">
						<p class="text-sm text-primary/40">
							Last edited on {formatDate($selectedNote.updated_at)}
						</p>
					</div>
				</div>

				<!-- Extracted Tasks -->
				{#if extractedTasks.length > 0}
					<Collapsible.Root class="w-full">
						<Collapsible.Trigger class="w-full"
							><div class="flex flex-col w-full gap-2 p-4 border rounded-md">
								<p class="flex items-center text-sm font-semibold">
									<Wand class="w-4 h-4 mr-4"></Wand>
									Found {extractedTasks.length} tasks in this note
								</p>
							</div></Collapsible.Trigger
						>
						<Collapsible.Content>
							<div class="flex flex-col w-full gap-2 pt-2">
								{#each extractedTasks as task}
									{#if task.aiGenerated}
										<AiTask {task} {data} />
									{:else}
										<Task {task} {data} />
									{/if}
								{/each}
							</div>
						</Collapsible.Content>
					</Collapsible.Root>
				{/if}

				<div class="flex flex-col w-full h-full gap-8 px-8 pt-4">
					<!-- Title -->
					<input
						on:blur={() => saveNote(true)}
						on:input={saveNoteDebounced}
						bind:value={fileName}
						class="text-2xl font-semibold bg-transparent border-none outline-none focus:ring-0"
					/>

					<!-- Text area -->
					<MarkdownEditor
						noteId={$selectedNote.id}
						{textareaRef}
						supabase={data.supabase}
						bind:noteContent
						bind:isEditing
						bind:parsedContent
						onInput={saveNoteDebounced}
						onBlur={() => {
							saveNote(true);
						}}
					/>
				</div>
			</div>
		{/if}
	</div>
</Page>
