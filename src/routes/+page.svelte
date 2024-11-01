<script lang="ts">
	import { onMount } from 'svelte';
	import { Play, Pause, SkipForward, Plus, X, GripVertical, Check } from 'lucide-svelte';
	import { flip } from 'svelte/animate';
	import { dragHandle, dragHandleZone } from 'svelte-dnd-action';

	type Task = {
		id: string;
		title: string;
		duration: number;
		deletedAt?: number;
		completedAt?: number;
		startedAt?: number;
		timeElapsed?: number;
	};

	type TasksDatabase = Record<string, Task>;

	let mode = $state('edit'); // 'edit' or 'play'
	let tasksDatabase = $state<TasksDatabase>({});
	let taskIds = $state<string[]>([]);
	// let tasks = $derived(taskIds.map((id) => tasksDatabase[id]));
	let tasks = $state<Task[]>([]);
	let newTaskTitle = $state('');
	let timerInterval: undefined | number = $state(undefined);
	let playState = $state({
		currentTaskIndex: 0,
		timeRemaining: 0,
		isPlaying: false,
		lastUpdate: Date.now(),
		taskIds: [] as string[]
	});
	let currentTask = $derived(tasksDatabase[playState.taskIds[playState.currentTaskIndex]]);
	let planState = $state({});

	$effect(() => {
		tasks = taskIds.map((id) => tasksDatabase[id]);
	});

	onMount(() => {
		const savedTasksDatabase = localStorage.getItem('tasksDatabase');
		const savedTaskIds = localStorage.getItem('taskIds');
		if (savedTasksDatabase && savedTaskIds) {
			tasksDatabase = JSON.parse(savedTasksDatabase);
			taskIds = JSON.parse(savedTaskIds);
		}
	});

	// Save tasks whenever they change
	$effect(() => {
		if (tasks) {
			localStorage.setItem('tasksDatabase', JSON.stringify(tasksDatabase));
			localStorage.setItem('taskIds', JSON.stringify(taskIds));
		}
	});

	// Timer logic
	$effect(() => {
		if (playState.isPlaying && currentTask) {
			timerInterval = setInterval(() => {
				const elapsedTime = Date.now() - (currentTask.startedAt ?? Date.now());
				const timeRemaining = currentTask.duration - Math.floor(elapsedTime / 1000);

				if (timeRemaining <= 0) {
					playState.isPlaying = false;
					playState.timeRemaining = 0;
					clearInterval(timerInterval);
				} else {
					playState.timeRemaining = timeRemaining;
				}
			}, 1000);

			return () => clearInterval(timerInterval);
		}
	});

	function addTask(e: Event) {
		e.preventDefault();

		if (!newTaskTitle) return;

		// Parse title and duration from format "title duration"
		const parts = newTaskTitle.trim().split(/\s+/);
		if (parts.length < 2) {
			alert('Please enter task in format "title duration" (e.g. "go to store 5")');
			return;
		}

		const durationStr = parts[parts.length - 1];
		const title = parts.slice(0, -1).join(' ');
		const durationMins = parseInt(durationStr);

		if (isNaN(durationMins)) {
			alert('Duration must be a number');
			return;
		}

		if (durationMins <= 0) {
			alert('Duration must be greater than 0');
			return;
		}

		const duration = durationMins * 60; // Convert to seconds
		const id = crypto.randomUUID();
		tasksDatabase[id] = { id, title, duration };
		taskIds.push(id);
		newTaskTitle = '';

		// Return focus to input for mobile
		const input = document.querySelector('input[type="text"]') as HTMLInputElement;
		input?.focus();
	}

	function removeTask(id: string) {
		tasksDatabase[id].deletedAt = Date.now();
		taskIds = taskIds.filter((taskId) => taskId !== id);
	}

	function startPlaylist() {
		if (tasks.length === 0) return;
		mode = 'play';
		playState.currentTaskIndex = 0;
		playState.taskIds = [...taskIds];
		playState.timeRemaining = tasksDatabase[playState.taskIds[0]].duration;
		tasksDatabase[currentTask.id].startedAt = Date.now();
		playState.isPlaying = true;
	}

	function nextTask() {
		if (playState.currentTaskIndex < playState.taskIds.length - 1) {
			playState.currentTaskIndex++;
			playState.timeRemaining =
				tasksDatabase[playState.taskIds[playState.currentTaskIndex]].duration;
			playState.isPlaying = true;
			tasksDatabase[currentTask.id].startedAt = Date.now();
		} else {
			mode = 'edit';
			playState.isPlaying = false;
		}
	}

	function skipTask() {
		tasksDatabase[currentTask.id].timeElapsed =
			Date.now() - (tasksDatabase[currentTask.id]?.startedAt ?? Date.now());
		nextTask();
	}
	function completeTask() {
		tasksDatabase[currentTask.id].completedAt = Date.now();
		taskIds = taskIds.filter((taskId) => taskId !== currentTask.id);
		nextTask();
	}

	function formatTime(seconds: number) {
		const hours = Math.floor(seconds / 3600);
		const mins = Math.floor((seconds % 3600) / 60);
		const secs = seconds % 60;
		if (hours > 0) {
			return `${hours}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
		}
		return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
	}

	function handleDndConsider(e: CustomEvent<{ items: Task[] }>) {
		console.log('CONSIDERING');
		console.log(e.detail.items);
		// taskIds = e.detail.items.map((item) => item.id);
		tasks = e.detail.items;
		// taskIds = e.detail.items.map((item) => item.id);
	}

	function handleDndFinalize(e: CustomEvent<{ items: Task[] }>) {
		console.log('FINALIZING');
		console.log(e.detail.items);
		tasks = e.detail.items;
		taskIds = e.detail.items.map((item) => item.id);
	}

	const flipDurationMs = 300;

	$effect(() => {
		console.log(tasksDatabase);
		localStorage.setItem('tasksDatabase', JSON.stringify(tasksDatabase));
		localStorage.setItem('playState', JSON.stringify(playState));
		localStorage.setItem('taskIds', JSON.stringify(taskIds));
	});
</script>

<div class="max-w-md mx-auto p-4 space-y-4">
	{#if mode === 'edit'}
		<div class="space-y-4">
			<form onsubmit={addTask} class="space-y-2">
				<div class="flex gap-2">
					<input
						type="text"
						bind:value={newTaskTitle}
						placeholder="task <duration in min>"
						class="w-full p-2 border rounded"
						required
						autocomplete="off"
						data-lpignore="true"
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
				}}
				onconsider={handleDndConsider}
				onfinalize={handleDndFinalize}
			>
				{#each tasks.filter((x) => !!x) as task (task.id)}
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
			<div class="text-center text-sm text-gray-500">
				Total time: {formatTime(tasks.reduce((acc, task) => acc + task.duration, 0))}
			</div>
		</div>
	{:else}
		<div class="space-y-4">
			<div
				class="text-center p-6 {playState.isPlaying
					? 'bg-blue-500'
					: 'bg-gray-500'} text-white rounded-lg"
			>
				<div class="text-sm mb-2">Current Task</div>
				<div class="text-2xl font-bold mb-4">
					{tasksDatabase[playState.taskIds[playState.currentTaskIndex]]?.title}
				</div>
				<div class="text-4xl font-mono">
					{formatTime(playState.timeRemaining)}
				</div>
			</div>

			<div class="flex justify-center gap-4">
				<button
					onclick={() => (playState.isPlaying = !playState.isPlaying)}
					class="p-3 bg-gray-100 rounded-full hover:bg-gray-200"
				>
					{#if playState.isPlaying}
						<Pause class="w-6 h-6" />
					{:else}
						<Play class="w-6 h-6" />
					{/if}
				</button>

				<button onclick={skipTask} class="p-3 bg-gray-100 rounded-full hover:bg-gray-200">
					<SkipForward class="w-6 h-6" />
				</button>
				<button onclick={completeTask} class="p-3 bg-gray-100 rounded-full hover:bg-gray-200">
					<Check class="w-6 h-6" />
				</button>
			</div>

			<div class="text-center text-sm text-gray-500">
				Task {playState.currentTaskIndex + 1} of {playState.taskIds.length}
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
