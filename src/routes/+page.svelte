<script>
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
		try {
			const devices = await navigator.mediaDevices.enumerateDevices();
			cameras = devices.filter(device => device.kind === 'videoinput');
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

	async function setupCamera() {
		try {
			if (!selectedCameraId) return;
			
			const stream = await navigator.mediaDevices.getUserMedia({
				video: {
					deviceId: selectedCameraId
				}
			});
			
			if (videoElement.srcObject) {
				videoElement.srcObject.getTracks().forEach(track => track.stop());
			}
			
			videoElement.srcObject = stream;
			await videoElement.play();
		} catch (error) {
			console.error('Camera error:', error);
			status = 'Camera error: ' + error.message;
		}
	}

	function takePhoto() {
		const canvas = document.createElement('canvas');
		canvas.width = videoElement.videoWidth;
		canvas.height = videoElement.videoHeight;
		const ctx = canvas.getContext('2d');
		ctx.drawImage(videoElement, 0, 0);
		
		const link = document.createElement('a');
		link.download = `photo-${new Date().toISOString()}.png`;
		link.href = canvas.toDataURL('image/png');
		link.click();

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

<div 
	class="min-h-screen bg-black relative overflow-hidden"
	on:mousemove={handleMouseMove}
>
	<!-- Main Content -->
	<div class="grid h-screen grid-cols-1 lg:grid-cols-3 gap-8 p-8">
		<!-- Camera Preview (2 columns) -->
		<div class="lg:col-span-2 relative aspect-video bg-black rounded-xl overflow-hidden">
			<video
				bind:this={videoElement}
				class="absolute inset-0 w-full h-full object-cover"
			/>
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
			<div class="flex items-center space-x-2 flex-1">
				<label for="camera-select" class="text-sm text-gray-300">
					Camera:
				</label>
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