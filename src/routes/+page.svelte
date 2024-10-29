<script lang="ts">
	import { onMount } from 'svelte';
	import { Clock, Play, Pause, SkipForward, Plus, X, Save, GripVertical } from 'lucide-svelte';
	import { flip } from 'svelte/animate';
	import { dragHandle, dragHandleZone } from 'svelte-dnd-action';

	type Task = {
		id: string;
		title: string;
		duration: number;
	};

	let mode = $state('edit'); // 'edit' or 'play'
	let tasks = $state<Task[]>([]);
	let currentTaskIndex = $state(0);
	let timeRemaining = $state(0);
	let isPlaying = $state(false);
	let newTaskTitle = $state('');
	let newTaskDuration = $state('');
	let timerInterval: undefined | number = $state(undefined);
	let draggedIndex = $state<number | null>(null);
	let dropTarget = $state<number | null>(null);
	let touchStartY = $state<number | null>(null);
	let draggedElement = $state<HTMLElement | null>(null);

	onMount(() => {
		const savedTasks = localStorage.getItem('tasks');
		if (savedTasks) {
			tasks = JSON.parse(savedTasks);
		}
	});

	// Save tasks whenever they change
	$effect(() => {
		if (tasks) {
			localStorage.setItem('tasks', JSON.stringify(tasks));
		}
	});

	// Timer logic
	$effect(() => {
		if (isPlaying && timeRemaining > 0) {
			timerInterval = setInterval(() => {
				if (timeRemaining <= 1) {
					isPlaying = false;
					timeRemaining = 0;
					clearInterval(timerInterval);
				} else {
					timeRemaining--;
				}
			}, 1000);

			return () => clearInterval(timerInterval);
		}
	});

	function addTask(e: Event) {
		e.preventDefault();
		if (!newTaskTitle || !newTaskDuration) return;

		const duration = parseInt(newTaskDuration) * 60;
		tasks = [
			...tasks,
			{
				id: crypto.randomUUID(),
				title: newTaskTitle,
				duration
			}
		];
		newTaskTitle = '';
		newTaskDuration = '';
		console.log(tasks);
	}

	function removeTask(id: string) {
		tasks = tasks.filter((task) => task.id !== id);
	}

	function startPlaylist() {
		if (tasks.length === 0) return;
		mode = 'play';
		currentTaskIndex = 0;
		timeRemaining = tasks[0].duration;
		isPlaying = true;
	}

	function skipTask() {
		if (currentTaskIndex < tasks.length - 1) {
			currentTaskIndex++;
			timeRemaining = tasks[currentTaskIndex].duration;
			isPlaying = true;
		} else {
			mode = 'edit';
			isPlaying = false;
		}
	}

	function formatTime(seconds: number) {
		const mins = Math.floor(seconds / 60);
		const secs = seconds % 60;
		return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
	}

	function handleDndConsider(e: CustomEvent<{ items: Task[] }>) {
		tasks = e.detail.items;
	}

	function handleDndFinalize(e: CustomEvent<{ items: Task[] }>) {
		tasks = e.detail.items;
	}

	const flipDurationMs = 300;
</script>

<div class="max-w-md mx-auto p-4 space-y-4">
	{#if mode === 'edit'}
		<div class="space-y-4">
			<h1 class="text-2xl font-bold text-gray-900">Task Playlist</h1>

			<form onsubmit={addTask} class="space-y-2">
				<input
					type="text"
					bind:value={newTaskTitle}
					placeholder="Task name"
					class="w-full p-2 border rounded"
					minlength="1"
					maxlength="100"
					required
				/>
				<div class="flex gap-2">
					<input
						type="number"
						bind:value={newTaskDuration}
						placeholder="Duration (minutes)"
						class="flex-1 p-2 border rounded"
						min="1"
						max="120"
						required
					/>
					<button type="submit" class="p-2 bg-blue-500 text-white rounded hover:bg-blue-600">
						<Plus class="w-5 h-5" />
					</button>
				</div>
			</form>

			<div
				class="space-y-2"
				use:dragHandleZone={{
					items: tasks,
					flipDurationMs
					// dragHandle: '.handle'
				}}
				onconsider={handleDndConsider}
				onfinalize={handleDndFinalize}
			>
				{#each tasks as task (task.id)}
					<div
						animate:flip={{ duration: flipDurationMs }}
						class="task-item flex items-center justify-between p-3 bg-gray-50 rounded hover:bg-gray-100 transition-colors"
					>
						<div class="flex items-center gap-3">
							<div class="touch-none cursor-move handle" use:dragHandle>
								<GripVertical class="w-5 h-5 text-gray-400" />
							</div>
							<div>
								<div class="font-medium">{task.title}</div>
								<div class="text-sm text-gray-500 select-none">{formatTime(task.duration)}</div>
							</div>
						</div>
						<button
							onclick={() => removeTask(task.id)}
							class="p-1 text-gray-500 hover:text-red-500"
						>
							<X class="w-5 h-5" />
						</button>
					</div>
				{/each}
			</div>

			{#if tasks.length > 0}
				<button
					onclick={startPlaylist}
					class="w-full p-3 bg-green-500 text-white rounded-lg hover:bg-green-600 flex items-center justify-center gap-2"
				>
					<Play class="w-5 h-5" />
					Start Playlist
				</button>
			{/if}
		</div>
	{:else}
		<div class="space-y-4">
			<div class="text-center p-6 bg-blue-500 text-white rounded-lg">
				<div class="text-sm mb-2">Current Task</div>
				<div class="text-2xl font-bold mb-4">
					{tasks[currentTaskIndex].title}
				</div>
				<div class="text-4xl font-mono">
					{formatTime(timeRemaining)}
				</div>
			</div>

			<div class="flex justify-center gap-4">
				<button
					onclick={() => (isPlaying = !isPlaying)}
					class="p-3 bg-gray-100 rounded-full hover:bg-gray-200"
				>
					{#if isPlaying}
						<Pause class="w-6 h-6" />
					{:else}
						<Play class="w-6 h-6" />
					{/if}
				</button>

				<button onclick={skipTask} class="p-3 bg-gray-100 rounded-full hover:bg-gray-200">
					<SkipForward class="w-6 h-6" />
				</button>
			</div>

			<div class="text-center text-sm text-gray-500">
				Task {currentTaskIndex + 1} of {tasks.length}
			</div>
		</div>
	{/if}
</div>

<style>
	:global(.handle) {
		cursor: move;
	}
	:global(.task-item) {
		cursor: default !important;
	}
</style>
