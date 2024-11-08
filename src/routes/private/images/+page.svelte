<script lang="ts">
	import type { Session, SupabaseClient } from '@supabase/supabase-js';
	import { onMount } from 'svelte';
	import { notes } from '$lib/stores/notes';

	export let data: {
		supabase: SupabaseClient;
		session: Session;
	};

	let images: any[] = [];

	onMount(async () => {
		const { data: imageData, error } = await data.supabase.storage
			.from('images')
			.list(`${data.session.user.id}/images`);

		if (error) {
			console.error('Error loading images:', error);
			return;
		}

		// Filter out placeholder files
		images = imageData?.filter((img) => img.name !== '.emptyFolderPlaceholder') || [];
		console.log(images);
	});

	function findNote(noteId: string) {
		return $notes.find((note) => note.id === noteId) ?? null;
	}
</script>

<div class="flex flex-row w-full h-full">
	<!-- Navigator -->
	<div class="flex flex-col w-full h-full px-4 pt-6">
		<!-- Title -->
		<p class="text-2xl font-medium">Images</p>

		<!-- Filter bar -->
		<div class="flex justify-between w-full gap-2 pt-6 pb-4">
			<p class="text-sm text-gray-500">
				Images are automatically uploaded when you add images to your notes.
			</p>
		</div>

		<div class="flex items-start justify-start w-full h-full overflow-y-scroll">
			<div class="flex flex-row flex-wrap gap-4 p-2">
				{#each images as image}
					<div class="relative aspect-square max-w-[10rem] max-h-[10rem]">
						{#await data.supabase.storage
							.from('images')
							.download(`${data.session.user.id}/images/${image.name}`)}
							<!-- Loading state -->
							<div class="w-full h-full bg-gray-200 rounded-lg" />
						{:then { data }}
							<img
								src={data ? URL.createObjectURL(data) : ''}
								alt={image.name}
								class="object-cover w-full h-full rounded-lg"
							/>
						{/await}

						<div
							class="absolute bottom-0 left-0 right-0 flex items-center justify-center w-full p-2 bg-white rounded-b-lg"
						>
							{#await findNote(image.metadata.note_id) then note}
								{#if note}
									<a
										href={`/private/notes?id=${image.metadata.note_id}`}
										class="text-sm truncate hover:underline">{note.fileName ?? 'Note deleted'}</a
									>
								{:else}
									Note deleted
								{/if}
							{/await}
						</div>
					</div>
				{/each}
			</div>
		</div>
	</div>
</div>
