<script>
// @ts-nocheck

	import { onMount } from 'svelte';
	let serialPort;
	let videoElement;
	let status = 'Disconnected';
	let lastPhoto = null;
	let cameras = [];
	let selectedCameraId = '';
	let rotation = Math.random() * 6 - 3;
	let showControls = false;
	let controlsTimeout;
	let dirHandle = null;

	function handleMouseMove(event) {
		const threshold = window.innerHeight - 100;
		if (event.clientY > threshold) {
			showControls = true;
			clearTimeout(controlsTimeout);
		} else {
			clearTimeout(controlsTimeout);
			controlsTimeout = setTimeout(() => {
				showControls = false;
			}, 1000);
		}
	}

	async function getCameras() {
		console.log("getting cams")
		try {
			navigator.getUserMedia({video: true}, ()=>{}, ()=>{})
			const devices = await navigator.mediaDevices.enumerateDevices();
			console.log(devices)
			cameras = devices.filter((device) => device.kind === 'videoinput');
			if (cameras.length > 0) {
				selectedCameraId = cameras[0].deviceId;
				await setupCamera();
			}
		} catch (error) {
			console.error('Error getting cameras:', error);
			status = 'Camera error: ' + error.message;
		}
	}

	async function connectToSerial() {
		try {
			serialPort = await navigator.serial.requestPort();
			await serialPort.open({ baudRate: 115200 });
			status = 'Connected';

			const reader = serialPort.readable.getReader();
			const textDecoder = new TextDecoder();
			let buffer = '';

			while (true) {
				const { value, done } = await reader.read();
				if (done) break;

				buffer += textDecoder.decode(value);
				const messages = buffer.split('\n');
				buffer = messages.pop() || '';

				for (const message of messages) {
					if (message.includes('BUZZER0 PRESSED')) {
						takePhoto();
					}
				}
			}
		} catch (error) {
			console.error('Error:', error);
			status = 'Error: ' + error.message;
		}
	}

	/**
	 * Prompts the user to select a folder using the File System Access API
	 * and stores the resulting handle in the global 'dirHandle' variable.
	 *
	 * @returns {Promise<boolean>} A promise that resolves to true if a folder
	 * was successfully selected and the handle stored,
	 * and false otherwise (e.g., user cancelled, API error).
	 */
	async function selectFolder() {
		// Check if the API is supported
		if (!('showDirectoryPicker' in window)) {
			console.error(
				"File System Access API's showDirectoryPicker is not supported in this browser."
			);
			alert("Your browser doesn't support the required API to select folders.");
			// Ensure dirHandle remains null or unchanged if already set
			// dirHandle = null; // Or leave it as it was
			return false; // Indicate failure
		}

		try {
			console.log('Requesting folder selection...');
			// Prompt the user to select a directory
			const handle = await window.showDirectoryPicker();

			// Store the obtained handle in the global variable
			dirHandle = handle;

			console.log(`Folder selected: ${dirHandle.name}. Handle stored in global 'dirHandle'.`);
			// You can optionally add UI feedback here (e.g., update a status element)
			return true; // Indicate success
		} catch (error) {
			// Reset global handle if an error occurred during selection
			dirHandle = null;

			if (error.name === 'AbortError') {
				// This specific error means the user cancelled the picker
				console.warn('Folder selection was cancelled by the user.');
			} else {
				// Handle other potential errors
				console.error(`Error selecting folder: ${error.name} - ${error.message}`);
				// Optionally alert the user for more critical errors
				// alert(`An error occurred while selecting the folder: ${error.message}`);
			}
			return false; // Indicate failure
		}
	}
	async function setupCamera() {
		console.log("setting up cam")
		try {
			if (!selectedCameraId) return;

			const stream = await navigator.mediaDevices.getUserMedia({
				video: {
					deviceId: selectedCameraId
				}
			});

			if (videoElement.srcObject) {
				videoElement.srcObject.getTracks().forEach((track) => track.stop());
			}

			videoElement.srcObject = stream;
			await videoElement.play();
		} catch (error) {
			console.error('Camera error:', error);
			status = 'Camera error: ' + error.message;
		}
	}

	// Helper to get canvas Blob data as a Promise
	function getCanvasBlob(canvas, mimeType = 'image/png', quality = 0.92) {
		return new Promise((resolve, reject) => {
			canvas.toBlob(
				(blob) => {
					blob ? resolve(blob) : reject(blob);
				},
				mimeType,
				quality
			);
		});
	}

	async function takePhoto() {
		const canvas = document.createElement('canvas');
		canvas.width = videoElement.videoWidth;
		canvas.height = videoElement.videoHeight;
		const ctx = canvas.getContext('2d');
		ctx.drawImage(videoElement, 0, 0);



		const filename = `photo-${new Date().toISOString().replaceAll(":", "_")}.png`;


		const blob = await getCanvasBlob(canvas, 'image/png');

		// Get file handle (create if needed)
		const fileHandle = await dirHandle.getFileHandle(filename, { create: true });

		// Create writable stream, write data, and close
		const writableStream = await fileHandle.createWritable();
		await writableStream.write(blob);
		await writableStream.close();

		const link = document.createElement('a');
		link.download = filename;
		link.href = canvas.toDataURL('image/png');
		// link.click();
		lastPhoto = link.href;
		rotation = Math.random() * 6 - 3;
	}

	$: if (selectedCameraId) {
		setupCamera();
	}

	onMount(() => {
		getCameras();
	});
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div class="min-h-screen bg-black relative overflow-hidden" on:mousemove={handleMouseMove}>
	<!-- Main Content -->
	<div class="grid h-screen grid-cols-1 lg:grid-cols-3 gap-8 p-8">
		<!-- Camera Preview (2 columns) -->
		<div class="lg:col-span-2 relative aspect-video bg-black rounded-xl overflow-hidden">
			<!-- svelte-ignore a11y-media-has-caption -->
			<video bind:this={videoElement} class="absolute inset-0 w-full h-full object-cover" />
		</div>

		<!-- Latest Capture -->
		<div class="hidden lg:block">
			{#if lastPhoto}
				<div
					class="relative w-full transition-transform duration-300"
					style="transform: rotate({rotation}deg)"
				>
					<div class="bg-white p-4 rounded-sm shadow-[0_10px_20px_rgba(0,0,0,0.3)]">
						<div class="relative aspect-[4/3] mb-4 bg-black">
							<!-- svelte-ignore a11y-img-redundant-alt -->
							<img
								src={lastPhoto}
								alt="Last captured photo"
								class="absolute inset-0 w-full h-full object-cover"
							/>
						</div>
						<div class="h-4"></div>
					</div>
				</div>
			{:else}
				<div class="aspect-video rounded-lg bg-gray-800/50 flex items-center justify-center">
					<p class="text-gray-500">No photos captured yet</p>
				</div>
			{/if}
		</div>
	</div>

	<!-- Overlay Controls -->
	<div
		class="fixed bottom-0 left-0 right-0 transition-transform duration-300"
		class:translate-y-full={!showControls}
		style="transform: translateY({showControls ? '0' : '100%'})"
	>
		<div class="bg-black/80 backdrop-blur-md p-4 flex items-center gap-4">
			<!-- Connection Status -->
			<div class="flex items-center space-x-4">
				{#if status !== 'Connected'}
					<div class="flex items-center space-x-2">
						<div class="w-2 h-2 rounded-full bg-red-500"></div>
						<span class="text-gray-300 text-sm">{status}</span>
					</div>
				{/if}
				<button
					on:click={connectToSerial}
					class="bg-indigo-600 hover:bg-indigo-700 text-white px-3 py-1.5 rounded-md text-sm transition-colors duration-200"
				>
					{status === 'Connected' ? 'Reconnect Device' : 'Connect Device'}
				</button>
			</div>

			<!-- Camera Selection -->
			<div class="flex items-center space-x-2">
				<label for="camera-select" class="text-sm text-gray-300"> Camera: </label>
				<select
					id="camera-select"
					bind:value={selectedCameraId}
					class="bg-gray-700 text-white text-sm rounded-md px-2 py-1.5 focus:outline-none focus:ring-2 focus:ring-indigo-500"
				>
					{#each cameras as camera}
						<option value={camera.deviceId}>
							{camera.label || `Camera ${cameras.indexOf(camera) + 1}`}
						</option>
					{/each}
				</select>
			</div>

			<!-- Folder selector -->
			<div class="flex items-center space-x-4">
				<button
					on:click={selectFolder}
					class="bg-indigo-600 hover:bg-indigo-700 text-white px-3 py-1.5 rounded-md text-sm transition-colors duration-200"
				>
					Select Folder
				</button>
			</div>
		</div>
	</div>
</div>
