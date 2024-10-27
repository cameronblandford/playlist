<script lang="ts">
	import { onMount } from 'svelte';
	import { Clock, Play, Pause, SkipForward, Plus, X, Save, GripVertical } from 'lucide-svelte';

	type Task = {
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

		const duration = parseInt(newTaskDuration) * 60; // Convert minutes to seconds
		tasks = [...tasks, { title: newTaskTitle, duration }];
		newTaskTitle = '';
		newTaskDuration = '';
	}

	function removeTask(index: number) {
		tasks = tasks.filter((_, i) => i !== index);
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

	// Enhanced drag and drop functions for both mouse and touch
	function handleDragStart(e: DragEvent | TouchEvent, index: number) {
		draggedIndex = index;
		if (e instanceof DragEvent) {
			e.dataTransfer?.setData('text/plain', index.toString());
		} else {
			touchStartY = e.touches[0].clientY;
			(e.target as HTMLElement).closest('.task-item')!.style.opacity = '0.5';
		}
	}

	function handleDragEnd(e: DragEvent | TouchEvent) {
		draggedIndex = null;
		dropTarget = null;
		touchStartY = null;
		if (e instanceof TouchEvent) {
			(e.target as HTMLElement).closest('.task-item')!.style.opacity = '1';
		}
	}

	function handleDragOver(e: DragEvent | TouchEvent, index: number) {
		// e.preventDefault();
		if (draggedIndex === null || draggedIndex === index) return;

		const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
		const midpoint = rect.top + rect.height / 2;
		const clientY = e instanceof DragEvent ? e.clientY : e.touches[0].clientY;

		// If mouse/touch is in bottom half of element, we'll drop after it
		dropTarget = clientY > midpoint ? index + 1 : index;
	}

	function handleDrop(e: DragEvent | TouchEvent) {
		e.preventDefault();
		if (draggedIndex === null || dropTarget === null) return;

		const newTasks = [...tasks];
		const [draggedTask] = newTasks.splice(draggedIndex, 1);

		// Adjust insert position if we removed an item from before the drop target
		const adjustedDropTarget = draggedIndex < dropTarget ? dropTarget - 1 : dropTarget;
		newTasks.splice(adjustedDropTarget, 0, draggedTask);

		tasks = newTasks;
		draggedIndex = null;
		dropTarget = null;
		touchStartY = null;
	}

	function handleTouchMove(e: TouchEvent, index: number) {
		// e.preventDefault();
		if (draggedIndex === null) return;
		handleDragOver(e, index);
	}
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
				/>
				<div class="flex gap-2">
					<input
						type="number"
						bind:value={newTaskDuration}
						placeholder="Duration (minutes)"
						class="flex-1 p-2 border rounded"
						min="1"
					/>
					<button type="submit" class="p-2 bg-blue-500 text-white rounded hover:bg-blue-600">
						<Plus class="w-5 h-5" />
					</button>
				</div>
			</form>

			<div
				class="space-y-2"
				role="list"
				ondragover={(e) => {
					e.preventDefault();
					// If we're not over a task card, we're likely in the "dead zone"
					// Find the closest task based on mouse position
					const rect = e.currentTarget.getBoundingClientRect();
					const mouseY = e.clientY - rect.top;
					const taskHeight = 70; // Approximate height of a task card
					const index = Math.floor(mouseY / taskHeight);
					dropTarget = Math.min(index, tasks.length);
				}}
				ondrop={handleDrop}
				ontouchend={handleDrop}
			>
				{#each tasks as task, index}
					{#if dropTarget === index}
						<div class="h-1 bg-blue-300 rounded transition-all"></div>
					{/if}
					<div
						role="listitem"
						class="task-item flex items-center justify-between p-3 bg-gray-50 rounded hover:bg-gray-100 transition-colors"
						class:opacity-50={draggedIndex === index}
						draggable="true"
						ondragstart={(e) => handleDragStart(e, index)}
						ondragend={handleDragEnd}
						ondragover={(e) => handleDragOver(e, index)}
					>
						<div class="flex items-center gap-3">
							<div
								class="touch-none cursor-move"
								ontouchstart={(e) => handleDragStart(e, index)}
								ontouchmove={(e) => handleTouchMove(e, index)}
								ontouchend={handleDragEnd}
								ontouchcancel={handleDragEnd}
							>
								<GripVertical class="w-5 h-5 text-gray-400" />
							</div>
							<div>
								<div class="font-medium">{task.title}</div>
								<div class="text-sm text-gray-500 select-none">{formatTime(task.duration)}</div>
							</div>
						</div>
						<button onclick={() => removeTask(index)} class="p-1 text-gray-500 hover:text-red-500">
							<X class="w-5 h-5" />
						</button>
					</div>
				{/each}
				{#if dropTarget === tasks.length}
					<div class="h-1 bg-blue-300 rounded transition-all"></div>
				{/if}
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
